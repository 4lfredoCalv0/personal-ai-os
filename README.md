# personal-ai-os

Asistente personal: Hermes como orquestador, Obsidian como memoria, Claude Code
como ejecutor técnico.

```
        TÚ  (terminal · Telegram · Obsidian)
                     │
            ┌────────▼───────┐
            │     HERMES     │  orquestador · cron · gateway
            └───┬────┬────┬──┘
   filesystem   │    │    │  MCP
   (write-safe) │    │    └──────► n8n
                │    └─ subprocess `claude -p` ──► CLAUDE CODE
                ▼
          OBSIDIAN VAULT  ◄── fuente de verdad
                ▲
        solo tras revisión humana
                │
        Whisper local ← ScreenCaptureKit
```

| Carpeta | Qué contiene |
|---|---|
| `hermes/` | Configuración de Hermes (`config.yaml`, `SOUL.md`) |
| `skills/` | Skills de Hermes en formato agentskills.io |
| `scripts/` | Captura de audio y utilidades |
| `docs/` | Arquitectura y decisiones de diseño |

La arquitectura completa está en [`docs/arquitectura.md`](docs/arquitectura.md).

**El vault vive aparte**, en `~/Documents/Obsidian Vault`, con su propio repositorio.
