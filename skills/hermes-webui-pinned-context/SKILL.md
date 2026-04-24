---
name: hermes-webui-pinned-context
description: Pin project-specific context (rules, glossary, decision log) to Hermes Web UI so every new conversation auto-loads it. Uses the AGENTS.md / CLAUDE.md auto-injection mechanism — Hermes reads these files from its cwd at session start via build_context_files_prompt(). Use when user wants "置顶" (pinned) project knowledge available in every Web UI chat without manually restating.
version: 1.0.0
metadata:
  hermes:
    tags: [webui, context, pinning, agents-md, memory]
    related_skills: [hermes-webui, hermes-webui-api]
---

# Pinning Project Context to Hermes Web UI

## The problem

User wants a "置顶频道" / pinned reference so that every Web UI conversation automatically knows about a specific project (rules, glossary, key facts). Hermes `memory` is limited (~2200 chars) — not enough for a full project spec.

Web UI has no built-in "pinned message" or "channel system prompt" feature.

## The solution: AGENTS.md in Web UI's cwd

Hermes automatically loads context files from its current working directory at the start of every conversation. Detection logic is in `agent/prompt_builder.py` → `build_context_files_prompt()`:

- Reads `AGENTS.md` or `agents.md` (top-level in cwd, no recursive walk)
- Reads `CLAUDE.md` or `claude.md` (cwd only)
- Reads `.cursorrules` + `.cursor/rules/*.mdc` (cwd only)

Controlled by `skip_context_files: bool = False` on AIAgent (defaults to loading).

## Finding Web UI's cwd

Web UI is launched by `/usr/local/bin/hermes-webui-start.sh`. Check the `cd` line:

```bash
read_file /usr/local/bin/hermes-webui-start.sh
# Look for: cd "$WEBUI_DIR"  (typically /root/hermes-webui)
```

As of 2026-04-24: cwd is `/root/hermes-webui/`.

## Layered context strategy (what goes where)

| Layer | Location | Size limit | Scope | Use for |
|-------|----------|-----------|-------|---------|
| `memory` (user+memory) | `~/.hermes/memories/MEMORY.md` | ~2200 chars each | ALL sessions (WhatsApp, Web UI, CLI) | Tiny facts, preferences, one-liners |
| Web UI `AGENTS.md` | `/root/hermes-webui/AGENTS.md` | ~32KB truncate limit | Web UI only | Full project glossary, rules, term tables |
| Full reference file | `~/.hermes/<project>_rules.md` | Unbounded | On-demand reads | Complete specs (13-chapter docs etc.) |

## Recipe

### Step 1: Store complete reference (unbounded)

```bash
write_file ~/.hermes/<project>_rules.md  # full content
```

### Step 2: Summary to memory (for other platforms)

Keep a compressed version in `memory` with a POINTER to the full file:

```
<Project> 项目规则全文: ~/.hermes/<project>_rules.md。核心: <key terms>。关键比例: <numbers>.
```

Keep under 500 chars — memory is shared across platforms and fills fast.

### Step 3: Pin structured context to Web UI

```bash
write_file /root/hermes-webui/AGENTS.md
```

Structure:
```markdown
# Project Context for Hermes Web UI Sessions

This file is automatically loaded by Hermes at the start of every Web UI conversation.

## 📌 置顶参考：<Project Name>

- **完整规则文件**：`~/.hermes/<project>_rules.md`
- **一句话定位**：<one-sentence what-is-it>

### 核心术语
| 名称 | 性质 | 关键约束 |
|------|------|----------|
| TERM_A | ... | ... |

### 关键规则 / 数字
- 比例/配额/阈值

### 合规定位 / 避雷表达
- ✅ 应说: ...
- ❌ 必须避免: ...

### 待确认项
- ...

## 交流习惯
- 称呼、语言、偏好（可选：方便 Web UI 继承同样风格）
```

### Step 4: Verify

User opens Web UI, starts a new conversation, asks something specific from the pinned content:

> <Project> 的 <specific rule> 是什么？

Web UI Hermes should answer without any file reads — confirmed pinned.

## Pitfalls

1. **Web UI cwd can change across Hermes versions** — the launcher script is the truth. Don't hardcode `/root/hermes-webui/`; verify each time.

2. **32KB truncation** — `_truncate_content()` in `prompt_builder.py` caps AGENTS.md injection. For huge projects, put the full text in `~/.hermes/<project>_rules.md` and only a TOC + key facts in AGENTS.md.

3. **Conflicts with existing AGENTS.md** — if the cwd already has an AGENTS.md (e.g. `/root/.hermes/hermes-agent/AGENTS.md` for development guide), don't overwrite — MERGE your project section in.

4. **Only affects NEW sessions** — existing open Web UI conversations keep their original context. User may need to start a new chat to see pinned content.

5. **`memory` (user+memory profile) is global across platforms** — if you put project specifics there, WhatsApp/Telegram sessions also get them. That's usually fine, but be aware.

6. **Not propagated to CLI sessions** — CLI cwd depends on where user launched from. If they want CLI to pin the same context, put AGENTS.md in their project directory or `~/`.

## Alternative: System-level prompt injection

For truly global pinning (all platforms, all sessions), could modify `agent/prompt_builder.py` to inject a user-configured snippet. Not recommended — upstream PRs preferred, local forks drift.

## Related: api_server target confusion

`send_message(target='api_server')` does NOT exist as a platform. Web UI is not a messaging platform — it's a conversation UI. To "send something to Web UI", you write to shared storage (memory, AGENTS.md, files) that Web UI Hermes reads at session start.

Available messaging platforms (as of 2026-04-24): telegram, discord, slack, whatsapp, signal, bluebubbles, qqbot, matrix, mattermost, homeassistant, dingtalk, feishu, wecom, wecom_callback, weixin, email, sms.
