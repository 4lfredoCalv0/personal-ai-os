# Configuración de Hermes

Copia versionada de `~/.hermes/config.yaml`. **No incluye secretos** — las API keys
viven en `~/.hermes/.env`, que nunca se versiona.

## Decisiones reflejadas aquí

| Clave | Valor | Por qué |
|---|---|---|
| `model.default` | `claude-sonnet-5` | Buena relación costo/latencia para conversar y capturar. Lo pesado se delega a Claude Code. |
| `model.provider` | `anthropic` | Anthropic directo. **El instalador trae `base_url` de OpenRouter por defecto** — se quitó, o una key de Anthropic no funcionaría. |
| `approvals.mode` | `smart` | Un LLM auxiliar evalúa el riesgo; lo peligroso se deniega solo. |
| `approvals.cron_mode` | `deny` | Las tareas programadas nunca auto-aprueban comandos peligrosos. |

En `.env` (fuera de git): `HERMES_WRITE_SAFE_ROOT` apuntando al vault, que acota
`write_file` y `patch` a esa carpeta **a nivel de herramienta**.

## Modelos auxiliares

Hermes usa un modelo barato aparte para tareas laterales (visión, resumen web,
compresión de contexto, títulos de sesión, búsqueda en sesiones). Por defecto
Gemini Flash, autodetectado. **Se dejan en `auto`**: el propio config advierte que
cambiarlos a proveedores distintos de OpenRouter o Nous Portal es experimental.

No existe un router por dificultad para la conversación principal. Para una sesión
que se sabe pesada: `hermes -m claude-opus-5`.

## Cuidado al editar

`hermes config set` **reescribe el archivo normalizado y borra todos los comentarios**
(de 1924 líneas a 135 la primera vez). El original comentado quedó en
`~/.hermes/config.yaml.bak-preclaude`.
