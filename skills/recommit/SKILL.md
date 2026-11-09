---
name: recommit
version: 1.0.0
description: Reescribe la historia de una rama para que los commits reflejen solo el resultado final, reconstruyéndolos desde el diff total en vez de reordenar la historia real, verificando que cada commit del resultado compile. Usar cuando una rama pasó por varias opciones contradictorias antes de aterrizar, cuando el usuario pida limpiar, reagrupar o "arreglar" los commits de una PR antes de pedir revisión, o invoque /recommit.
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - AskUserQuestion
---

# recommit

Una rama que iteró (se derivó por un stack, se descartó, se rehízo por otro) llega a la PR con commits
que se contradicen entre sí. Esta skill reconstruye la historia **desde el árbol final**, para que la
lista de commits describa el resultado y no el camino recorrido.

Hermana de `/commits`: mismo criterio de atomicidad y misma tabla, pero `/commits` opera sobre trabajo
sin commitear y esta opera sobre historia ya commiteada.

## Dos problemas distintos

1. **Historia confusa** — la lista de commits cuenta la exploración. Es lo que esta skill arregla.
2. **Residuos de la exploración en el diff final** — código muerto de la opción descartada: un helper
   exportado que nadie usa, un flag de config del approach viejo, tests del stack abandonado, una dep
   sin uso.

Reagrupar commits **no** arregla el problema 2: mueve las mismas líneas de sitio. El diff total de una
rama nunca contiene los cambios que se revirtieron (un archivo tocado y devuelto a su estado original no
aparece), así que si las líneas a revisar *crecieron*, es residuo real. Se audita en la Fase 2 y se
limpia **fuera** de la reescritura (§Fase 2), nunca durante.

## Contrato

Aplica durante toda la ejecución. Si choca con la instrucción de avanzar rápido, gana la regla.

1. **El árbol final es la fuente de verdad, y el working tree no se toca.** La reescritura solo mueve
   `HEAD` y el índice. Ningún paso edita, borra ni restaura archivos del directorio de trabajo. Esa es
   la propiedad que hace la operación segura: si algo falla, el contenido final sigue en disco intacto.
2. **Verificación obligatoria.** Al terminar, el árbol del nuevo tip tiene que ser **byte-idéntico** al
   del tip original. Si `git diff --exit-code <sha-original> HEAD` no sale vacío, la reescritura está
   mal: se revierte con `git reset --hard <sha-original>` y se reporta. No se "arregla a mano".
3. **Gate humano antes de reescribir.** La tabla de la Fase 3 se entrega y se espera OK explícito. Sin
   OK no se corre ningún comando que mueva `HEAD` o el índice.
4. **Cero push y cero merge sin OK explícito.** Nunca `push --force`; solo se **propone**
   `push --force-with-lease`. Si el lease es rechazado, se para: alguien más pusheó.
5. **Historia ajena, intocable.** Si hay commits de otro autor en el rango, se aborta.
6. **Cada commit tiene que compilar, o se dice que no se verificó.** El plan se ordena para que ningún
   commit dependa de uno posterior (§3b) y después se verifica commit a commit (§Fase 5). Si el repo no
   tiene un comando verificable, o el usuario declina el costo, el reporte final lo dice con esas
   palabras: *la garantía cubre solo el tip*. Nunca se afirma una verificación que no corrió.

---

## Fase 0 — Preflight

Determinar la base real. **No** asumir `origin/dev`: si la rama es una capa de un stack, la base es el
tip de su rama padre, y equivocarse acá re-commitea las capas de abajo.

```bash
BR=$(git branch --show-current)
gh stack view --json >/dev/null 2>&1; echo "stack: $?"
```

- `stack: 0` → hay stack rastreado. Leer `gh stack view --json` y tomar el campo `base` de la entrada
  cuyo `name` es `$BR`. Anotar también si hay capas **arriba** (las que tienen a `$BR` como `base`) y el
  `isQueued` / `isMerged` de la rama.
- cualquier otro código → rama suelta: `BASE=$(git merge-base HEAD origin/<base>)`, con `<base>` la rama
  base de la PR (`gh pr view --json baseRefName`) o la default del repo.

Después, **asegurar que la base es ancestro de HEAD**. Si el padre fue mergeado con squash su tip ya no
lo es, y el `reset` posterior produciría basura:

```bash
git merge-base --is-ancestor "$BASE" HEAD; echo "ancestor: $?"
```

`ancestor: 1` → parar. La rama necesita `/rebase-stack` primero; recién después tiene sentido reescribir.

Condiciones de salida, todas antes de tocar nada:

| Condición | Chequeo | Qué hacer |
|---|---|---|
| Working tree sucio | `git status --porcelain -uall` | **ABORTAR.** El `reset` mezclaría el trabajo suelto con la historia reescrita y lo commitearía en silencio. Commitear o stashear primero |
| Commits de otro autor | `git log --format='%ae' "$BASE"..HEAD \| sort -u` | **ABORTAR** salvo OK explícito del usuario |
| Diff neto vacío | `git diff --exit-code "$BASE" HEAD` | **PARAR.** La exploración se anuló a sí misma: no hay nada que reagrupar, la PR debería cerrarse |
| Un solo commit en el rango | `git rev-list --count "$BASE"..HEAD` | **PARAR** salvo que el usuario quiera partirlo en varios |
| Rama mergeada | `isMerged` en `gh stack view --json` | **ABORTAR.** Reescribir historia ya mergeada no tiene sentido |
| PR en merge queue | `isQueued`, o `gh pr view --json state,mergeStateStatus` | **ABORTAR.** Un force-push la saca de la cola |
| PR ya con comentarios de review | `gh api repos/{owner}/{repo}/pulls/<n>/comments --jq 'length'` (los comentarios **inline**; `gh pr view --json comments` devuelve los del hilo general, que no se desanclan) | **ADVERTIR fuerte y preguntar.** El force-push desancla los comentarios inline: quedan como *outdated* y el revisor pierde el hilo. Esta skill se corre **antes** de pedir revisión |
| Merge commits en el rango | `git log --merges "$BASE"..HEAD` | **ADVERTIR.** La topología del merge desaparece y el historial queda lineal. Suele ser deseable, pero decirlo |
| Rename que solo cambia la caja | `git diff -M --name-status "$BASE" HEAD` + `git config --get core.ignorecase` | **ABORTAR** en macOS/Windows (`core.ignorecase=true`). Tras el `reset` git ve el árbol como limpio y el rename se pierde: el commit sale vacío y el árbol final no se puede reproducir. Se rehace ese rename a mano (`git mv --force`) en un commit aparte |
| Rutas del diff cubiertas por `.gitignore` | `git diff --name-only "$BASE" HEAD \| git check-ignore --stdin` | Seguir, pero anotarlas: en la Fase 4 necesitan `git add -f`. Sin `-f` el `add` falla y esas rutas **no aparecen en `git status`** |
| Sparse checkout activo | `git config --get core.sparseCheckout` | **ABORTAR** si el diff toca rutas fuera del cono: no están en disco y el `add` registraría un borrado falso |
| Capas arriba en el stack | `gh stack view --json` | Seguir; la Fase 5 obliga a `gh stack rebase --upstack` |

Con todo en verde, respaldo y SHA original:

```bash
OLD=$(git rev-parse HEAD)
git branch "backup/recommit/$BR-$(date +%Y%m%d-%H%M)" "$OLD"
echo "OLD=$OLD"
```

El respaldo es una rama, no solo el reflog: sobrevive a un `gc` y se puede comparar por nombre.

## Fase 1 — Rescatar la narrativa antes de destruirla

Los mensajes de los commits actuales son el único registro de qué se probó y por qué se descartó. Eso es
justo lo que vale la pena conservar en la descripción de la PR, y después de la Fase 4 ya no existe.

```bash
git log --reverse --date=short --format='%h %ad %an%n%s%n%b%n---' "$BASE"..HEAD \
  > "$(git rev-parse --git-dir)/recommit-narrative-$BR.md"
```

Se escribe bajo `.git/`, **nunca** en una ruta trackeada: un archivo nuevo en el árbol contaminaría el
diff y rompería la verificación de la regla 2.

Sobre ese log, redactar un resumen corto — 3 a 6 bullets — con las opciones que se exploraron y el
motivo del descarte, en el formato que usa la descripción de la PR. Entregarlo en la respuesta junto con
la tabla de la Fase 3, para que el usuario lo pegue en la PR o lo consuma `/ship`.

## Fase 2 — Auditoría de residuos

Sobre el diff total (`git diff -M --name-status "$BASE" HEAD`), buscar restos del approach descartado:

- **Símbolos nuevos sin referencias.** Para cada export/función/tipo agregado, `grep` de su nombre en el
  repo fuera del archivo que lo define.
- **Imports y variables muertas.** Si el repo tiene typecheck o lint rápido, correrlo y quedarse con los
  hallazgos que caen en archivos del diff.
- **Config, flags y feature toggles** agregados y referenciados solo desde su propia definición.
- **Dependencias** nuevas en el manifiesto sin ningún import que las use.
- **Tests** que ejercitan el camino abandonado (nombres que mencionan el approach viejo, o que testean
  un símbolo que ya no existe en el código de producción).

Se **reporta** como tabla, con archivo, qué es y por qué parece residuo. **No se borra nada acá.**

> **Por qué el borrado va aparte:** limpiar residuos cambia el árbol final, y eso invalida la
> verificación de la regla 2 — el único chequeo que garantiza que la reescritura no perdió trabajo.
> Si el usuario aprueba la limpieza, va **después** de la Fase 4 y verificada, como uno o más commits
> normales sobre la historia nueva (`refactor(scope): drop leftovers of <approach>`). Así queda visible
> y revisable en vez de escondida dentro de un reagrupado.

## Fase 3 — Propuesta

La tabla de `/commits`, aplicada a `git diff "$BASE" HEAD` en vez de al working tree. Mismo criterio de
agrupación (unidad lógica, no carpeta ni tipo de archivo), mismo orden por dependencias, mismas reglas
de idioma: subject en inglés e imperativo, columna "Qué hace" en español.

```
## Descomposición en commits
```

| # | Commit (conventional) | Archivos | Qué hace |
|---|---|---|---|

Además de las reglas de `/commits`, acá:

- **Todo archivo del diff total aparece en exactamente una fila**, salvo los que necesitan partirse
  (marcarlos, van con la mecánica de la Fase 4).
- **Los dos lados de un rename van en la misma fila.** `git diff -M` los detecta; separarlos produce un
  commit que borra un archivo que nada recrea, y ese punto del historial no compila.
- Debajo de la tabla, el resumen narrativo de la Fase 1 y la tabla de residuos de la Fase 2.

### 3b — Coherencia del plan (estática, antes del gate)

El orden de la tabla no es estético: define qué existe en cada punto del historial. En el paso `i` el
repo es *base + filas 1..i*; todo lo que aportan las filas posteriores **no existe todavía**. Una arista
que apunte hacia adelante es un commit que no compila.

Esto se chequea sin correr nada, y es lo que hace que la Fase 5 pase a la primera. El análisis se limita
a los archivos del diff: un archivo que la rama no tocó está en su versión base y no puede referenciar
nada nuevo.

| Arista | Cómo detectarla | Arreglo |
|---|---|---|
| **Import de un módulo que crea una fila posterior** | Por cada archivo de la fila `i`, listar sus imports en su contenido **final** y resolverlos; si el módulo lo crea la fila `j > i`, hay violación | Subir la fila `j`, o fusionar `i` y `j` |
| **Import de un símbolo que una fila posterior agrega a un módulo existente** | El módulo existe en base pero el símbolo no: `git show "$BASE":<módulo>` y buscar el export | Igual que arriba |
| **Borrado o rename que deja referencias colgando** | Si la fila `i` borra o renombra `M`, buscar quién referencia `M` (`grep` de la ruta de módulo). Todo referenciador tiene que estar en una fila `≤ i` | Mover los referenciadores a la fila `i`, o bajar el borrado. Si un referenciador **no está en el diff**, no es un problema de orden: es un bug del árbol final — reportarlo en la Fase 2 |
| **Dependencia nueva usada antes de declararse** | Un import de un paquete que el manifiesto agrega en la fila `j > i` | El cambio de manifiesto + lockfile va en la primera fila que usa la dep |
| **Migración después del código que la usa** | Heurística, no automática: si una fila trae schema/migración y otra el código que consulta los campos nuevos, la migración va primero | Reordenar |
| **Test antes del código que prueba** | Un test de la fila `i` que ejercita algo que aporta la fila `j > i` | Van en la misma fila (regla de `/commits`: el test que documenta el fix va con el fix) |

Las dos primeras aristas, que son las que aparecen casi siempre, se detectan con `git` y `grep`:

```bash
# 1. imports relativos de los archivos de una fila, leídos de su contenido final
grep -ohE "(from|require\()[[:space:]]*['\"](\./|\.\./)[^'\"]+" <archivos de la fila> \
  | grep -oE "(\./|\.\./)[^'\"]+"

# 2. ¿el módulo ya existe en la base, o lo crea una fila posterior?
git cat-file -e "$BASE:<ruta resuelta>" 2>/dev/null && echo "existe en base" || echo "lo crea la rama"

# 3. si lo crea la rama, ¿en qué fila está?
git diff --diff-filter=A --name-only "$BASE" HEAD
```

El grep es un **filtro rápido, no una prueba**. No resuelve imports sin extensión (`./helper` puede ser
`.ts`, `.tsx`, `index.ts`), ni alias de `tsconfig.json` (`@/lib/x` — hay que leer `compilerOptions.paths`),
ni barrels que re-exportan, ni imports dinámicos, ni dependencias que no son imports (inyección por
string, rutas en config, nombres de columna en SQL). Esas se le escapan; para eso está la Fase 5. Las dos
capas son complementarias: la estática hace que la Fase 5 pase a la primera, la Fase 5 es la que prueba.

Iterar hasta que no queden aristas hacia adelante. Si eliminarlas colapsa dos filas en una, se colapsan:
un commit que no compila es peor que un commit grande. Si el plan **no se puede** ordenar sin fusionar
todo en un commit, decirlo — el diff es genuinamente un solo cambio atómico.

Debajo de la tabla, listar en una línea qué aristas se encontraron y cómo cambió el orden. Si no había
ninguna, decirlo también: es información para quien aprueba.

**Gate.** Esperar OK explícito. Sin OK, termina acá.

## Fase 4 — Ejecución

```bash
git reset --mixed "$BASE"
```

`--mixed`, no `--soft`. Con `--soft` el índice queda con el árbol final completo y el primer `commit`
se lleva todo. Con `--mixed` el índice vuelve a la base y el **working tree queda intacto**: exactamente
el estado que permite re-armar los commits uno por uno.

Por cada fila de la tabla, en orden:

```bash
git add -A -- <lista explícita de archivos>
git commit -m "<subject>" -m "<body opcional>"
```

- `-A` con **lista explícita de archivos**, nunca un directorio: un directorio arrastra archivos que
  pertenecen a otra fila. `-A` es necesario para que las **eliminaciones** se registren (tras el
  `--mixed`, un archivo que la rama borró vuelve al índice desde la base y hay que volver a borrarlo).
- **Rutas cubiertas por `.gitignore`** (anotadas en la Fase 0): `git add -f -- <archivo>`. Sin `-f`
  el comando falla con *paths are ignored*; verificar el exit code, porque esas rutas tampoco se ven
  en `git status`.
- **Un archivo que se reparte entre dos commits:** no usar `git add -p` (es interactivo). Generar el
  patch del archivo, quedarse con los hunks de esa fila y aplicarlo **solo al índice**:

  ```bash
  git diff -- <archivo> > "$(git rev-parse --git-dir)/recommit-hunk.patch"
  # editar el patch para dejar únicamente los hunks de este commit
  git apply --cached "$(git rev-parse --git-dir)/recommit-hunk.patch"
  ```

  `--cached` no toca el disco: el working tree sigue en su estado final y los hunks restantes quedan
  disponibles para la fila siguiente.

  Si el archivo es **nuevo** (lo agrega la rama), `git diff` no devuelve nada — es untracked. Antes de
  generar el patch hay que declararlo: `git add -N -- <archivo>`.
- Si un `commit` falla con *nothing to commit*, la lista de archivos de esa fila está mal. Corregir la
  fila, no pasar `--allow-empty`.

**Verificación (regla 2), obligatoria:**

```bash
git diff --exit-code "$OLD" HEAD; echo "tree: $?"
git status --porcelain -uall
```

`git diff "$OLD" HEAD` con **dos** puntos: compara los dos árboles. Con tres puntos compararía contra el
merge-base y mostraría el diff completo, ocultando cualquier diferencia real.

- `tree: 0` y `status` vacío → la reescritura es correcta.
- **El chequeo que manda es el del árbol.** `git status` puede salir vacío con un archivo faltante: las
  rutas ignoradas nunca se listan, y en un FS case-insensitive un rename de solo caja se ve como limpio.
  Nunca dar por buena la reescritura solo porque `status` está vacío.
- Cualquier otra cosa → `git reset --hard "$OLD"` y reportar qué quedó fuera. No improvisar parches.

Si un hook de pre-commit reformatea archivos, la verificación lo va a marcar. No es motivo para
`--no-verify`: el árbol original ya pasó por ese hook, así que un reformateo indica que el hook no es
idempotente y eso hay que mirarlo aparte.

## Fase 5 — Verificación por commit

La Fase 4 garantiza el árbol del tip. Esto garantiza los pasos intermedios. Corre **siempre**, salvo que
el usuario lo declina explícitamente o el repo no tenga un comando verificable (regla 6).

### 5a. Comando de verificación

Cachear por repo en `.git/recommit-verify-cmd` (ruta no trackeada, no contamina el diff). Si no existe,
detectar y **confirmar con el usuario** antes de gastar N builds:

```bash
CACHE="$(git rev-parse --git-dir)/recommit-verify-cmd"
[ -f "$CACHE" ] && cat "$CACHE"
```

Orden de detección: script `typecheck` / `type-check` / `check` del manifiesto → `build` → `tsc --noEmit`
si hay `tsconfig.json` y `typescript` en las deps → nada. El gestor sale del lockfile (`yarn.lock` →
`yarn`, `package-lock.json` → `npm`, `bun.lock` → `bun`, `pnpm-lock.yaml` → `pnpm`).

Preferir typecheck sobre build: es más rápido y atrapa exactamente la clase de error que produce un mal
orden de commits (símbolo o módulo inexistente). El build completo y los tests van solo si el usuario
pide modo estricto.

Si no hay nada detectable, **no inventar un comando**: preguntar cuál es, o seguir sin verificar y
decirlo en el reporte.

### 5b. Loop de verificación en un worktree temporal

Nunca en el working tree principal. `git rebase --exec` sirve para esto y preserva los SHAs, pero al
fallar deja el repo en *detached HEAD* con el árbol intermedio en disco — exactamente el estado confuso
que esta skill promete no crear. El worktree aislado mantiene el invariante de la regla 1.

```bash
WT=$(mktemp -d)/recommit-verify
git worktree add --detach -q "$WT" HEAD
# node_modules compartido: reinstalar por commit cuesta minutos
[ -d node_modules ] && ln -s "$(pwd)/node_modules" "$WT/node_modules"

for SHA in $(git rev-list --reverse "$BASE"..HEAD); do
  [ "$SHA" = "$(git rev-parse HEAD)" ] && continue   # el tip ya se verifica aparte
  git -C "$WT" checkout -q --detach "$SHA"
  ( cd "$WT" && eval "$VERIFY_CMD" ) || { echo "FALLA en $SHA"; break; }
done

git worktree remove --force "$WT"
```

- **El tip se saltea**: su árbol es idéntico al que el usuario ya probó. Ahorra un build y no pierde
  cobertura, siempre que el paso 6.1 corra la suite en el tip.
- El worktree se limpia siempre, incluso si el loop se corta (`git worktree remove --force`, y
  `git worktree prune` si quedó huérfano).
- Si el build necesita `.env` o artefactos que no están en git, copiarlos al worktree antes del loop. Si
  aun así no corre, caer a `git rebase --exec '<cmd>' "$BASE"` **con `git rebase --abort` obligatorio**
  ante cualquier falla, y re-verificar la regla 2 después.

### 5c. Cuando un commit falla

La falla es del **plan**, no del código: el árbol final compila (el usuario lo estuvo corriendo). Casi
siempre es una arista que la Fase 3b no vio.

1. Leer el error y nombrar el símbolo, módulo o dep que falta.
2. Ubicar en qué fila posterior aparece eso.
3. Proponer el arreglo mínimo — mover ese archivo a la fila que falla, o fusionar las dos filas — y
   mostrar la tabla corregida.
4. `git reset --hard "$OLD"` (vuelve al tip original, árbol idéntico, cero riesgo) y repetir la Fase 4
   con el plan corregido.

**Máximo dos reintentos.** Al tercero, parar y entregarle al usuario el commit que falla, el error y la
tabla; seguir adivinando quema builds sin converger. Dejar el respaldo intacto.

## Fase 6 — Cierre

1. **Tests en el tip.** Correr la suite del repo.
2. **Reporte.** `git range-diff "$BASE" "$OLD" "$BASE" HEAD` para mostrar la equivalencia commit a
   commit, más un antes/después del conteo de commits.
3. **Limpieza de residuos**, si el usuario la aprobó en la Fase 2: ahora, como commits normales.
4. **Proponer** (no ejecutar):
   - `git push --force-with-lease` para la rama.
   - Si hay capas arriba en el stack, `gh stack rebase --upstack` **antes** de pushear. El replay debería
     ir sin conflictos: el árbol del nuevo tip es idéntico al viejo, así que el contexto de los patches
     de arriba no cambió. Confirmar con `gh stack view --json` que ninguna rama quedó con
     `needsRebase: true`; el tracking local de `gh stack` guarda SHAs y puede quedar viejo.
   - Recordar que el respaldo `backup/recommit/...` queda local y se puede borrar cuando la PR aterrice.

## Casos borde verificados

Probados en repos de juguete; los tres primeros hacen fallar la reescritura si no se manejan.

| Caso | Qué pasa | Manejo |
|---|---|---|
| Rename de solo caja en FS case-insensitive | Tras el `reset --mixed`, `git status` reporta *working tree clean* y el `commit` sale vacío. El árbol final queda con el nombre viejo | Se detecta en la Fase 0 y se aborta. El rename se rehace con `git mv --force` en un commit propio |
| Ruta del diff cubierta por `.gitignore` (commiteada con `add -f` en su momento) | `git add -A -- <ruta>` falla con exit 1 y la ruta **no aparece en `git status`**: es invisible en los dos chequeos obvios | Fase 0 la marca con `git check-ignore`; Fase 4 usa `git add -f` |
| Partir un archivo **nuevo** entre dos commits | `git diff -- <archivo>` devuelve vacío (untracked), así que no hay patch que recortar | `git add -N -- <archivo>` antes de generar el patch |
| Hook de pre-commit que reformatea | Cambia el árbol y la verificación falla | Se aborta. No es motivo para `--no-verify`: indica un hook no idempotente |
| Trabajo sin commitear al empezar | El `reset --mixed` lo mezclaría con la historia reescrita y lo commitearía en silencio | Precondición dura de la Fase 0 |
| Fila con lista de archivos incorrecta | `git commit` falla con *nothing to commit* | Se corrige la fila; jamás `--allow-empty` |
| `git rebase --exec` para verificar | Preserva los SHAs y ve el árbol intermedio correcto, pero al fallar deja *detached HEAD* con el árbol intermedio en disco | Por eso la Fase 5 usa un worktree aislado. Como fallback, `git rebase --abort` obligatorio ante cualquier falla |
| Repo sin comando de build ni typecheck | No hay forma de verificar los pasos intermedios | Se sigue, y el reporte dice *la garantía cubre solo el tip* (regla 6). No se afirma lo que no se corrió |

En todos ellos, el diff de árboles de la Fase 4 es la red de seguridad: cualquier pérdida se ve como
`tree: 1` y se revierte con `git reset --hard "$OLD"` sin tocar el working tree.

## Límites

- **No pushea, no mergea, no cierra la PR.** Propone los comandos.
- **No borra residuos dentro de la reescritura** (§Fase 2).
- **No reemplaza `/review`.** Esta skill ordena la historia; la calidad del código es otra pasada.
- **No reescribe historia compartida a la ligera.** Si otra persona tiene la rama checkouteada, el
  force-push le rompe el árbol local. Con capas propias arriba lo resuelve `--upstack`; con gente
  encima, se avisa antes.
- **No aplica a ramas mergeadas ni en merge queue** (§Fase 0).
- **La verificación por commit cubre lo que cubre el comando.** Un typecheck no garantiza que los tests
  pasen en cada paso intermedio; para eso hace falta modo estricto y N corridas de la suite. El reporte
  dice qué comando corrió, para que la garantía sea legible en vez de implícita.
