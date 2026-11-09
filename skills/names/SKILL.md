---
name: names
description: Audita los nombres que introduce el diff actual (o un archivo, ruta o rango dado) contra la política de nombres y propone una tabla de renombres para aplicar. Úsala cuando pidan revisar o arreglar los nombres del código.
disable-model-invocation: true
allowed-tools: Read, Grep, Glob, Bash(git diff:*), Bash(git status:*), Bash(git log:*)
---

# Auditoría de nombres

Alcance por defecto: los identificadores **introducidos o modificados** por el trabajo sin commitear
(`git diff HEAD`). Con argumento, ese alcance: una ruta (`/names cloud/lib/x.ts`), un rango
(`/names main...HEAD`) o `--all` para un archivo completo incluyendo lo preexistente.

## Procedimiento

1. Obtén el diff del alcance. Si no hay cambios, dilo en una línea y termina.
2. Lista los identificadores declarados en las líneas agregadas: variables, parámetros, funciones,
   tipos, interfaces, campos propios. Descarta los que replican un contrato externo (campos de API,
   columnas, payloads) y los nombres cortos legítimos por convención.
3. Aplica los tests de `~/.claude/policies/agent-policies.md` (§Nombres) y los casos de
   `~/.claude/rules/naming.md`. Anota qué regla viola cada nombre.
4. Antes de proponer un reemplazo, `grep` el concepto en el repo y usa el término que ya existe. No
   introduzcas un sinónimo nuevo.
5. Entrega una sola tabla, ordenada de más grave a menos (procedencia > exactitud > ruido de tipo >
   resto):

   | archivo:línea | actual | propuesto | regla |
   |---|---|---|---|

6. No apliques nada hasta que el usuario confirme. Al aplicar: renombra todas las ocurrencias del
   scope, corre el type-check y el lint del proyecto, y reporta el resultado real.

## Reglas de la auditoría

- Solo nombres del diff. Los preexistentes que el diff no toca van en una sección aparte, marcada
  como opcional y para su propio commit.
- Si un nombre está bien, no lo propongas "por variar". Una tabla vacía es un resultado válido.
- Cada fila propone un nombre concreto, no una crítica. Sin nombre propuesto no hay fila.
