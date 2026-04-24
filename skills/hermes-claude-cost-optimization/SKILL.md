---
name: hermes-claude-cost-optimization
description: 诊断和优化 Hermes Agent 的 Anthropic Claude API 消费。用于分析 usage CSV、识别浪费、应用省钱配置（分层模型、1h prompt cache、auxiliary 显式指定）。
---

# Hermes Claude API 成本优化

## 何时使用
- 用户报告 Anthropic 账单过高
- 用户想了解为什么钱烧得快
- 新装 Hermes 后预防性配置
- 看到 console 账单飙升想定位根因

## 核心洞察（2026-04-24 实战得出）

**三大烧钱元凶（按影响排序）：**

1. **auxiliary auto-detect 跟随主模型** — 如果 config.yaml 里 `auxiliary.*.model: ''` 和 `provider: auto`，Hermes 会把所有后台任务（title 生成、compression、memory flush、skill 加载、vision、web_extract 等 9 项）跑成当前主模型。当主模型是 Opus 时，每条消息都触发 5-10 次 Opus 后台调用。**单这一项就能占到总消费的 20-25%。**

2. **delegation（子 agent）同样会 auto-detect** — `delegation.model: ''` + `provider: ''` 导致 subagent 也跑 Opus。子任务本质重复性高，用 Sonnet 足够。

3. **prompt_cache_ttl 默认 5m 在断续使用场景下完全失效** — 5 分钟一过 cache 作废，下次重新写入（1.25x 单价）。用户真实数据：cache_write $139 vs cache_read $85（写比读还多 60% = 极度不健康）。切到 1h（2x 写入单价）反而省 60-80%。

## 诊断流程

### 第一步：拿到 usage CSV
1. 用户打开 https://platform.claude.com/usage
2. 选时间范围（建议 Last 30 days 或 This month）
3. 右上角 Export → 下载 CSV
4. 文件扔到 `/root/workspace/` 或 `/mnt/c/Users/<YOUR_WINDOWS_USER>/Downloads/`

CSV 列：`usage_date_utc, model, workspace, api_key, usage_type, context_window, token_type, cost_usd, list_price_usd, cost_type, inference_geo, speed`

### 第二步：运行分析脚本

```python
import csv
from collections import defaultdict

rows = []
with open("/root/workspace/claude_api_cost_*.csv") as f:
    for r in csv.DictReader(f):
        r["cost_usd"] = float(r["cost_usd"])
        rows.append(r)

total = sum(r["cost_usd"] for r in rows)

# 按模型
by_model = defaultdict(float)
for r in rows: by_model[r["model"]] += r["cost_usd"]

# 按 token_type (看 cache 效率)
by_type = defaultdict(float)
for r in rows: by_type[r["token_type"]] += r["cost_usd"]

# 按 context_window
by_ctx = defaultdict(float)
for r in rows: by_ctx[r["context_window"]] += r["cost_usd"]
```

### 第三步：判断是否不健康

| 指标 | 健康 | 警报 |
|---|---|---|
| cache_write / cache_read 比例 | <0.3 (读远多于写) | >1.0 (写比读多) |
| Opus 模型占比 | <40% | >60% |
| 200k-1M 长上下文占比 | <10% | >20% |
| 每日消费波动 | 平稳 | 某天突增 5x+ |

### 第四步：查 agent.log 找 auto-detect 证据
```bash
grep "Auxiliary auto-detect" /root/.hermes/logs/agent.log | tail -20
```
如果看到 `using main provider anthropic (claude-opus-*)` → 后台任务在跑 Opus → 立刻改配置。

## 修复配置（config.yaml）

### 1. 所有 auxiliary 显式指定 Sonnet 4.6（9 项）

每个 auxiliary 子项：
```yaml
auxiliary:
  approval:
    model: 'claude-sonnet-4-6'
    provider: 'anthropic'
  compression:
    model: 'claude-sonnet-4-6'
    provider: 'anthropic'
  flush_memories:
    model: 'claude-sonnet-4-6'
    provider: 'anthropic'
  mcp:
    model: 'claude-sonnet-4-6'
    provider: 'anthropic'
  session_search:
    model: 'claude-sonnet-4-6'
    provider: 'anthropic'
  skills_hub:
    model: 'claude-sonnet-4-6'
    provider: 'anthropic'
  title_generation:
    model: 'claude-sonnet-4-6'
    provider: 'anthropic'
  vision:
    model: 'claude-sonnet-4-6'
    provider: 'anthropic'
  web_extract:
    model: 'claude-sonnet-4-6'
    provider: 'anthropic'
```

### 2. delegation 显式指定
```yaml
delegation:
  model: 'claude-sonnet-4-6'
  provider: 'anthropic'
```

### 3. 主对话保持 Opus（如果用户需要最高智能）
```yaml
model:
  name: claude-opus-4-7
  provider: anthropic
  prompt_cache_ttl: '1h'  # 新增，需要源码支持
fallback_providers:
- model: claude-sonnet-4-6
  provider: anthropic
```

### 4. 源码打补丁：支持 prompt_cache_ttl 配置

Hermes v0.10.0 的 `run_agent.py:1010` 硬编码 `self._cache_ttl = "5m"`。改法：

**a)** 把 `self._cache_ttl = "5m"` 改成 `self._cache_ttl = self._resolve_cache_ttl()`

**b)** 在 `_anthropic_prompt_cache_policy` 方法前插入：

```python
def _resolve_cache_ttl(self) -> str:
    """Resolve prompt cache TTL from config. Returns '5m' or '1h'."""
    try:
        from cli import load_cli_config
        cfg = load_cli_config() or {}
        val = str((cfg.get("model") or {}).get("prompt_cache_ttl") or "").strip().lower()
        if val in ("5m", "1h"):
            return val
        if val:
            logger.warning(f"Invalid model.prompt_cache_ttl={val!r}; falling back to '5m'")
    except Exception as e:
        logger.debug(f"_resolve_cache_ttl config lookup failed: {e}")
    return "5m"
```

**警告：** Hermes 升级会覆盖这个补丁。升级后必须重新打。

### 5. 重启 Hermes 让配置生效
退出 CLI 重进，或 `systemctl restart hermes-webui`。

## 预期效果（来自真实用户数据）

- **3 天花费 $246 → 预期降到 $100-130**（节省 50%）
- **cache_write 占比 56% → <30%**（1h 缓存复用生效）
- **Opus 占比 76% → 40-50%**（子任务迁移到 Sonnet）
- **月消费从 $2400 节奏 → $900-1200**（正好落在 tier1 $1000 额度内）

## 陷阱

1. **写 config.yaml 时不要用 execute_code + read_file["content"]** — read_file 输出带 `NNN|` 行号前缀，直接写回会把行号写进文件，导致 YAML 解析失败。必须用 patch 或先 strip 行号。

2. **Anthropic 硬编码 cache_ttl 只有 "5m" 和 "1h"** — 不存在 2h/24h 选项。再问用户"你硬件强能不能更长" = 误解，因为 cache 在 Anthropic 服务器上，不在本地。

3. **Admin Key 页面对很多用户 404** — platform.claude.com/settings/admin-keys 通常只对高 tier 账户开放。Workspace API Key (`sk-ant-api03-*`) 无法调 `/v1/organizations/usage_report/*`（返回 401）。**目前没有自动化拉 usage 的路径，必须手动 CSV 导出。**

4. **组织 spending limit 有硬上限** — tier1 账户最多 $1000/月，不可突破。用户看到 "Cannot exceed organization's enforced limit of $1,000.00" 不是错误，是 Anthropic 的保护。

5. **fallback 救不了月度额度耗尽** — fallback 只应对 provider 故障，不是 quota exceeded。同 org 内所有模型共享同一个 $1000 额度。真正想防断服要加 Ollama 本地模型作为末级 fallback。

## 验证闭环

每周让用户重新导 CSV，对比：
- cache_write/cache_read 比例（应持续下降）
- auxiliary auto-detect log（应完全消失）
- 总日均消费（应稳定在 $30-40）

如果一周后数据没改善 → 检查 Hermes 是否真的重启了、source 补丁是否被升级覆盖。
