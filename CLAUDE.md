# CLAUDE.md — personal-ai-os

Repositorio del asistente personal: configuración de Hermes, skills, scripts de
reuniones y documentos de arquitectura.

**Este repo guarda el sistema. El conocimiento vive en el vault de Obsidian.**

---

## 1. Frontera con el vault

`~/Obsidian Vault` es la **fuente de verdad del conocimiento** y tiene su
propio `CLAUDE.md` con las convenciones. Este repo no las duplica: las referencia.

| Aquí | En el vault |
|---|---|
| Configuración de Hermes | Proyectos, decisiones, personas |
| Skills (código y prompts) | Runbooks y procedimientos |
| Scripts de captura de audio | Notas de reuniones destiladas |
| Documentos de arquitectura | El *porqué* de las decisiones |

**Nunca** escribir credenciales aquí. Van en `.env` (gitignored) o en el gestor de
secretos correspondiente.

**Nunca** guardar transcripciones ni audio de reuniones en git. `meetings/` está
ignorado a propósito: es material crudo que puede estar mal y que no debe
sobrevivir a su destilación.

---

## 2. Trabajo en paralelo

Varias sesiones de Claude Code trabajan a la vez en este repo mediante worktrees
nativos:

```bash
claude --worktree hermes      # una sesión aislada
claude --worktree meetings    # otra, en otra terminal
```

Cada una vive en `.claude/worktrees/<nombre>/` sobre la rama `worktree-<nombre>`.
Claude Code **bloquea a nivel de herramienta** cualquier intento de escribir en el
checkout principal desde un worktree — no es convención, es aislamiento real.

Para trabajo de solo lectura (investigar, leer documentación) **no hace falta
worktree**: `claude` a secas basta y no genera conflictos.

`git worktree list` muestra los activos. Los worktrees sin cambios se limpian solos
al salir.

---

## 3. Reglas de git

Commits con ámbito, para que cuatro ramas convergiendo sigan siendo legibles:

```
feat(hermes): conectar el vault por filesystem
fix(meetings): corregir la captura de audio del sistema
docs(arch): registrar el modelo de delegación
research(mcp): documentar hallazgos
```

**Prohibido:** `push --force`, `reset --hard` sobre ramas compartidas, borrar ramas
ajenas, mergear a `main` sin revisión humana.

---

## 4. El vault NO usa worktrees

Si una tarea toca `~/Obsidian Vault`, se trabaja sobre su **único
directorio**, nunca sobre una copia. Razones:

- Obsidian indexa un solo directorio; un worktree sería invisible para la app.
- Hay ~119 enlaces internos entre las notas. Git mergea texto sin ver que un
  `[[enlace]]` quedó apuntando a una nota renombrada en otra rama.
- El conocimiento no se ramifica: no hay dos versiones de una decisión que fusionar.

**Un escritor a la vez** para `CLAUDE.md`, `Inicio.md`, `Pendientes.md`, plantillas,
scripts y vistas `.base` del vault. Las notas nuevas en carpetas distintas sí pueden
ir en paralelo.

Después de cualquier escritura en el vault:

```bash
cd ~/Documents/Obsidian\ Vault
git status --porcelain          # qué se tocó
"99 System/Scripts/check-links.sh"   # los enlaces siguen resolviendo
```

El segundo es el que importa: **detecta lo que git no puede ver.**

---

## 5. Fuentes válidas sobre Hermes

Solo estas dos:

- `https://github.com/NousResearch/hermes-agent`
- `https://hermes-agent.nousresearch.com`

Hay al menos cuatro dominios suplantando el proyecto (`hermes-agent.org`,
`hermes-ai.net`, `hermes-agent.ai`, `hermesagent.agency`). No son de Nous Research
y publican datos falsos. **No instalar ni copiar comandos de ninguno.**

---

## 6. Seguridad de Hermes

Configuración objetivo, no negociable sin discutirlo:

```yaml
approvals:
  mode: smart          # evalúa riesgo; lo peligroso se deniega solo
  cron_mode: deny      # las tareas programadas NO auto-aprueban
```

```bash
HERMES_WRITE_SAFE_ROOT="/Users/alfredocalvo/Obsidian Vault"
```

Con eso, Hermes solo puede escribir dentro del vault. Todo lo demás queda bloqueado
a nivel de herramienta.

**Nunca `--yolo`.** **Nunca `GATEWAY_ALLOW_ALL_USERS=true`.**

En el vault, nada se borra: `status: archived` y mover a `08 Archive`.
