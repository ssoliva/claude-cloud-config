# Perfil del repo (fase 2)

Se deriva **una vez por repo** y se guarda en `~/.claude/profiles/<slug>.md`, donde
`<slug>` es el nombre del directorio raíz del repo. Versionado: viaja a todas tus
máquinas.

Su razón de ser: la pregunta "¿cómo pruebo esto?" no cambia entre features. Pagarla
una vez por feature es el desperdicio más grande del flujo.

## Cómo derivarlo

En una sola pasada, barata. No leas el código de la aplicación:

- `package.json` / `Makefile` / `pyproject.toml` / `Cargo.toml` → scripts reales
- `.github/workflows/*.yml` → **qué corre CI**, que es la definición operativa de
  "verde". Si CI corre algo, la validación local debería correr eso mismo.
- `CLAUDE.md`, `AGENTS.md`, `README.md`, `CONTRIBUTING.md` → convenciones y gates
- `docker-compose.yml`, `.env.example`, `.envrc` → cómo se levanta local
- `git symbolic-ref refs/remotes/origin/HEAD` o `gh repo view --json defaultBranchRef`
  → rama base real, no asumida
- **la plantilla de PR**, con `find` case-insensitive sobre las cuatro ubicaciones
  que GitHub acepta. `pull_request_template.md` en minúsculas es tan válido como en
  mayúsculas y es la variante que más se escapa:

  ```bash
  find . -maxdepth 3 \( -ipath "./.github/pull_request_template.md" -o -ipath "./.github/pull_request_template/*.md" -o -ipath "./docs/pull_request_template.md" -o -ipath "./pull_request_template.md" \) 2>/dev/null
  ```

  Guardá la **ruta**, no el contenido: la plantilla cambia y la fase 8 la relee.

Lo que **no** puedas derivar con confianza, preguntalo. Es la única vez que se
pregunta, así que vale la ronda.

## Plantilla

```markdown
# perfil: <nombre-repo>
derivado: <YYYY-MM-DD> · confirmado por el usuario: sí

## Identidad
ruta: <ruta absoluta>
rama base: <dev | main | master>
convención de ramas: <feat/xxx, fix/xxx>

## Validación local
install:   <comando | n/a>
tests:     <comando>            # el que corre CI, no uno inventado
build:     <comando | n/a>
typecheck: <comando | n/a>
lint:      <comando | n/a>
alcance:   <corre todo | scoped: cómo se deriva el alcance>

## Validación funcional
levantar local: <comando + puerto | n/a>
qué mirar:      <ruta, pantalla, endpoint>
datos de prueba: <seed, fixture, cuenta | n/a>

## Staging
existe: <sí | no>
cómo se despliega: <comando exacto>
PROPONER Y PARAR — nunca ejecutar. Tener el comando acá no es autorización.

## Requiere permiso explícito
<lista: deploys, invocaciones cloud, escrituras en bases compartidas, perfiles AWS>

## Convenciones de entrega
commits: <conventional commits, idioma del subject>
PR:      <rama destino, idioma de la descripción>
plantilla PR: <ruta | ninguna>
convención escrita de PR: <archivo + qué exige | ninguna>
esqueleto PR: <ruta a un override para este repo | ~/.claude/rules/pr-body.md>
# precedencia: plantilla del repo > esqueleto del perfil > rules/pr-body.md
merge:   <squash | merge>

## Gotchas
<lo que sorprende: gates que fallan por razones ajenas, deuda preexistente,
 pasos que hay que correr antes de los tests>
```

## Mantenimiento

Si durante una feature descubrís que el perfil está desactualizado — un comando que
ya no existe, un gate nuevo —, **corregilo en el momento** y decílo en una línea. Un
perfil que miente es peor que no tenerlo.
