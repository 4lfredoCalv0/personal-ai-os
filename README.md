# personal-ai-os

Notas de diseño de mi asistente personal: **Claude Code como agente, un vault de
Obsidian como memoria, y n8n para lo que tiene que correr solo.**

Este repositorio es sobre todo un documento de arquitectura y un registro de
decisiones. El código operativo vive en otras partes — el vault tiene su propio
repositorio, y las automatizaciones viven en n8n.

## La decisión principal: quitar la capa de orquestación

El diseño original tenía un orquestador propio, **Hermes**, en el centro:
recibiría mensajes por Telegram, decidiría qué hacer, delegaría a Claude Code
por subprocess, y escribiría en el vault.

Se instaló, se configuró y se probó. **Se descartó el 21-ago-2026.**

La razón: la capa que iba a añadir ya existía. Se justificó a Hermes como "capa
conversacional", pero Claude Code ya lo es, y tiene terminal, escritorio y web.
Al revisar capacidad por capacidad, la única que Hermes aportaba de verdad era
Telegram y WhatsApp. Todo lo demás —conversar, recuperar contexto, delegar,
capturar reuniones, automatizar, proactividad— ya estaba resuelto por Claude
Code, por un plugin de Obsidian o por el MCP de n8n.

Construir un orquestador para cubrir un hueco de mensajería no compensaba
mantener una pieza más en el camino crítico.

```
        TÚ  (terminal · escritorio · Obsidian)
                     │
            ┌────────▼────────┐
            │   CLAUDE CODE   │  agente · tareas programadas · MCP
            └────┬───────┬────┘
                 │       └──────────► n8n  (lo que corre sin nadie delante)
                 ▼
          OBSIDIAN VAULT  ◄── fuente de verdad
                 ▲
         solo tras revisión humana
```

## El hallazgo que ahorró el trabajo

La Parte 1 del documento de arquitectura resuelve un problema concreto: **correr
varias sesiones de agente en paralelo sobre el mismo repositorio sin que se
pisen.**

Dos sesiones editando el mismo checkout se sobrescriben sin avisar. La solución
que iba a construir —scripts para crear, listar y cerrar worktrees— resultó
innecesaria: Claude Code trae worktrees nativos con aislamiento aplicado a nivel
de herramienta.

```bash
claude --worktree hermes
```

No es una convención que el agente pueda ignorar. Bloquea activamente las
ediciones fuera del worktree, los comandos cuyo directorio resuelva al checkout
principal, la redirección de git (`git -C`, `--git-dir`, `GIT_DIR`), y las
construcciones de shell que no puede trazar sin ejecutarlas. Un agente no se
escapa de su worktree aunque se lo pidas.

Tres scripts que estaban planeados no se escribieron.

## Qué hay en el repositorio

| Carpeta | Contenido |
|---|---|
| `docs/arquitectura.md` | El documento completo. Parte 1: trabajo en paralelo, worktrees, política de escritura del vault, convenciones de git. Parte 2: la arquitectura de Hermes que se descartó, con sus riesgos declarados |
| `CLAUDE.md` | Contexto e instrucciones para las sesiones de Claude Code sobre este repo |
| `hermes/` | Configuración de la aproximación descartada. Se conserva como registro, no está en uso |

El vault vive aparte, en `~/Obsidian Vault`, con su propio repositorio.

## Por qué está publicado

Menos por el código —hay poco— y más por el razonamiento: qué se evaluó, qué se
construyó, y por qué se quitó. La decisión de borrar una pieza que ya funcionaba
suele estar peor documentada que la de construirla.
