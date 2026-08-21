# Personal AI OS — cómo construirlo y qué construir

Dos partes. La primera define **cómo trabajamos** (sesiones paralelas, git, vault); la segunda **qué construimos** (Hermes + Obsidian + Claude Code). La primera va antes porque condiciona a la segunda.

---
---

# PARTE 1 — Trabajo en paralelo

## Contexto

Quieres 3–5 sesiones de Claude Code simultáneas sin que se pisen. La idea de abrir varias terminales y hacer `cd` a la misma carpeta **sí es peligrosa**: Claude Code no bloquea archivos, dos sesiones editando el mismo repo se sobrescriben sin avisar, y `.claude/settings.local.json` lo escribe cada sesión.

Pero la solución ya existe y no hay que construirla.

## El hallazgo que evita todo el trabajo

**Claude Code trae worktrees nativos con aislamiento aplicado a nivel de herramienta.**

```bash
claude --worktree hermes
```

Crea `.claude/worktrees/hermes/` en la rama `worktree-hermes` y arranca ahí. Repetir con otro nombre en otra terminal da otra sesión aislada.

No es convención — Claude Code **bloquea activamente** cuatro cosas mientras la sesión está aislada:

| Comprobación | Qué bloquea |
|---|---|
| Ediciones | `Edit`/`Write`/`NotebookEdit` que apunten al checkout principal |
| Directorio | Comandos Bash cuyo cwd resuelva al principal, o que no pueda verificar |
| Redirección git | `git -C`, `--git-dir`, `GIT_DIR`, `GIT_WORK_TREE`, o un `cd` previo al principal |
| Forma del comando | Construcciones que no puede trazar sin ejecutarlas (heredocs sin delimitador citado, brace expansion) |

La última no se puede desactivar. Un agente no puede escaparse de su worktree aunque se lo pidas.

**Consecuencia directa:** los scripts de tu sección 18 (`new-agent-task.sh`, `list-agent-worktrees.sh`, `finish-agent-task.sh`) **no se crean**. `claude --worktree <nombre>` y `git worktree list` ya lo hacen.

### Dos detalles que resuelven problemas que encontré

- **Las aprobaciones de permisos se guardan en el checkout principal** y aplican a todos los worktrees. Eso elimina el conflicto de `settings.local.json` que detecté en la auditoría.
- **`.worktreeinclude`** copia archivos gitignored (`.env`) a cada worktree nuevo. Sin él, un worktree arranca sin credenciales locales.

## Evaluación honesta: ¿necesitas 5 sesiones?

**Para código, sí. Para lo que tienes por delante, probablemente no todavía.**

El trabajo que listaste — Hermes, reuniones, skills de Obsidian, investigación — es mayormente **secuencial**: la fase de reuniones depende de que Hermes funcione; las skills dependen de que el vault esté conectado. Paralelizar tareas dependientes no acelera nada y multiplica los merges.

Lo que sí paraleliza bien desde el día uno:

- **Investigación** — leer documentación no toca archivos, no conflictúa con nada
- **Trabajo en repos distintos** — `salus-portal` y `personal-ai-os` no comparten nada
- **Tareas mecánicas independientes** — dos skills separadas, dos scripts separados

Mi recomendación: **empieza con dos sesiones**, no cinco. Una que construye, una que investiga. Sube a tres o cuatro cuando el trabajo sea genuinamente independiente.

## El hueco que encontré: no hay repositorio

El plan de Hermes no tiene dónde vivir. Los repos actuales:

| Repo | Rama | Worktrees | Remoto |
|---|---|---|---|
| `Obsidian Vault` | main | 0 | GitHub privado |
| `n8n-agent-builder` | main | 0 | GitHub privado |
| `salus-portal` | main | 0 | GitHub (freddysalus) |
| `emerald` | master | 0 | GitHub |
| `salus-saunas-product-photos` | — | — | **no es repo** (2.1 GB) |

Ninguno es el sitio correcto: `n8n-agent-builder` es automatización de Salus, y el vault guarda conocimiento, no código.

**Se crea `~/Documents/Proyectos Claude Code/personal-ai-os/`** — repo nuevo, privado, para configuración de Hermes, skills, scripts de reuniones y documentos de arquitectura. **Es aquí donde los worktrees pagan.**

## Obsidian: worktrees NO

Esta es la decisión importante, y va en contra de aplicar la misma estrategia a todo.

**El vault no usa worktrees. Un único directorio de trabajo, un escritor a la vez.**

Cuatro razones, en orden de peso:

**1. Obsidian apunta a un solo directorio.** Un worktree es otra carpeta que Obsidian no indexa. Estarías editando notas que la app no ve: sin backlinks, sin Tasks, sin vistas `.base`. La herramienta que da valor al vault quedaría ciega a la mitad del trabajo.

**2. Git no ve el acoplamiento semántico.** Hay **119 enlaces internos entre 40 notas**. `Salus Saunas` está enlazada 14 veces, `Gmail Assistant` 13. Si una rama renombra una nota y otra crea enlaces al nombre viejo, **git mergea limpio y el resultado está roto**. Git compara texto; el grafo de conocimiento no es texto.

**3. El conocimiento no se ramifica.** Una rama de código es una hipótesis que se prueba y se mergea. ¿Qué sería una rama de una decisión? No hay dos versiones de *por qué decidimos algo* que se fusionen — hay una correcta.

**4. El auto-commit de las 21:00** hace `git add -A` sobre la rama actual del vault. Con ramas paralelas, commitea la que esté puesta.

### Política de escritura del vault

| Nivel | Qué | Cómo |
|---|---|---|
| **Paralelo sin riesgo** | Leer. Cualquier sesión, siempre. | Sin coordinación |
| **Paralelo con cuidado** | Crear notas nuevas en carpetas distintas | Nombres con fecha o distintivos evitan colisión |
| **Un escritor a la vez** | `CLAUDE.md`, `Inicio.md`, `Pendientes.md`, plantillas, scripts, vistas `.base`, estructura de carpetas | **Una sola sesión. Nunca en paralelo.** |

**Sin lock files, sin colas.** Para un usuario único con dos o tres terminales, un servidor de locks es infraestructura para un problema que no tienes: las escrituras duran segundos y tú controlas quién hace qué. La protección real es más barata:

```bash
git -C "$VAULT" status --porcelain      # qué se tocó
"99 System/Scripts/check-links.sh"      # los enlaces siguen resolviendo
```

`check-links.sh` es el que importa: **detecta exactamente lo que git no puede ver.**

Y como red de fondo: commit después de cada sesión de escritura, no solo el automático de las 21:00. Así el punto de retorno es por sesión.

## Hermes y los worktrees

**Hermes no crea un worktree por tarea.** La documentación es clara en un punto que lo desaconseja: las corridas con `-p` **no limpian su worktree al terminar** y dejan un lock puesto hasta que una barrida posterior lo libere. Un worktree por tarea acumula directorios bloqueados.

Además, la mayoría de las delegaciones (*investiga esto*, *analiza este código*, *por qué falla X*) son **de solo lectura** — no necesitan aislamiento en absoluto.

La política:

| Tipo de delegación | Aislamiento |
|---|---|
| Leer, analizar, investigar | Ninguno. `claude -p` en el directorio del repo. |
| Escribir código | Un worktree **persistente** `hermes-tasks`, reutilizado |
| Escribir en el vault | Ninguno — el vault no usa worktrees. Hermes escribe en `00 Inbox` del checkout único. |

**Regla que sí adopto de tu sección 15:** Hermes nunca commitea a `main` de un repo de código. Trabaja en su rama, tú revisas, tú mergeas.

## Convenciones de git

Commits con ámbito, útiles cuando cuatro ramas convergen:

```
feat(hermes): conectar el vault por filesystem
fix(meetings): corregir la captura de audio del sistema
docs(arch): registrar el modelo de delegación
research(mcp): documentar los hallazgos de hermes mcp serve
```

**Prohibido para cualquier agente:** `push --force`, `reset --hard` sobre ramas compartidas, borrar ramas ajenas. Ninguna sesión mergea a `main` sin que tú lo veas.

## Coordinación entre sesiones

Git + Markdown alcanza. **No se crea un directorio `/tasks` ni un sistema de tickets.**

- **Qué se hizo** → `git log` y los mensajes de commit
- **Qué se decidió** → `07 Decisions/` del vault, que ya existe para esto exactamente
- **Qué queda pendiente** → tareas inline en la nota del proyecto, que `Pendientes.md` ya recoge
- **En vuelo** → el nombre de la rama

Construir un mecanismo de handoff aparte duplicaría lo que el vault ya hace. Ese fue el error que corregimos con la memoria; no lo repetimos.

## Flujo diario real

```bash
# Terminal 1 — construir
cd ~/Documents/Proyectos\ Claude\ Code/personal-ai-os
claude --worktree hermes

# Terminal 2 — investigar (sin worktree: solo lee)
cd ~/Documents/Proyectos\ Claude\ Code/personal-ai-os
claude

# Terminal 3 — otro repo, sin relación
cd ~/Documents/Proyectos\ Claude\ Code/salus-portal
claude --worktree crm-decision
```

Al terminar, en cada worktree: commit, y `git worktree list` para ver qué queda. Claude limpia solo los worktrees sin cambios al salir.

## Prueba de verificación

1. Dos sesiones con `--worktree`, cada una crea un archivo distinto.
2. Desde una, pedir explícitamente que edite un archivo del checkout principal → **debe ser bloqueado**. Si lo consigue, el aislamiento no está activo.
3. `git worktree list` muestra ambos.
4. Merge de las dos ramas a main sin conflicto.
5. En el vault, tras cualquier escritura: `check-links.sh` sin enlaces rotos.

---
---

# PARTE 2 — Arquitectura de Hermes

*(Sin cambios respecto a lo presentado; la Parte 1 no la contradice. Resumen para contexto.)*

**Decisiones tomadas:** modelo Anthropic API · audio de reuniones 100% local · Hermes arranca con lectura + escritura solo en `00 Inbox`.

```
        TÚ  (terminal · Telegram · Obsidian)
                     │
            ┌────────▼───────┐
            │     HERMES     │  orquestador · cron · gateway
            └───┬────┬────┬──┘
   filesystem   │    │    │  MCP (ya existe)
   (write-safe) │    │    └──────► n8n
                │    └─ subprocess `claude -p` ──► CLAUDE CODE
                ▼
          OBSIDIAN VAULT  ◄── fuente de verdad · git = deshacer
                ▲
        solo tras revisión humana
                │
        Whisper local ← ScreenCaptureKit
```

**Por qué no todo por MCP:** verificado que `hermes mcp serve` expone **solo 10 herramientas de mensajería** — no memoria ni archivos. Claude Code va por subproceso. Obsidian va por filesystem acotado con `HERMES_WRITE_SAFE_ROOT`; un MCP de Obsidian añadiría un proceso y un modo de fallo a cambio de nada.

**Memoria — no hay conflicto:** la memoria propia de Hermes son ~3.600 caracteres (`MEMORY.md` + `USER.md`) sobre cómo hablarte, no sobre tu negocio. **Se prohíben los 8 backends de memoria externa** (Mem0, Supermemory…): corren *junto* a la interna y ahí sí crearían la segunda fuente de verdad. **Sin vector DB** — `grep` + frontmatter resuelve en un paso.

**Reuniones:** Voice Mode de Hermes es para hablarle a él, no para transcribir reuniones. Capa aparte, local. **La transcripción cruda nunca entra al vault** — solo la destilación que revisaste.

**Seguridad:** `approvals.mode: smart`, `cron_mode: deny`, `HERMES_WRITE_SAFE_ROOT` apuntando al vault. Nunca `--yolo`. Nunca borrar: `status: archived` + `08 Archive`.

**Skills: tres, no catorce** — `obsidian-vault`, `meeting-assistant`, `delegate-to-claude`. El `CLAUDE.md` del vault sigue siendo la definición de convenciones; la skill lo referencia, no lo copia.

**Fases:** (1) Hermes + vault lectura · (2) escritura en cuarentena · (3) delegación · (4) Telegram · (5) reuniones · (6) proactividad · (7) n8n.

---

## Riesgos declarados

**Hermes tiene 6 meses.** 234k estrellas y desarrollo muy activo, pero joven. El vault no depende de él: son archivos de texto que sobreviven si Hermes desaparece.

**El instalador es `curl | bash`.** Vía oficial de Nous Research, pero es el patrón que el propio escáner de Hermes marca como peligroso. Se lee el script antes, y solo desde `hermes-agent.nousresearch.com`.

**Hay granjas de contenido suplantando el proyecto.** `hermes-agent.org`, `hermes-ai.net`, `hermes-agent.ai`, `hermesagent.agency` — ninguno es de Nous Research, y al menos uno publica una versión de mayo de 2024 para un proyecto lanzado en febrero de 2026. **Fuentes válidas: solo `github.com/NousResearch/hermes-agent` y `hermes-agent.nousresearch.com`.**

**Un agente que escribe y luego lee lo que escribió se cree a sí mismo.** Razón de la cuarentena en `00 Inbox` y de la revisión humana en reuniones.

**Sigue sin haber Time Machine.** Git cubre el vault y tres repos. Nada más — ni las 2.1 GB de fotos de producto.

---

## Orden de ejecución propuesto

1. Crear el repo `personal-ai-os` con su `CLAUDE.md`, `.gitignore` (incluyendo `.claude/worktrees/`) y `.worktreeinclude`.
2. Probar el aislamiento de worktrees con la verificación de arriba.
3. Recién entonces, Fase 1 de Hermes.
