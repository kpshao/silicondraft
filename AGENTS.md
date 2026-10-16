# SiliconDraft — agent 约定

半导体 CP/WAT 测试数据生成工具（.NET 10 + Avalonia + DuckDB）。

权威文档：`doc/requirements-spec.MD`（What，唯一权威需求来源）、`doc/design.MD`（How）、`doc/contracts/`（对外契约）。

## Agent skills

### Issue tracker

Issues and wayfinder maps live as GitHub issues in this repo (`kpshao/silicondraft`), driven by the `gh` CLI. See `docs/agents/issue-tracker.md`.

### Triage labels

The five canonical roles, each label string equal to its name (`needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix`). See `docs/agents/triage-labels.md`.

### Domain docs

Single-context: one `GLOSSARY.md` + `docs/adr/` at the repo root. See `docs/agents/domain.md`.
