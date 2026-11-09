# La PR (fase 8, después de la Puerta 3)

**Una PR, una sola vez.** Ya la abriste: no la vuelvas a abrir, no abras una segunda
para el mismo trabajo.

Rama destino, idioma y convenciones salen del perfil del repo, no de tu default.

## Antes de abrirla

```bash
git status --short && git log --oneline <base>..HEAD
```

El árbol tiene que estar limpio y la lista de commits tiene que ser exactamente la
tabla aprobada. Si no coincide, pará y decílo — no abras la PR para "arreglarlo
después".

## Precedencia del cuerpo

```
1. Pedido explícito del usuario en ESTA corrida   ← gana sobre todo
2. Plantilla del repo, si existe                  ← su estructura es el cuerpo
3. Override en el perfil del repo                 ← si el perfil declara uno
4. ~/.claude/rules/pr-body.md                     ← el esqueleto por defecto del usuario
```

Nunca inventes un esqueleto propio. Si los cuatro fallan, preguntá.

**Qué cuenta como pedido explícito (capa 1).** Cualquiera de las dos, con el mismo
efecto: el flag `--esqueleto=propio` en la invocación, o que el usuario lo pida en
prosa en esta corrida ("usá mi esqueleto", "ignorá la plantilla del repo"). En ambos
casos saltás directo a `~/.claude/rules/pr-body.md` y la plantilla del repo no se lee.

**El pedido tiene que ser de esta corrida.** Que el usuario haya preferido su
esqueleto otra vez, o que lo tenga declarado en el perfil de otro repo, no es un
pedido: eso es la capa 3 y se resuelve sola. Ante la duda, respetá la plantilla del
repo y decí en una línea que la usaste.

## 1. Encontrar la plantilla del repo

`gh pr create --body "..."` **ignora la plantilla del repo por completo**. GitHub sólo
la precarga cuando abrís la PR por la web, o cuando `gh` abre el editor sin `--body`.
Como acá siempre pasás `--body`, hay que leerla a mano o no llega nunca.

El perfil guarda su ruta en `plantilla PR:`. Si la tiene, leela directo. Si el perfil
es viejo y no la tiene, buscala — **case-insensitive**, porque
`pull_request_template.md` en minúsculas es tan válido para GitHub como en mayúsculas
y es la variante que más se escapa:

```bash
find . -maxdepth 3 \( -ipath "./.github/pull_request_template.md" -o -ipath "./.github/pull_request_template/*.md" -o -ipath "./docs/pull_request_template.md" -o -ipath "./pull_request_template.md" \) 2>/dev/null
```

Si `.github/PULL_REQUEST_TEMPLATE/` devuelve varias, son plantillas alternativas por
tipo de cambio: elegí la que corresponda y decí cuál elegiste.

## 2. El esqueleto propio: `~/.claude/rules/pr-body.md`

Se usa cuando no hay plantilla del repo, o cuando el usuario lo pidió explícitamente
(capa 1) — en ese caso ni siquiera corras la búsqueda de la sección anterior.

Cargalo y seguilo. Ahí viven el esqueleto, sus gatillos por sección y las dos reglas
que lo gobiernan. **No lo parafrasees de memoria** — cambia, y el archivo manda.

Las dos que más se violan por inercia, así las tenés presentes al escribir:

- **Sección sin gatillo se borra entera.** Nunca `N/A`, nunca "Ninguno".
- **Los commits se referencian por subject, nunca por sha.** Un `/recommit` los
  invalida y el cuerpo queda mintiendo.

## 3. Llenar, venga de donde venga el esqueleto

- **Cada sección con contenido real.** Ninguna con el texto de ejemplo.
- **Los checkboxes se respetan**: marcá sólo lo que es verdad. Lo que no aplica queda
  sin marcar, con una línea que diga por qué. No marques nada para que se vea completo.
- **Los comentarios HTML** (`<!-- … -->`) con instrucciones se borran: son para el que
  escribe, no para el que lee.
- **No agregues secciones que el esqueleto no tiene.** La excepción es la de abajo.
- Para cambios visuales, las capturas o el video van **dentro de la sección que
  declara lo que validaste**, no en una sección nueva.

## 4. Cuando la plantilla y la convención escrita no coinciden

Un repo puede tener una plantilla pobre y, a la vez, una convención de PR escrita en
`CLAUDE.md`, `AGENTS.md` o `CONTRIBUTING.md` que exija contenido para el que la
plantilla no tiene sección.

Eso no es un conflicto entre el repo y vos: son **dos fuentes del repo**. Gana la
suma. Mantené la estructura de la plantilla y agregá al final **sólo** las secciones
que la convención escrita exige y la plantilla no cubre.

El disparador es una **exigencia escrita** del repo. No lo uses para colar secciones
que a vos te parecen útiles.

## 5. De dónde sale el material

El flujo ya produjo casi todo el cuerpo. No lo re-derives leyendo el diff de nuevo:

| Sección | Material |
|---|---|
| qué cambia y por qué | el bloque `## Contrato` del archivo de estado |
| decisiones | la bitácora: lo que ajustaste tras la crítica de `codex` (Puerta 1) y los hallazgos que tomaste o descartaste (Puerta 2) |
| incertidumbres cerradas | las preguntas del encuadre (fase 1) y lo que cerró la crítica |
| lo que no está resuelto | los hallazgos descartados con su razón, y los no-objetivos del contrato |
| despliegue | el punto de orden de despliegue del plan (fase 3) |
| validado localmente | lo que corriste en la fase 6, con sus comandos |

**Filtralo por los gatillos del esqueleto.** Que la bitácora registre una decisión no
la hace publicable: tiene que calificar por el gatillo de su sección. La mayoría de lo
que anotaste durante el flujo no califica, y eso está bien.

## Lo que no va

- **No pegues el plan ni la crítica como artefactos.** Alimentan las secciones de
  arriba ya destilados; no van crudos.
- **No listes las fases del flujo.** Al que revisa no le importa cómo se hizo.
- **No narres las iteraciones de la Puerta 2.** El diff final es el entregable; los
  intentos descartados no existen en la historia. La excepción es una alternativa que
  califique por el gatillo de decisiones — esa sí va, destilada.
- **No repitas lo que cuenta la serie de commits**: qué cambió, en qué orden, cómo
  está partido.

El cuerpo de la PR describe **el árbol final y lo que los commits no pueden contar**.

## Después

Decile al usuario que la PR quedó abierta, con su URL. La revisión automática va a
correr sobre ella: resolver esos comentarios es otra skill, no esta.
