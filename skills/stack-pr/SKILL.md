---
name: stack-pr
version: 1.0.0
description: Apilar PRs dependientes en un stack de GitHub con `gh stack link`, sin tracking local, para ramas que viven en worktrees separados (Conductor). Usar cuando el usuario vaya a crear un PR desde una rama que cuelga de otra rama en vez de dev/main, cuando pida apilar, encadenar o linkear PRs, cuando mencione stacked PRs o `gh stack`, o cuando una rama dependa de un PR todavía abierto.
allowed-tools:
  - Bash
  - Read
  - Grep
  - Glob
---

# stack-pr

Convierte una cadena de ramas dependientes en un **stack de PRs en GitHub**, sin escribir estado
local. Es el complemento de `rebase-stack`: esta skill arma el stack, `rebase-stack` lo sincroniza y
lo aterriza.

Argumento opcional: la rama padre (`/stack-pr ssoliva/capa-1`) o los PRs en orden de abajo hacia
arriba (`/stack-pr 5228 5231 5238`).

## Por qué `link` y no `init`

En Conductor cada workspace es un worktree distinto del mismo repo, y **una rama solo puede estar
checkouteada en un workspace a la vez**. El modelo local de `gh stack` (`init`, `add`, `rebase`)
necesita checkoutear cada capa para moverla: con las capas repartidas en workspaces, git lo rechaza.

`gh stack link` crea y actualiza el stack **solo por API**: no escribe `.git/gh-stack`, no checkoutea
nada, y corrige la base de los PRs que no matcheen la cadena. Es la ruta que la doc oficial de
`gh-stack` indica para worktrees y para herramientas externas.

Corolario: en un repo con workspaces de Conductor, **nunca** correr `gh stack init`, `gh stack add`
ni `gh stack rebase` sin que el usuario lo pida explícitamente. Dejan tracking local que miente en
cuanto otro workspace toca una capa.

## Contrato

1. **Nada remoto sin OK explícito.** `gh pr create`, `gh pr edit --base` y `gh stack link` escriben
   en GitHub: se proponen con los números concretos y se espera. El diagnóstico es read-only.
2. **El padre no se adivina en silencio.** Si la detección da más de un candidato, o ninguno claro,
   preguntar mostrando los candidatos. Un stack armado sobre el padre equivocado se ve bien en la
   lista y muestra un diff mentiroso en el review.
3. **Cadena lineal o nada.** Si una capa no contiene el tip de la de abajo, el stack no va a poder
   mergear. Eso se arregla con `rebase-stack` **antes** de linkear, no después.
4. **Una rama sola no es un stack.** Si no hay padre, decirlo en una línea y terminar. No proponer
   stacks donde no hay dependencia.

## Fase 1 — Identificar el padre

En este orden; el primero que da resultado gana:

1. **Argumento del usuario.** Siempre gana, incluso contra la heurística.
2. **Lo que ya se registró.** `git config --get branch.<rama>.stackParent`. Lo escribe la Fase 5 de
   esta misma skill. En Conductor el `.git/config` es compartido por todos los worktrees, así que un
   padre registrado en un workspace se lee desde cualquier otro.
3. **Heurística.** Ramas de PRs abiertos que son ancestros de HEAD:

   ```
   git fetch origin --quiet
   for ref in $(gh pr list --state open --limit 200 --json headRefName -q '.[].headRefName'); do
     [ "$ref" = "$(git branch --show-current)" ] && continue
     git merge-base --is-ancestor "origin/$ref" HEAD 2>/dev/null && echo "$ref"
   done
   ```

   El padre es el candidato **más cercano**: el que tiene a todos los demás candidatos como
   ancestros. Los otros son abuelos, y también son capas del stack.

4. **Nada de lo anterior.** Es una rama normal colgando de la base: decirlo y terminar (contrato 4).

Antes de seguir, ver si la rama ya está en un stack. El endpoint devuelve **todos** los stacks del
repo, así que hay que filtrar por la rama propia:

```
BR=$(git branch --show-current)
gh api repos/{owner}/{repo}/stacks 2>/dev/null \
  | jq -r --arg br "$BR" '.[] | select(.open) | select(any(.pull_requests[]; .head.ref == $br)) | "stack #\(.number) prs=\(.pull_requests | map(.number) | join(","))"'
```

- Ya está en un stack → no hay nada que hacer, salvo que falte una capa arriba: ahí es
  `gh stack link <stack-number> <pr-nuevo>`, que es aditivo.
- No está → seguir.

**Salida de la fase:** la cadena completa, de abajo hacia arriba, con rama y PR de cada capa, y de
dónde salió el padre (argumento, config o heurística). Recién entonces seguir.

## Fase 2 — Validar la cadena

Cada capa tiene que contener el tip de la de abajo:

```
git merge-base --is-ancestor origin/<capa-n> origin/<capa-n+1>   # debe dar 0 para cada par
```

Si alguno falla, la cadena perdió linealidad (típico: la base se movió y alguien rebaseó solo una
capa). **Parar acá** y mandar a `rebase-stack`; linkear primero solo esconde el problema hasta el
merge.

## Fase 3 — Asegurar los PRs

Cada capa necesita su PR abierto. Para las que falten:

```
gh pr view <rama> --json number,state,baseRefName 2>/dev/null   # ¿ya existe?
gh pr create --base <capa-de-abajo> --head <rama>               # remoto: pedir OK
```

La base con la que se crea no es crítica: `link` corrige después las bases que no matcheen la
cadena. Sí es crítica cuando el PR se va a revisar antes de linkear, porque contra `dev` el diff
incluye los commits del padre y el reviewer ve el doble de lo que corresponde.

## Fase 4 — Linkear

```
gh stack link <pr-abajo> <pr-medio> <pr-arriba>    # en orden, de abajo hacia arriba
gh stack link <stack-number> <pr-nuevo>            # agregar una capa a un stack existente
gh stack link --base dev <rama-abajo> <rama-arriba>  # también acepta ramas: las pushea y crea PRs
```

- Es aditivo: nunca saca PRs de un stack.
- Corrige las bases de los PRs existentes para que sigan la cadena.
- No deja tracking local, así que los comandos de navegación (`up`, `down`, `top`) no van a
  funcionar. Es el precio de que sea worktree-safe, y no importa: la navegación en Conductor es
  cambiar de workspace.

## Fase 5 — Registrar y reportar

Registrar el padre de cada capa para que la próxima corrida no vuelva a adivinar:

```
git config branch.<rama>.stackParent <rama-padre>
```

Va al `.git/config` compartido, no al del worktree: no usar `--worktree`.

Reporte, en español:

**1. Stack armado** — tabla de capas, de abajo hacia arriba:

| # | Rama | PR | Base antes | Base ahora |
|---|---|---|---|---|

**2. Cómo se identificó el padre** — argumento, `stackParent` registrado, o heurística (y con qué
candidatos).

**3. Qué se ejecutó y qué no** — PRs creados, bases corregidas, `link` corrido. Lo que quedó
propuesto sin ejecutar, con el comando exacto.

**4. Siguiente paso** — sin ejecutar:

```
gh stack merge <pr-top> --yes --squash    # aterrizar: un commit squash por PR (ver /rebase-stack land)
```

Y el recordatorio de que cuando la base se mueva, la sincronización es el botón **Rebase stack** del
merge box más un `git reset --hard origin/<rama>` en cada workspace, o `/rebase-stack` en cada uno.
