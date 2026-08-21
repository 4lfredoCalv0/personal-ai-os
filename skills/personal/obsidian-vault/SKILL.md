---
name: obsidian-vault
description: Consultar y capturar en el vault de Obsidian de Alfredo
version: 1.0.0
platforms: [macos]
metadata:
  hermes:
    category: personal
    tags: [obsidian, memoria, conocimiento]
---

# Vault de Obsidian

El vault es la **fuente de verdad del conocimiento**: proyectos, decisiones,
personas, procedimientos y reuniones de Alfredo (Salus Saunas).

```
/Users/alfredocalvo/Documents/Obsidian Vault
```

## Antes de nada: leer las convenciones

**Leer `CLAUDE.md` en la raíz del vault al empezar cualquier trabajo con él.**
Ahí viven el esquema de frontmatter por tipo, las convenciones de nombres y las
reglas de escritura. Este archivo no las copia — se desincronizarían.

## Cuándo usar esta skill

Cuando Alfredo pregunte por proyectos, decisiones, tareas, reuniones, personas o
procedimientos. Cuando quiera registrar algo. Cuando diga *"ponte al día con X"*.

## Mapa de carpetas

```
00 Inbox/       captura sin decidir dónde va
01 Projects/    tiene final
02 Areas/       responsabilidad continua (las 12 bases de Airtable viven aquí)
03 Resources/   procedimientos y runbooks
04 People/      personas y empresas
05 Meetings/    una nota por reunión
06 Daily/       diario
07 Decisions/   por qué decidimos lo que decidimos
08 Archive/     terminado
99 System/      plantillas, vistas y scripts — no es contenido
```

## Orden de búsqueda

De más barato a más caro. **No hacer grep de todo el vault como primer recurso.**

1. **Fecha conocida** → `05 Meetings/2026-08-18*` o `06 Daily/2026-08-18.md`.
   El nombre lleva la fecha ISO al frente, así que es acceso directo.
2. **"¿Por qué decidimos X?"** → `07 Decisions/`. La carpeta existe para eso.
3. **Un proyecto o tema** → buscar en `01 Projects/` y `02 Areas/`, luego seguir
   los enlaces `[[...]]` de esa nota.
4. **Una persona** → `04 People/Nombre.md`, y buscar quién la enlaza para ver
   cada reunión y tarea suya.
5. **Un procedimiento** → `03 Resources/`.
6. Último recurso: grep de todo.

## Backlinks: cómo se encuentra el contexto

El vault no mantiene índices a mano. Una reunión apunta a su proyecto, y el
proyecto "sabe" de la reunión porque alguien la enlazó.

Para responder *"¿qué ha pasado con el proyecto X?"*, buscar qué archivos
contienen `[[X]]`:

```bash
grep -rl '\[\[Gmail Assistant\]\]' "/Users/alfredocalvo/Documents/Obsidian Vault" --include='*.md'
```

## Tareas

Son casillas inline `- [ ]` dentro de la nota donde nacieron, con fecha opcional
`📅 2026-08-25` y prioridad `⏫`. **No existe una carpeta de tareas.**

Para listarlas sin contar los ejemplos de la documentación:

```bash
"/Users/alfredocalvo/Documents/Obsidian Vault/99 System/Scripts/pendientes.sh"
```

## Reglas de escritura — FASE ACTUAL

**Solo se puede escribir en `00 Inbox/`.** Cualquier nota nueva lleva:

```yaml
---
type: unverified
created: 2026-08-21
---
```

Alfredo la revisa y la asciende. **Nada de lo que yo infiera es conocimiento
permanente hasta que un humano lo confirme.**

Fuera del Inbox: **preguntar antes**. Y nunca borrar nada — se marca
`status: archived` y se mueve a `08 Archive`.

## Después de escribir, siempre

```bash
cd "/Users/alfredocalvo/Documents/Obsidian Vault"
git status --porcelain
"99 System/Scripts/check-links.sh"
```

El segundo detecta enlaces rotos, que es **lo que git no puede ver**: si una nota
se renombra, git mergea limpio y los `[[enlaces]]` quedan apuntando al vacío.

## Trampas

- **Nunca escribir credenciales en el vault.** Es texto plano y va a GitHub.
- **No duplicar datos de Airtable.** Airtable guarda registros, el vault guarda
  razonamiento. Guardar el `recXXXXXXXX`, no una copia de la tabla.
- **Los correos del equipo casi nunca son `nombre@`.** George es `service@`,
  Verónica `tc2@`, Sergio `service3@`. Verificar en `04 People/`.
- El vault **no usa git worktrees**. Un solo directorio de trabajo.

## Verificación

Una consulta salió bien si se llegó a la nota concreta en uno o dos pasos citando
su ruta, no si se resumió el vault entero.
