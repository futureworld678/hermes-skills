# Hermes Skills

Curated skills for [Hermes Agent](https://github.com/NousResearch/hermes-agent) — reusable procedural knowledge packaged as SKILL.md files.

## What's a "skill"?

A skill is a markdown file (`SKILL.md`) with YAML frontmatter that teaches an AI agent how to accomplish a specific task. It includes trigger conditions, step-by-step instructions, pitfalls, and verification steps. Hermes Agent auto-discovers skills placed under `~/.hermes/skills/`.

## Included skills

### devops

- **[hermes-claude-cost-optimization](skills/hermes-claude-cost-optimization/)** — Diagnose and reduce Anthropic API spend. Covers the auxiliary auto-detect trap (where subtasks silently run on Opus), 1h prompt cache config, and CSV-based usage analysis. Saved a real user ~50% of monthly spend.

- **[hermes-api-server-exposure](skills/hermes-api-server-exposure/)** — Open the Hermes Gateway API Server to other agents on the LAN/WSL host. Documents the non-obvious config precedence: `.env` overrides systemd env, and `config.yaml` `gateway.api_server` fields are ignored entirely.

- **[hermes-webui-pinned-context](skills/hermes-webui-pinned-context/)** — Pin project-specific context (glossaries, rules, decision logs) so every new Web UI conversation auto-loads it. Uses the AGENTS.md / CLAUDE.md auto-injection mechanism.

- **[wsl2-optimization](skills/wsl2-optimization/)** — TCP tuning, mirrored networking, Windows interop pitfalls, creating Windows desktop shortcuts from WSL, and setting up WSL services for auto-start.

### research

- **[extract-chatgpt-share](skills/extract-chatgpt-share/)** — Extract conversation text from a `chatgpt.com/s/...` share URL without a browser or login. Handles the server-rendered react-router streaming format where content is embedded in escape-encoded script tags.

## Install

Clone into your Hermes skills directory:

```bash
cd ~/.hermes/skills
git clone https://github.com/futureworld678/hermes-skills.git community-skills
# Skills are auto-discovered from subdirectories
```

Or cherry-pick individual skills:

```bash
curl -o ~/.hermes/skills/devops/hermes-claude-cost-optimization.md \
  https://raw.githubusercontent.com/futureworld678/hermes-skills/main/skills/hermes-claude-cost-optimization/SKILL.md
```

## Contributing

Submit a PR with a new skill or improvement. Each skill needs:

1. `SKILL.md` with YAML frontmatter: `name`, `description`
2. Clear trigger conditions ("use when...")
3. Numbered steps with exact commands
4. A "Pitfalls" section for things you discovered the hard way
5. No personal info — use placeholders like `<USERNAME>`, `<WSL_USER>`, `<YOUR_EMAIL>`

## License

MIT — see [LICENSE](LICENSE).

## Related

- [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent) — the agent these skills run in
