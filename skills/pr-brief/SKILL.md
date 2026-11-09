---
name: pr-brief
description: Prepara la revisión de una PR ajena — reconstruye cómo funcionaba lo que toca, qué cambia en comportamiento y qué decisiones hay que juzgar — y lo deja en un brief de 2 minutos en .context/reviews/ del repo, más un dossier para contestar preguntas durante la revisión. Usar cuando el usuario invoque /pr-brief <número|url>.
disable-model-invocation: true
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - Agent
  - AskUserQuestion
---

# pr-brief

El usuario va a revisar una PR de su equipo. Lo que le cuesta no es encontrar bugs: es
entender **qué hacía esto antes, para qué existe, qué cambia en comportamiento y qué se
decidió**. Tu trabajo es reconstruir eso y entregarlo en un documento que se lea en
**2 minutos**, y quedarte con el contexto completo para contestar preguntas después.

No es una revisión de bugs: para eso está `/code-review`. Si ves un bug evidente, entra
como un ítem de "Para juzgar", pero no salgas a buscarlos.

## Dos artefactos, dos lectores

| Artefacto | Ruta | Lector | Tamaño |
|---|---|---|---|
| **brief** | `.context/reviews/pr-<n>/brief.md` | el usuario | ≤2 min. Presupuesto duro |
| **dossier** | `.context/reviews/pr-<n>/dossier.md` | vos, al contestar | sin límite, todo con fuente |

La separación es lo que permite que el brief sea chico sin perder nada: **todo lo que
investigues va al dossier; al brief sólo sube lo que pasa los filtros**. Lo que no sube no
se pierde, se queda disponible para las preguntas.

`.context/` vive en la raíz del repo o worktree actual (Conductor lo muestra en el
workspace). Antes de escribir, comprobá `git check-ignore -q .context/x`; si no está
ignorado, agregá `.context/` a `.git/info/exclude` (local, nunca al `.gitignore`).

## Fuentes de contexto

Hay fuentes **comunes**, que se consultan siempre y no se guardan, y fuentes **del
repo**, que se acumulan en `~/.claude/profiles/<repo>.review.md`. `<repo>` es
`basename -s .git "$(git remote get-url origin)"` — no el nombre del directorio, que en
Conductor es el de la ciudad del workspace.

**Comunes:** el body, los comentarios y las reviews de la PR; los issues que enlaza; sus
commits; el diff; el código tocado **en la base**; el `git log` de esas zonas; las PRs
anteriores que tocaron los mismos archivos.

**Del repo:** cargá `references/fuentes.md` para el formato del archivo, la búsqueda
inicial cuando no existe y cómo se agrega una fuente nueva. Resumen:

- **No existe** → búsqueda inicial barata, proponé las candidatas con `AskUserQuestion`
  (multiSelect), guardá las elegidas y las descartadas.
- **Existe** → usala sin preguntar. Si durante la investigación aparece una fuente que no
  está ni en `Fuentes` ni en `Descartadas`, proponela **al final**, en una sola pregunta.

Ninguna consulta sale de la máquina sin estar registrada como permitida en el archivo
(AWS, conectores). Una fuente cloud nunca se consulta contra producción sin OK explícito.

## Procedimiento

### 1. Resolver la PR

```bash
gh pr view <n> --json number,title,author,body,baseRefName,headRefName,headRefOid,url,files,closingIssuesReferences,comments,reviews
git fetch origin "pull/<n>/head" "<base>"
HEAD_SHA=$(git rev-parse FETCH_HEAD)  # o el headRefOid
BASE_SHA=$(git merge-base "origin/<base>" "$HEAD_SHA")
```

No hagas checkout: el usuario puede estar en otra rama. Leé con `git show <sha>:<ruta>` y
`git diff "$BASE_SHA" "$HEAD_SHA"`.

Si `.context/reviews/pr-<n>/brief.md` existe y su header tiene el mismo `head`, no
regeneres: cargá el dossier y pasá al paso 6. Si el head cambió, regenerá y decí en una
línea qué cambió desde la versión anterior.

### 2. Clasificar

Antes de leer a fondo, con `--stat` y los nombres de archivo:

- **Zona:** qué parte del sistema es (servicio, módulo, flujo de negocio), en una línea.
- **Tipo:** `producto` (cambia comportamiento observable) · `refactor` · `infra` · `mixto`.
- **Mecánico vs. sustantivo:** separá los archivos que son wiring, renombres, tests
  espejo, lockfiles y bumps. Esos se resumen en una línea y no se estudian.

El tipo decide dónde gastar: en `producto`, el antes y el después; en `infra`, los
recursos y el orden de despliegue; en `refactor`, sólo verificar que no cambió el
comportamiento.

### 3. Reconstruir el antes

Sólo sobre los archivos sustantivos:

- el código en `BASE_SHA`: qué hace, quién lo llama, qué contrato expone
- el **para qué**: las fuentes del repo (wiki, specs) y el `git log` / las PRs que lo
  crearon. El objetivo original suele estar en la PR que introdujo el código, no en el código
- si toca infraestructura: qué recursos existían (IaC en la base)

Todo va al dossier con su fuente (`ruta:línea@base`, página de wiki, `#PR`).

### 4. Calcular el delta

- **Comportamiento:** casos concretos (entrada o situación → resultado) que cambian. Para
  cada uno, si está **declarado** (body, wiki, spec, issue) o no.
- **Decisiones:** dónde se eligió entre alternativas. Califica si cumple alguna de: cambia
  un contrato de borde (API, evento, columna, config, secreto); revertirla cuesta más que
  rehacer la PR; había una alternativa razonable que un revisor podría preferir.
- **Infra:** recursos nuevos, cambiados o borrados, permisos que se amplían, orden de
  despliegue, migraciones y su ventana de incompatibilidad.
- **Silencios:** lo que la PR cambia y no menciona. Es lo más valioso del brief.

Para PRs grandes (>25 archivos sustantivos) podés lanzar subagentes `Explore` en paralelo,
uno por zona, para el antes. Lo que devuelvan va al dossier; el razonamiento del delta lo
hacés vos, porque sos quien va a contestar las preguntas.

### 5. Escribir el dossier y después el brief

Cargá `references/brief.md`: plantilla, gatillos, presupuesto y la pasada de editor.
Primero el dossier completo y después el brief, como destilación. **La pasada de editor
no es opcional.**

### 6. Entregar y quedarse

En el chat, tres líneas: la ruta del brief, el veredicto de una frase (qué es la PR) y
cuántos ítems hay "para juzgar". Nada más: el contenido está en el archivo.

Después, el modo pregunta:

- Contestá desde el dossier, citando la fuente. Si no está, investigalo, **agregalo al
  dossier** y contestá.
- Si una pregunta revela algo que debería haber estado en el brief, decilo y corregí el
  brief.
- Si aparecieron fuentes candidatas nuevas (paso de fuentes), proponelas ahora.

### 7. Retroalimentación

Cuando el usuario diga qué sobró o qué faltó del brief, registralo en
`## Preferencias del brief` del archivo de fuentes del repo, en una línea, con la fecha.
Las preferencias se aplican en la pasada de editor de las próximas PRs. No preguntes por
retroalimentación si el usuario no la da: invitalo una sola vez, en la línea final de la
entrega.
