# Fuentes de contexto del repo

Archivo: `~/.claude/profiles/<repo>.review.md`. Vive junto al perfil de `/flow`, pero
aparte: `/flow` toma la existencia de su perfil como señal de que ya está derivado, y un
perfil a medio llenar lo engañaría.

Es **acumulativo**: cada PR revisada puede agregar una fuente, una descartada o una
preferencia. Nunca se re-deriva desde cero.

## Formato

```markdown
# fuentes de revisión: <repo>
actualizado: <YYYY-MM-DD>

## Fuentes
### <nombre>
tipo:      repo | submódulo | cloud | conector | web
cómo:      <comandos o pasos exactos para consultarla desde una PR>
aporta:    <qué tipo de contexto da: objetivo de producto, decisiones, specs, runbooks>
cuándo:    siempre | si la PR toca <rutas o zonas>
permiso:   local | read-only permitido (<perfil/cuenta>) | preguntar cada vez

## Descartadas
- <fuente> — <por qué> (<YYYY-MM-DD>)

## Preferencias del brief
- <YYYY-MM-DD> — <qué sobró o faltó, en una línea>
```

`Descartadas` evita volver a proponer lo mismo en cada PR.

## Búsqueda inicial (el archivo no existe)

Una pasada barata, sin leer código de la aplicación:

- `.gitmodules` → submódulos de documentación (wiki, specs, docs compartidas)
- directorios de conocimiento: `docs/`, `wiki/`, `adr/`, `openspec/`, `rfcs/`, `specs/`
- `CLAUDE.md`, `AGENTS.md`, `README.md` → a dónde mandan a leer
- IaC: `cloud/`, `infra/`, `cdk/`, `terraform/`, `*.tf`, `serverless.yml`, `template.yaml`
- los bodies de las últimas ~10 PRs mergeadas (`gh pr list --state merged --limit 10 --json body`):
  dominios enlazados (Linear, Jira, Notion, Figma, Slack) y rutas citadas
- la PR actual: lo mismo sobre su body

Proponé las candidatas con `AskUserQuestion` (multiSelect), cada una con qué aportaría en
una línea. Lo elegido va a `Fuentes`; lo no elegido, a `Descartadas`. Para una fuente
cloud o un conector, preguntá también el permiso (perfil o cuenta, y si es read-only).

## Agregar una fuente (el archivo existe)

Durante la investigación anotá cada fuente candidata que no esté ni en `Fuentes` ni en
`Descartadas`: un link a un dominio nuevo en el body, un directorio de docs que la PR toca
por primera vez, un servicio cloud que el IaC no explica. Al terminar el brief, proponelas
**en una sola pregunta** y actualizá el archivo con lo que responda el usuario.

Si una fuente registrada deja de funcionar (la ruta no existe, el comando falla),
corregila en el momento y decilo en una línea. Una fuente que miente es peor que no
tenerla.

## Fuentes cloud

El IaC del repo suele alcanzar para saber qué recursos se agregan. Consultá la cuenta sólo
cuando el código no alcanza: el recurso ya existe y hay que saber su estado, o hay que
saber qué depende de algo que se borra o se renombra.

- Sólo comandos de lectura (`describe-*`, `get-*`, `list-*`).
- Sólo con el perfil registrado como permitido. Producción, nunca sin OK explícito.
- Lo que se consultó y lo que devolvió va al dossier.
