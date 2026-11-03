# Presentation for Financing

Плагин **financing-slides** — презентации в Figma Slides в шаблоне «Манила 30»
(https://www.figma.com/slides/Va5dPaSmSmwC228TKjiaGE). Один исходник для Claude и Codex.

| Где | Как поставить |
|---|---|
| Claude (приложение) | файл `financing-slides.plugin` из релиза или чата |
| Claude Code | `/plugin marketplace add dalie86/Presentation-for-Financing`, затем `/plugin install financing-slides --marketplace dalie86/Presentation-for-Financing` |
| Codex | `codex plugin marketplace add dalie86/Presentation-for-Financing`, затем Plugins → Financing → Financing Slides |

Подробности, требования к Figma и описание навыков — `plugins/financing-slides/README.md`.

## Устройство репозитория

```
.claude-plugin/marketplace.json     каталог для Claude Code
.agents/plugins/marketplace.json    каталог для Codex
plugins/financing-slides/
  .claude-plugin/plugin.json        манифест Claude
  .codex-plugin/plugin.json         манифест Codex
  .mcp.json                         Figma MCP (общий)
  skills/                           10 навыков slides-* (общие)
  agents/                           агенты Claude
```

Навыки правятся в одном месте — `plugins/financing-slides/skills/`. Версию поднимать в обоих
манифестах.
