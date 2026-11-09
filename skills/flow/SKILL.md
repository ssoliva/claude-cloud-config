---
name: flow
description: Flujo completo de construcción y shipeo de una feature — encuadre, plan criticado por otro modelo, implementación, validación, revisión del árbol sucio y composición de commits antes de abrir una única PR. Usar cuando el usuario invoque /flow.
disable-model-invocation: false
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - Agent
  - AskUserQuestion
  - Skill
---

# flow

Flujo de punta a punta para construir y shipear un cambio, en un solo hilo.

**Vos implementás.** No delegues la implementación a un subagente: el hilo principal
mantiene el contexto de primera mano y es lo que hace que tus explicaciones al
usuario tengan sustento. Hay **un solo** subagente en todo el flujo (fase 7) y es
read-only.

## Las dos reglas duras

Todo lo demás es criterio. Estas dos no:

1. **Nada se commitea antes de la Puerta 2.** La fase 5 implementa sobre el árbol
   sucio y ahí se queda. Los hallazgos de la fase 7 y los cambios que pida el
   usuario en la Puerta 2 **se funden en el trabajo**, no generan commits `fix:`
   encima. El usuario revisa código, no historia.
2. **La PR se abre una vez, después de la Puerta 3.** Nunca antes, nunca de nuevo.
   Un revisor automático (CodeRabbit y equivalentes) gasta su cuota por push: si la
   PR nace temprano y recibe N rondas de fix, las últimas iteraciones no se revisan.

Si alguna fase te tienta a romper una de las dos, parás y le preguntás al usuario.

## Economía

El flujo existe porque el pipeline completo de un repo es caro. No lo reproduzcas:

- **No cargues el `reference` de una fase que no vas a correr.** Cada uno entra
  cuando su fase arranca y no antes.
- **El perfil del repo se deriva una vez por repo**, nunca por feature.
- **No releas archivos que ya leíste en esta sesión**, ni repitas en el chat lo que
  ya escribiste en el archivo de estado.
- **Un solo subagente**, en la fase 7, read-only, en paralelo con otro trabajo.
- Cuando el triage diga que el cambio es trivial, **corré el carril rápido**. La
  ceremonia completa sobre un typo es el fracaso de esta skill, no su cumplimiento.

## Estado

Un archivo por feature, en `~/.claude/work/<repo>/<slug>.md`. Es lo que permite
cortar el flujo en sesiones distintas y retomarlo con `/flow build --resume`.

```markdown
# <slug>
repo: <ruta> · rama: <branch> · base: <rama base> · creado: <YYYY-MM-DD>
fase: <n>

## Contrato
objetivo / no-objetivos / criterio de aceptación / riesgos

## Plan
(fase 3)

## Crítica externa
(fase 4)

## Bitácora
- fase N — qué pasó, qué decidió el usuario
```

Escribilo al cerrar cada fase, en una pasada corta. Es un checkpoint, no un informe.

## Subcomandos

| Invocación | Qué corre |
|---|---|
| `/flow <descripción>` | fases 0 → 8, con sus tres puertas |
| `/flow plan <descripción>` | fases 0 → 4 y para en la Puerta 1 |
| `/flow build` | fases 5 → 7, desde el estado guardado |
| `/flow ship` | fase 8, desde el estado guardado |
| `/flow <lo que sea> --resume` | lee el estado, audita el árbol y sigue desde la primera fase pendiente |
| `… --esqueleto=propio` | el cuerpo de la PR sale de `~/.claude/rules/pr-body.md`, ignorando la plantilla del repo |

Con `--resume` **auditá el árbol antes de confiar en el archivo de estado**:
`git status --short` y `git log --oneline <base>..HEAD`. Si el árbol y la bitácora
se contradicen, decíselo al usuario y que él resuelva; no elijas vos en silencio.

---

# Las fases

## Fase 0 — Triage

Decidí el carril. **Carril rápido** si se cumple todo: toca ≤2 archivos, no agrega
superficie de runtime nueva, no hay migración de datos ni de esquema, y no cambia
ningún contrato hacia afuera (API, evento, columna, variable de entorno). Un typo,
un ajuste de copy, el bump de una constante.

En carril rápido: implementá, corré la validación local del perfil, armá la tabla de
commits y parás en la Puerta 3. Nada más.

**Ante la duda, no es trivial.** Y si el usuario ya te dijo que es chico, igual
corré este criterio: la estimación de tamaño del usuario es un input, no el veredicto.

## Fase 1 — Encuadre

Buscá las incertidumbres de alto nivel **antes** de que existan archivos.

Una sola ronda de `AskUserQuestion`, **máximo 4 preguntas**, y sólo las que cambian
el plan. Si la respuesta a una pregunta no cambiaría ningún archivo a tocar, no la
hagas: asumí el default razonable y declaralo.

Salida: el bloque `## Contrato` del archivo de estado, en ≤10 líneas.

- **objetivo** — qué tiene que ser cierto cuando esto esté listo
- **no-objetivos** — lo que explícitamente queda afuera de este cambio
- **criterio de aceptación** — observable, no "funciona bien"
- **riesgos** — lo que sabés que puede salir mal

## Fase 2 — Perfil del repo

Si `~/.claude/profiles/<slug-del-repo>.md` existe, cargalo y seguí a la fase 3. **No
lo re-derives.**

Si no existe, cargá `references/perfil-repo.md`, derivalo, confirmalo con el usuario
en una sola pasada y guardalo. Es lo único de esta fase que se paga una vez por repo
en vez de una vez por feature.

## Fase 3 — Plan

En el hilo principal. Escribí en `## Plan`:

- **archivos a tocar**, cada uno con el cambio en una línea
- **orden de implementación**, si importa
- **tests a escribir** — nombre y qué ejercita cada uno. Todo cambio con
  comportamiento observable lleva test; si este no lo lleva, escribí por qué
- **cómo se valida** — tomado del perfil, no inventado acá
- **orden de despliegue** — si el código nuevo depende de un esquema, índice o
  variable que este mismo cambio crea, decilo: se parte en dos entregas
- **qué queda afuera**

## Fase 4 — Crítica externa

Cargá `references/critica.md` y corré `codex` una vez sobre el plan.

Si `codex` no está instalado, decílo en una línea y pasá a la Puerta 1 sin crítica.
No lo reemplaces por un subagente de Claude: el valor era que fuera otro modelo.

Resumí la crítica en `## Crítica externa`: qué levantó, y para cada punto si lo
tomás, lo descartás y por qué.

---

### ══ PUERTA 1 — el plan ══

Mostrale al usuario el contrato, el plan y la crítica, **en el chat, en forma
legible** — no lo mandes a leer el archivo. Esperá su OK explícito.

Si pide ajustes, ajustá y volvé a mostrar. No arranques la fase 5 sin OK.

---

## Fase 5 — Implementación

En el hilo principal, siguiendo el plan, **con los tests que el plan declaró**, y
**sin commitear nada** (regla dura 1).

Si mientras implementás descubrís que el plan está mal — un supuesto que no se
sostiene, un archivo que no existe, un acoplamiento que no viste —, **no improvises**:
parás, se lo decís al usuario con la evidencia, y volvés a la Puerta 1 con el plan
corregido. Un plan roto parchado sobre la marcha es exactamente lo que produce la PR
irrevisable.

Respetá las políticas del usuario para lo que queda escrito en el repo:
identificadores, comentarios y nombres de test en inglés; comentarios sólo si caen en
una categoría justificable.

## Fase 6 — Validación

Corré **sin preguntar** todo lo que el perfil declare como validación local: tests,
build, typecheck, linter, levantar la app.

**Todo lo que salga de la máquina, se propone y se para.** Desplegar a staging,
invocar un lambda, tocar una cuenta cloud, escribir en una base compartida: armás el
comando exacto, se lo mostrás al usuario y te detenés. No lo corras aunque el perfil
tenga el comando guardado — tenerlo documentado no es autorización.

Si la validación falla, arreglá y volvé a correr. Sigue sin haber commits.

## Fase 7 — Revisión y composición

**Lanzá las dos cosas a la vez, en un solo mensaje:**

1. Un subagente `general-purpose` con `model: "opus"`, **read-only**, que revise el
   diff del árbol sucio. Es el único subagente del flujo, y existe por una razón
   precisa: vos escribiste este código y no sos un buen revisor de tu propio trabajo.
2. Vos, en paralelo, armando la tabla de commits.

Cargá `references/review-y-commits.md` para el contrato del subagente y el formato de
la tabla.

Cuando vuelva el subagente, fusioná: presentá los hallazgos **y** la tabla juntos.

---

### ══ PUERTA 2 — el código ══

El usuario revisa el trabajo **por conjuntos coherentes** (las filas de la tabla), no
como un bloque. Iterá con él hasta cerrar.

**Los cambios de esta puerta se funden en el trabajo** (regla dura 1). Si un cambio
hace que la tabla ya no describa el diff, rehacé la tabla. Todavía no hay commits, así
que no hay historia que arreglar después.

---

## Fase 8 — Shipeo

Recién acá se escribe historia.

1. Commiteá **según la tabla aprobada**, una fila por commit, en orden. Conventional
   Commits, subject en inglés e imperativo.
2. Verificá que el árbol quedó limpio y que el diff commiteado es el que se aprobó:
   `git status --short` y `git log --oneline <base>..HEAD`.
3. Pusheá la rama.

### ══ PUERTA 3 — la PR ══

Abrir una PR es una acción hacia afuera: **necesita OK explícito, siempre**, incluso
si el usuario ya aprobó todo lo anterior.

Con el OK, cargá `references/pr.md` y abrí **una** PR. Una sola vez.

## Fase 9 — Revisión automática

Fuera de alcance. Resolver los comentarios de CodeRabbit es otra skill; acá sólo
avisá que la PR quedó abierta y que la revisión automática va a correr sobre ella.

---

## Si el flujo se corta

El archivo de estado es la fuente. Al retomar, audití el árbol (`git status --short`,
`git log --oneline <base>..HEAD`) y **creele al árbol, no a la bitácora**. Si se
contradicen, mostrale la contradicción al usuario.

Nunca arranques una fase nueva sobre un árbol que no auditaste.
