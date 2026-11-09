---
name: commits
description: Propone una descomposición atómica en commits del trabajo sin commitear, como tabla revisable (nº, nombre en convención de commits y en inglés, archivos, qué hace). Úsala cuando el usuario pida ver cómo dividir los cambios en commits, revisar trabajo suelto antes de commitear, o invoque /commits. No ejecuta los commits.
---

# Descomposición en commits

Objetivo: convertir un diff grande y difícil de revisar en un mapa de commits atómicos, para que el
usuario pueda revisar el trabajo por conjuntos coherentes en vez de como un solo bloque.

## Procedimiento

1. **Levanta el estado real del repo.** No asumas nada desde la conversación:

   ```bash
   git status --porcelain -uall
   git diff --stat HEAD
   ```

   Incluye archivos sin trackear. Si no hay nada sin commitear, dilo en una línea y termina.

2. **Entiende cada cambio.** Lee el diff de los archivos relevantes (`git diff HEAD -- <ruta>`, y el
   contenido directo de los untracked). Necesitas saber *qué hace* cada cambio, no solo que cambió.

3. **Agrupa por unidad lógica**, no por carpeta ni por tipo de archivo. Un commit atómico es un cambio
   que se sostiene solo: compila, es revisable de forma independiente y tiene una intención única.
   Señales para separar: refactor mecánico vs. cambio de comportamiento; migración de esquema vs. código
   que la usa; fix vs. el test que lo cubre (si el test documenta el fix, van juntos); cambios de
   configuración/infra vs. lógica de aplicación.

4. **Ordena los commits** en la secuencia en que deberían aplicarse: las dependencias primero, de modo
   que cada punto intermedio del historial quede en un estado coherente.

5. **Entrega la tabla** con este encabezado textual:

   ```
   ## Descomposición en commits
   ```

   | # | Commit (conventional) | Archivos | Qué hace |
   |---|---|---|---|
   | 1 | `refactor(invoice): extract provider parsing` | `src/a.ts`, `src/b.ts` | Mueve el parseo a un módulo propio, sin cambio de comportamiento. Va primero porque 2 depende de él. |
   | 2 | `fix(invoice): reject zero cost for Novasur` | `src/c.ts`, `test/c.spec.ts` | Rechaza costos ≤ 0 en la carga. Incluye el test que documenta el caso. |

   Reglas de la tabla:

   - Nombre en convención de commits (`feat`, `fix`, `refactor`, `chore`, `test`, `docs`, `perf` + scope),
     con el subject **en inglés e imperativo** (`extract`, no `extrae` ni `extracted`). La columna
     "Qué hace" va en español. Ver §Idioma de los artefactos en `~/.claude/policies/agent-policies.md`.
   - La explicación es de 1–2 líneas: qué hace ese conjunto y por qué se agrupa así. Si el orden importa,
     dilo ahí.
   - Todo archivo sin commitear aparece en exactamente una fila. Si algo queda fuera a propósito
     (basura, archivos temporales, secretos que no deberían commitearse), lístalo aparte bajo
     **Fuera de la descomposición** con el motivo en una línea.
   - Si el trabajo es genuinamente **un solo** commit, emite el mismo encabezado y dilo en una línea con
     el nombre propuesto.

## Límites

- **Es una propuesta para revisión, no una instrucción.** No ejecutes `git commit`, `git add` ni nada que
  modifique el índice o el historial, a menos que el usuario lo pida explícitamente después de ver la tabla.
- No modifiques archivos para "arreglar" la descomposición (por ejemplo, partiendo un archivo en dos para
  que calce en commits distintos). Si un archivo mezcla cambios que idealmente irían en commits separados,
  déjalo en una fila y menciónalo en su explicación.
