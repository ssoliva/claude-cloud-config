---
name: rebase-stack
version: 2.0.0
description: Sincronizar una rama o un stack contra su base (gh stack rebase, rebase --onto, reset) resolviendo conflictos sin perder la intención de ninguno de los dos lados, y aterrizar un stack completo con squash. Usar cuando el usuario pida hacer rebase, reset, "actualizar la base", "traer los cambios de dev/main", mover un stack de ramas, o mergear/aterrizar un stack.
allowed-tools:
  - Bash
  - Read
  - Edit
  - Grep
  - Glob
---

# rebase-stack

Dos modos, mismo criterio:

- **sync** (default) — actualizar la rama o el stack contra su base, resolviendo conflictos con criterio.
- **land** — aterrizar un stack completo: mergear sus PRs de abajo hacia arriba, con squash, en una sola operación.

Argumento: la rama base (`/rebase-stack dev`) o `land` (`/rebase-stack land`).

La mecánica de git la hace `gh stack` cuando hay un stack; lo que aporta esta skill es el criterio de
resolución, la verificación de que no se perdió trabajo, y el reporte. Los comandos, flags y exit
codes de la extensión son autoridad de la skill oficial `gh-stack` (`~/.claude/skills/gh-stack/`):
consultarla en vez de duplicarla acá.

## Contrato

Estas cuatro reglas aplican durante toda la ejecución. Si alguna choca con la instrucción de avanzar rápido, gana la regla.

1. **Preservar la intención, no el texto.** El objetivo no es unir los dos lados mecánicamente: es que el resultado haga lo que ambas ramas querían hacer. A veces eso significa descartar un lado (la base borró el archivo, o ya implementó lo mismo de otra forma, o refactorizó lo que tú tocaste). Unir todo produce código que compila y está mal.
2. **Prohibido resolver a ciegas.** No usar `-X ours`, `-X theirs`, `git checkout --ours/--theirs` ni `git rebase --skip` como atajo para que deje de conflictuar. Solo con justificación explícita del caso concreto. Recordar: **en un rebase `ours` es la base y `theirs` es tu commit** — está invertido respecto a un merge. Confundirlos es la forma #1 de perder trabajo.
3. **Preguntar durante, no al final.** Si un conflicto no está claro, deja el rebase pausado y pregunta ahí mismo. Con el rebase detenido el contexto está vivo y el usuario puede mirar el archivo; al final ya está todo aplanado y la pregunta es inútil. No saber es un resultado aceptable; adivinar no lo es.
4. **Cero push y cero merge sin OK explícito.** Nunca `push --force`. Al terminar, proponer el comando y esperar. Esto veta tres comandos durante el diagnóstico y la resolución, porque escriben en el remoto sin preguntar:

   | Comando | Qué hace sin pedir permiso |
   |---|---|
   | `gh stack sync` | pushea todas las ramas (`--force-with-lease`) en su paso 5 |
   | `gh stack push` | pushea todas las ramas del stack |
   | `gh stack merge` | mergea PRs en la base |

   Para actualizar se usa `gh stack rebase`, que es puramente local. `sync`, `push` y `merge` se
   proponen al final, o se corren solo si el usuario los pidió explícitamente en su mensaje.

---

## Fase 0 — Elegir la ruta

```
gh stack --version >/dev/null 2>&1; echo "ext: $?"
gh stack view --json >/dev/null 2>&1; echo "stack: $?"
```

Ojo con el exit code: `gh stack view --json | head` devuelve el status de `head`, no el de `gh`.
Redirigir y leer `$?` aparte.

| Resultado | Ruta |
|---|---|
| `stack: 0` | **Ruta A** — hay stack rastreado localmente (`.git/gh-stack`) |
| `stack: 2` | No hay stack local. Antes de caer a B, ver si existe en GitHub (abajo) |
| `stack: 9` | El repo no tiene la preview de stacked PRs habilitada → **Ruta B** |
| `ext: != 0` | La extensión no está instalada (`gh extension install github/gh-stack`) → **Ruta B**, o instalarla si el usuario quiere |

Con `stack: 2` la rama puede igual pertenecer a un stack creado en la web o por otra máquina. Pero
listar los stacks del repo no alcanza: el endpoint devuelve **todos**, incluidos los de otra gente.
Hay que matchear por la rama actual y adoptar solo si aparece en un stack abierto:

```
BR=$(git branch --show-current)
gh api repos/{owner}/{repo}/stacks 2>/dev/null \
  | jq -r --arg br "$BR" '.[] | select(.open) | select(any(.pull_requests[]; .head.ref == $br)) | "stack #\(.number) base=\(.base.ref) prs=\(.pull_requests | map(.number) | join(","))"'
```

- Devuelve un stack → `gh stack checkout <stack-number>` lo baja, lo rastrea local, y se sigue por Ruta A.
- No devuelve nada → **Ruta B**. Nunca hacer `checkout` de un stack que no contiene la rama actual:
  es el trabajo de otra persona y crea tracking local sobre ramas ajenas.

Ruta B es para: repos sin la preview, ramas sueltas que nadie stackeó, y remotos que no son GitHub.
Es una ruta de primera clase, no un modo degradado: para una rama sola colgando de `dev` es
exactamente lo que hay que hacer.

## Ruta A — hay stack

### A1. Diagnóstico (termina ANTES de ejecutar nada)

```
gh stack view --json
```

Por rama: `name`, `base`, `isMerged`, `isQueued`, `needsRebase`, y `pr` (`number`, `state`).
`needsRebase: true` significa que el tip del padre ya no es ancestro de esa rama.

Anotar el SHA original de **cada** rama del stack (`git rev-parse <rama>`) antes de tocar nada. Sirve
para dos cosas: verificar con `range-diff` al final, y volver atrás una vez que el rebase ya terminó,
cuando `--abort` ya no aplica.

**Salida de la fase:** decirle al usuario qué ramas se van a mover, contra qué base, y por qué.
Recién entonces ejecutar.

### A2. Actualizar

```
gh stack rebase                 # todo el stack, de trunk hacia arriba
gh stack rebase --upstack       # de la rama actual hacia arriba (después de editar una capa baja)
gh stack rebase --downstack     # de trunk hasta la rama actual
gh stack rebase --no-trunk      # alinear las ramas entre sí, sin fetch ni trunk
```

Lo que ya no hay que decidir a mano, y antes era la tabla de decisión de esta skill:

- **Padre mergeado con squash.** `gh stack rebase` lo detecta y replica con `--onto` contra el target
  correcto, salteando la rama mergeada. Los conflictos fantasma contra tu propio código ya mergeado
  no aparecen; no hace falta el `git reset --hard origin/<base>` manual.
- **Orden del cascade.** Rebasea de trunk hacia arriba, en orden, sin que haya que iterar rama por rama.
- **rerere.** `gh stack init` lo deja activado. Si el stack se adoptó de otra forma, verificar:
  `git config rerere.enabled true`. Importa: en un stack el mismo conflicto reaparece capa tras capa.

Exit codes que hay que manejar (el resto está en la skill `gh-stack`):

| Código | Qué hacer |
|---|---|
| 3 | Conflicto. `rebase` queda pausado con los archivos listados → pasar a **Resolución**. Si el 3 vino de un `sync`, el stack ya fue restaurado entero: correr `gh stack rebase` para reproducir el conflicto y poder resolverlo. |
| 7 | Ya hay un rebase en curso: `--continue` o `--abort`, no empezar otro. |
| 8 | El archivo del stack está lockeado por otro proceso: esperar ~5s y reintentar. |

Rollback, y dejarlo escrito antes de empezar:

```
gh stack rebase --abort                  # restaura TODAS las ramas del stack, no solo la actual
git reset --hard <SHA-original>          # por rama, si el rebase ya terminó y quedó mal
```

Nunca `gh stack modify`: es TUI y bloquea al agente para siempre. Para reordenar o sacar capas, ver
`references/troubleshooting.md` de la skill oficial (`unstack` + `init` + `submit`).

## Ruta B — git plano (sin stack)

### B0. ¿Esto quiere ser un stack?

Detección barata, no bloquea nada: se anota y se propone en el reporte, después de terminar el
trabajo que el usuario pidió.

```
git branch --contains <rama>                          # ramas locales que la contienen (descartar la propia y la base)
gh pr list --base <rama> --limit 200 --json number,headRefName   # PRs abiertos que la usan de base
```

- **Sin dependientes** — una rama sola sobre `dev`. No hay nada que ganar con stacks: seguir con la
  Ruta B y no mencionar el tema. Esta es la mayoría de los casos.
- **Con dependientes**, y el repo soporta stacks (la Fase 0 no dio `9`) — es justo el caso donde el
  loop manual duele. Proponer la adopción en el reporte, sin ejecutarla:

  ```
  gh stack init --base dev <rama-abajo> <rama-arriba> ...   # local; adopta ramas que ya existen
  gh stack submit --auto                                     # crea/actualiza PRs y linkea (remoto: pedir OK)
  gh stack link <pr-abajo> <pr-arriba> ...                   # si los PRs ya existen: solo linkea, sin tracking local
  ```

  `init` adopta ramas existentes y `link` no toca estado local, así que la adopción no rehace el
  trabajo. Requisito: la ancestría ya tiene que ser lineal, cada rama conteniendo el tip de la de
  abajo. Si no lo está, primero se ordena con esta misma Ruta B y se adopta después.

### B1. Diagnóstico

Determinar la base: argumento del usuario → `git rev-parse --abbrev-ref @{upstream}` → default branch
del remoto. Si sigue ambiguo, preguntar.

```
git fetch origin --prune
git status --porcelain          # working tree debe estar limpio
git log --oneline <base>..HEAD  # commits propios de la rama
git log --oneline HEAD..<base>  # qué trae la base de nuevo
git cherry -v <base> HEAD       # '-' = ya está en la base, '+' = falta
```

| Situación | Operación |
|---|---|
| Los commits de la rama no están en la base (`+` en `cherry`) | `git rebase <base>` |
| Los commits ya están en la base — típico de PR mergeado con squash (`-` en `cherry`, o el diff de la rama ya vive en la base) | `git reset --hard origin/<base>` |
| La rama cuelga de un padre cuya base cambió | `git rebase --onto <nueva-base> <vieja-base> <rama>` |
| Se quiere mover varias ramas de una | `git rebase --update-refs` (requiere git ≥ 2.38) |

Por qué importa: si el PR padre se mergeó **con squash**, los commits propios ya están en la base pero
con otro SHA y otro patch-id. Un `git rebase` los reproduce y genera **conflictos fantasma** —
conflictos contra tu propio código ya mergeado. Resolverlos "sin perder nada" es precisamente lo que
duplica código. Ahí el arreglo es `reset`, no resolver conflictos.

### B2. Red de seguridad y ejecución

Antes de ejecutar:

```
git rev-parse HEAD                                   # anotar: SHA original
git branch backup/<rama>-<algo-identificable> HEAD
git config rerere.enabled true
git rebase --abort                                   # rollback si está en curso
git reset --hard <SHA-original>                      # rollback si ya terminó
```

Si el working tree está sucio: `git stash push -u` y anotar que hay que hacer `pop` al final (y
avisarlo en el reporte), o preguntar si prefiere commitear primero.

---

## Resolución (común a las dos rutas)

Loop por cada conflicto. Para cada archivo, antes de tocarlo:

```
git log --oneline --left-right <base>...REBASE_HEAD -- <archivo>
git show REBASE_HEAD -- <archivo>            # qué quería mi commit
git diff <base-vieja>..<base> -- <archivo>   # qué quería la base
```

Reglas de resolución:

- Resolver en el contexto **de ese commit**, no del estado final de la rama. No adelantar cambios de commits posteriores del rebase.
- Al terminar cada archivo: cero markers (`git diff --check`), y releerlo completo — no solo el hunk — para ver si quedó lógica duplicada o un import huérfano.
- Anotar la decisión **en el momento** (archivo, qué quería cada lado, qué se hizo, por qué, confianza). El reporte final se arma con estas notas, no reconstruyendo al final: reconstruido sale una narrativa plausible en vez de lo que pasó.
- Continuar: `git add <archivos>` y después `gh stack rebase --continue` (Ruta A) o `git rebase --continue` (Ruta B).

**Gatillos de escalamiento — parar y preguntar, no decidir:**

- Los dos lados cambian la misma lógica con intenciones distintas e incompatibles.
- Un lado borra/renombra lo que el otro modifica.
- Migraciones de base de datos: nunca renumerar ni reordenar sin preguntar; el orden es semántico.
- Config de ambiente, secretos, feature flags.
- Más de ~3 hunks conflictivos en un mismo archivo (señal de refactor upstream: puede convenir rehacer el cambio sobre la base nueva en vez de mergearlo).
- Cualquier caso donde la resolución "correcta" implique borrar trabajo de un lado.

**Archivos que no se mergean a mano:**

- Lockfiles (`yarn.lock`, `package-lock.json`, `Podfile.lock`): tomar la versión de la base y **regenerar** con el gestor del proyecto. Nunca hunk por hunk.
- Archivos generados / snapshots / build artifacts: regenerar con su comando, no editar.

## Verificación (común a las dos rutas)

Que no queden markers no significa que esté bien. Los **conflictos semánticos** git no los marca: si
la base cambió la firma de una función que la rama llama, el merge sale limpio y el build revienta.
`gh stack` tampoco verifica nada de esto: mueve commits, no juzga el resultado.

Por cada rama que se movió:

```
git range-diff <base-vieja>..<SHA-original-rama> <base-nueva>..<rama>
git diff <SHA-original-rama> <rama>
```

- `range-diff` muestra commit por commit qué cambió respecto al original. Si un hunk propio desapareció en la resolución, aparece ahí. Revisarlo entero, no en diagonal.
- El `diff` contra el SHA original debería contener **solo** lo que aportó la base. Si aparece una reversión de código propio, es un bug de la resolución.

Después, las verificaciones del proyecto (typecheck, lint, build, tests). Usar los comandos reales del
repo, no adivinados. En un stack, correrlas al menos en la rama de arriba; si una capa baja tocó algo
crítico, también ahí. Si algo falla, **arreglarlo aquí**: no se reporta éxito con el build roto, y no
se baja ningún threshold de coverage para que pase.

---

## Modo land — aterrizar el stack

Solo cuando el usuario lo pide. Reemplaza el loop de mergear una, rebasear la siguiente, pushear,
mergear la siguiente: `gh stack merge` mergea el PR elegido y **todos los no mergeados debajo** en una
operación atómica, y con `--squash` deja **un commit squasheado por PR** en la base, de abajo hacia
arriba. Mergear n PRs deja n commits.

**Si el usuario ya mergeó desde la web**, el merge box hace exactamente lo mismo con un click: los
pre-checks ya los evaluó GitHub y el merge ya corrió. Saltar directo a **Después del merge**, que es
la única parte que la web no hace. No re-evaluar pre-checks de algo que ya pasó.

### Pre-checks (todos, antes de proponer nada)

```
gh stack view --json                                          # needsRebase debe ser false en todas
gh pr view <n> --json isDraft,reviewDecision,statusCheckRollup,mergeStateStatus
```

- Alguna rama con `needsRebase: true` → el stack no tiene historia lineal y **no puede mergear**.
  Correr primero el modo sync de esta skill (o el botón **Rebase stack** del merge box).
- Los requisitos (reviews, checks, CODEOWNERS, branch protection) se evalúan contra la **base del
  stack**, no contra la rama de abajo. Todos los PRs por debajo del elegido tienen que estar
  aprobados y verdes, no solo el de arriba.
- Se mergea **de abajo hacia arriba**. No se puede mergear un PR del medio en aislamiento: siempre
  arrastra los de abajo. Para aterrizar solo una parte, elegir el PR más alto que se quiera incluir.
- Bypassear requisitos no está soportado para stacks. Auto-merge tampoco.

### Comando propuesto (no ejecutar sin OK explícito)

```
gh stack merge <pr-top> --yes --squash     # ese PR y todos los no mergeados debajo
gh stack merge <stack-number> --yes --squash
```

- All-or-nothing: si un PR del set no puede mergear, no mergea ninguno, y dice por qué.
- Sin flag de método reusa el último usado: pasar `--squash` siempre y explícitamente.
- `gh pr merge` **no** puede mergear un stack.
- Si la base tiene merge queue, el stack se encola y **la queue elige el método**: `--squash` se
  ignora con un warning, y los PRs pueden aterrizar en grupos separados. Avisarlo antes.

### Después del merge

```
gh stack sync --prune      # pushea y borra ramas locales mergeadas: pedir OK antes
```

Si se aterrizó solo una parte del stack, los PRs de arriba quedan abiertos y GitHub los re-targetea y
rebasea solo contra la base del stack. Local no se entera hasta el `sync`.

Una vez que todos los PRs del stack mergearon, ese stack está completo y no se extiende: un
`gh stack submit` de ramas nuevas arranca un stack nuevo sobre trunk.

---

## Reporte

En español. Estructura:

**1. Operación** — ruta (A/B), qué se corrió (`gh stack rebase`, `rebase --onto`, `reset`, `merge`), contra qué SHA de la base, qué ramas se movieron.

**2. Conflictos resueltos** — una tabla:

| Archivo | Rama/capa | Qué quería la base | Qué quería mi rama | Resolución | Por qué |
|---|---|---|---|---|---|

**3. Revisá esto vos** — lista aparte con lo de confianza baja, lo que se decidió con un supuesto, y lo que git mergeó limpio pero conviene mirar (mismo archivo tocado por ambos lados sin conflicto textual).

**4. Verificaciones** — qué se corrió y con qué resultado. Si algo no se corrió, decirlo.

**5. Estado del stack** — Ruta A: salida de `gh stack view --json` traducida a tabla (capa, PR, estado, `needsRebase`). Ruta B: las ramas hijas que quedan pendientes de rebasear, y que el force-push de esta rama las deja apuntando a commits viejos. Si B0 encontró dependientes y el repo soporta stacks, acá va la propuesta de adopción, en dos líneas: qué comando la convierte en stack y qué se gana (la próxima vez es un `gh stack rebase`, y el aterrizaje es un `gh stack merge --squash` en vez del loop de mergear-rebasear-pushear).

**6. Siguiente paso, sin ejecutar** — el push o el merge propuesto, y el rollback:

```
gh stack push                            # Ruta A
git push --force-with-lease origin <rama>  # Ruta B
gh stack merge <pr-top> --yes --squash   # modo land

gh stack rebase --abort                  # rollback si el rebase está en curso
git reset --hard <SHA-original-rama>     # rollback por rama si ya terminó
git branch -D backup/<rama>-<...>        # limpiar el respaldo cuando esté OK
```
