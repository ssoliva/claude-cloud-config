# Políticas de desarrollo

Aplican a todos los repositorios. Tienen precedencia sobre los defaults del agente.

## Idioma de los artefactos

La conversación es en español; lo que queda escrito en el repo, en general no. Idioma por artefacto:

| Artefacto | Idioma |
|---|---|
| Identificadores, tipos, nombres de test | inglés |
| Comentarios y docstrings | inglés |
| Mensaje de commit: subject y body | inglés, imperativo (`add`, no `added` ni `adds`) |
| Nombre de rama | inglés (`feat/user-auth`, `fix/login-redirect`) |
| Título de PR | inglés |
| Descripción de PR | español |
| Respuestas en el chat, planes, tabla de commits | español |
| Copy visible al usuario final, claves i18n | español |
| Documentación (`dev-wiki/`, `docs/`) | el del archivo que estás editando |

Que el dominio sea español (`receta`, `bodega`, `despacho`, `convenio`) no autoriza prosa en español: el
término del dominio se usa como término dentro de una frase en inglés, y si el repo ya nombra así el
concepto, se respeta (§Nombres, un término por concepto).

**El idioma de esta política no es el idioma de la salida.** Estar leyendo instrucciones en español, o
venir de un turno conversado en español, no es razón para escribir un comentario o un commit en español.

## Comentarios en el código

**Idioma: inglés, siempre.** Ver §Idioma de los artefactos.

**Default: cero comentarios.** El código se explica con nombres descriptivos y buena estructura. Un
comentario no es el acompañamiento por defecto de un bloque: es una excepción que hay que justificar.

Un comentario se permite **solo** si cae en una de estas cuatro categorías cerradas. No hay quinta:

1. **Formato o unidad** que el tipo no expresa (`YYYY-MM-DD`, `ms`, `E.164`): va **inline, a la derecha
   del campo, y solo el token del formato** — sin verbo, sin oración, sin repetir el nombre del campo.
2. **Workaround o restricción externa**: por qué el código no es lo obvio (bug de una librería, límite de
   una API, orden que impone un proveedor).
3. **Decisión contraintuitiva**: el lector va a querer "arreglarlo"; el comentario dice por qué no.
4. **Invariante no verificable localmente**: algo que se cumple por un contrato que vive en otro archivo.

"Aclara", "documenta", "ayuda al lector" y "gana valor" **no** son categorías. Si el comentario parafrasea
la línea siguiente, no va. Si dice **qué** hace el código en vez de **por qué**, no va.

Cuando corresponda: **máximo 1–2 líneas**.

**Presupuesto.** A lo más 1 comentario por cada ~30 líneas nuevas. Más de 3 en un diff es señal de que
faltan nombres o funciones más chicas, no más prosa.

**Verificación antes de cerrar.** Relee tu propio diff y borra todo comentario que no puedas clasificar
diciendo el número de la categoría. Este paso no es opcional.

Estas reglas tienen precedencia sobre imitar la densidad de comentarios del archivo existente: en un
archivo ya sobre-comentado, el código nuevo sigue esta política, no la del archivo.

Al modificar código existente, simplifica los comentarios **solo dentro de las funciones o bloques que ya
estás tocando** por el trabajo en curso. No limpies comentarios del resto del archivo: eso mete ruido de
diff en código no relacionado y dificulta la revisión. La limpieza completa de un archivo es un pedido
aparte y su propio commit.

## Nombres

Un nombre debe permitir predecir qué contiene el valor **sin leer la línea que lo produjo**.

- **Referente, no procedencia.** Nunca `result`, `data`, `res`, `response`, `output`, `info`, `payload`,
  `obj`, `val`, `temp`, `item` como nombre de una variable. Test: tapa el lado derecho de la asignación;
  si el nombre ya no dice qué hay dentro, está mal.
- **Exactitud.** El nombre denota exactamente el conjunto que contiene: `activeOrders`, no `orders`.
- **Largo proporcional al alcance.** `i`, `dx` valen en un bloque de 3 líneas; en scope de función,
  de módulo o exportado, nombre completo.
- **Sin ruido de tipo.** Nada de sufijos `Data`, `Info`, `Object`, `Manager`, `Helper`, `Wrapper`: el
  tipo ya lo dice.
- **Booleano = predicado afirmativo.** `is`/`has`/`should`/`can`, y nunca negado (`isHidden`, no
  `isNotVisible`).
- **Unidad o marco de referencia en el nombre** cuando el tipo no los lleva: `maxAgeMs`, `createdAtSec`,
  `priceClp`, `distanceKm`.
- **Un término por concepto.** Antes de inventar una palabra para un concepto del dominio, busca en el
  repo cómo se llama ya (`grep`) y usa esa. No introduzcas sinónimos.
- Si necesitas un comentario para decir **qué** contiene una variable, el nombre falló: arregla el nombre,
  no agregues el comentario.

Excepción única: nombres que replican un contrato externo (campos de una API, columnas, payloads) se
escriben como los define ese contrato, aunque viole lo de arriba.

Aplica a lo que escribes. No renombres código existente que no estás tocando: eso mete ruido de diff y va
en su propio commit, igual que la limpieza de comentarios.

## Cierre de implementaciones grandes

Cuando **termines** una implementación que deja **4 o más archivos sin commitear**, cierra la respuesta con
una descomposición atómica en commits, bajo este encabezado:

```
## Descomposición en commits
```

Criterio de "terminaste": entregaste el alcance que se pidió y no quedan pasos pendientes de tu parte. Si
la implementación va a medias, quedaste esperando una decisión, o el turno fue una pregunta sobre cambios
en curso, **no** la emitas: el usuario la pide cuando quiera con `/commits`. Tampoco la repitas si ya la
entregaste en este mismo mensaje.

Debajo, una tabla con la descomposición atómica propuesta:

| # | Commit (conventional) | Archivos | Qué hace |
|---|---|---|---|
| 1 | `fix(scope): ...` | `a.ts`, `b.ts` | 1–2 líneas explicando el cambio |

Reglas:

- Cada fila es un commit atómico y revisable por separado, en el orden en que deberían aplicarse.
- Nombres en convención de commits (`feat`, `fix`, `refactor`, `chore`, `test`, `docs` + scope), con el
  subject **en inglés** (§Idioma de los artefactos).
- La explicación es de 1–2 líneas: qué hace ese conjunto de cambios y por qué se agrupa así.
- Si el trabajo es genuinamente **un solo** commit, emite el mismo encabezado y dilo en una línea, con el
  nombre del commit propuesto. El encabezado va siempre.
- Esto es una **propuesta para revisión**, no una instrucción para commitear. No ejecutes los commits a
  menos que se te pidan explícitamente.
