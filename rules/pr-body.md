# Esqueleto de PR

El cuerpo de la PR existe para cargar **lo que la serie de commits no puede cargar**.
Todo lo demás —qué cambió, en qué orden, cómo está partido— ya está en los commits y
no se repite acá.

Idioma: cuerpo en español, título en inglés con Conventional Commits
(`~/.claude/policies/agent-policies.md`, §Idioma de los artefactos).

## Las dos reglas que gobiernan todo

1. **Sección sin gatillo se borra entera.** Nunca `N/A`, nunca "Ninguno". Una sección
   vacía entrena al revisor a saltearse el cuerpo, y a partir de ahí no lee ninguna.
2. **Toda afirmación cita una ruta, un comando o un número.** Lo que no se puede
   chequear contra el árbol, se borra.

**Referenciá los commits por subject, nunca por sha.** Un `/recommit` reescribe la
historia y los shas dejan de existir: el cuerpo queda mintiendo y nadie se entera.

## La línea de encabezado

Va siempre, arriba de todo, como blockquote:

```markdown
> **Riesgo:** `compone` | `extiende` | `agrega` | `rompe` · **Despliega:** <unidades, en orden> · **Decisiones a validar:** <n> · **Abierto:** <n>
```

| Riesgo | Cuándo |
|---|---|
| `compone` | usa capacidades existentes tal cual: wiring, call sites, refactor |
| `extiende` | cambia el comportamiento de algo que ya existe |
| `agrega` | capacidad o unidad desplegable nueva |
| `rompe` | cambia un contrato del que ya depende algo desplegado |

Si el repo tiene su propio vocabulario de riesgo, gana el del repo.

**Invariante:** si `## Despliegue` dice algo, el riesgo no puede ser `compone`.

Los dos contadores se derivan del cuerpo: `Decisiones a validar` es la cantidad de
entradas de `## Decisiones`; `Abierto`, la de `## Lo que no está resuelto`. Si una de
esas secciones se borró, su contador es `0`.

## Las secciones, en orden

### `## Qué / Por qué` — obligatoria

3–5 frases: la capacidad que se mueve y el problema que resuelve. **Sin listas de
archivos** — eso es la serie de commits. Link al ticket o incidente si existe.

### `## Decisiones` — con gatillo

Una decisión califica **sólo** si cumple una de estas tres:

- **a)** cambia un contrato de borde (GraphQL, ruta, columna, parámetro de
  configuración, secreto, evento),
- **b)** revertirla cuesta más que rehacer el PR,
- **c)** había una alternativa razonable que un revisor podría preferir.

Elegir el nombre de una variable no es una decisión. **Si no califica ninguna, borrá
la sección entera** — es la señal más útil que puede dar un PR mecánico.

Esta sección existe porque es lo único que la serie de commits no puede cargar: la
historia se reconstruye desde el árbol final, así que **la alternativa descartada no
deja rastro en ningún commit ni en el diff**.

```markdown
### 1. <la decisión, en una frase>

- **Elegido / descartado:** <X> sobre <Y>, porque <razón>
- **Costo si me equivoqué:** <qué se rompe y qué tan caro es revertirlo>
- **Dónde aterriza:** `<subject del commit>`
```

### `## Incertidumbres cerradas` — con gatillo

**Gatillo:** se cerró una pregunta abierta que un revisor podría volver a abrir.

**Filtro:** si explicarla obliga a abrir un archivo, no va acá — va como comentario
inline sobre la línea.

Formato: `<pregunta> → <respuesta, y cómo la cerré>`.

### `## Lo que no está resuelto` — con gatillo

**Gatillo:** queda algo abierto a propósito. Decir por qué es aceptable ahora y qué lo
cerraría.

**Omitir esta sección afirma que no queda nada abierto.** No la borres por pereza.

### `## Diagrama` — con gatillo

**Gatillo:** el flujo es nuevo, o cambia de forma. Un refactor no lleva diagrama.

**Altura:** unidades desplegables y saltos entre procesos. Nunca clases ni funciones.

`mermaid`, con línea en blanco antes y después del fence.

### `## Despliegue` — con gatillo

**Gatillo:** hay migración, parámetro de configuración, secreto, variable de entorno,
orden entre unidades, o ventana de incompatibilidad.

El orden de despliegue lo declara el perfil del repo. Si una unidad de este PR depende
de un esquema, índice o parámetro que **este mismo PR** crea, hay ventana de rotura:
**decilo acá con su duración**, no lo dejes implícito.

### `## Validado localmente` — obligatoria

Qué **corriste**, contra qué datos, y qué quedó sin cubrir. Declarativo, en pasado: no
son instrucciones para el revisor.

Es la única parte del cuerpo que afirma que un humano ejecutó algo, y su trabajo es
**delimitar qué le queda al que revisa**.
