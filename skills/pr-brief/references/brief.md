# Brief y dossier (paso 5)

## El dossier

Es tu memoria para el modo pregunta. Sin presupuesto, pero **todo con fuente**: lo que
no tenga fuente no se puede usar para contestar.

```markdown
# dossier PR #<n> — <título>
head: <sha7> · base: <sha7> · generado: <YYYY-MM-DD>

## Fuentes consultadas
- <fuente> — qué aportó (o "nada relevante")

## Antes
### <zona o archivo>
qué hacía, quién lo llama, para qué existe — <fuente>

## Delta
### <zona o archivo>
qué cambia — `ruta:línea`

## Comportamiento
| Caso | Antes | Después | Declarado | Fuente |

## Decisiones e infra
<todas las candidatas, con evidencia>

## Descartado del brief
- <ítem> — por qué no subió (filtro que no pasó)
```

`## Descartado del brief` existe para que, si el usuario pregunta "¿qué más hay?", tengas
la respuesta sin rehacer la investigación.

## El brief

**Presupuesto:** ≤350 palabras de prosa, sin contar la tabla ni el diagrama. Tabla de
comportamiento ≤6 filas. "Para juzgar" ≤5 ítems. Un solo diagrama (antes y después). Si
no cabe, priorizá; no se amplía el presupuesto.

**Toda sección con gatillo que no se dispara se borra entera.** Nunca "N/A", nunca
"ninguno". En un refactor puro, el brief son el header, el "Qué / por qué" y el "Resto", y
eso es correcto.

```markdown
# PR #<n> — <título>
<autor> · `<base>` ← `<head-branch>` · head `<sha7>` · <YYYY-MM-DD> · [PR](<url>)

> **Zona:** <parte del sistema, 1 línea> · **Tipo:** <producto|refactor|infra|mixto> · **Para juzgar:** <n>

## Qué / por qué
2–3 frases: qué capacidad se mueve y qué problema resuelve. Cita de dónde sale el porqué
(body, issue, wiki). Si el porqué no está en ninguna fuente, decilo: es un hallazgo.

## Antes → después                    ← gatillo: cambia comportamiento observable
| Caso | Antes | Después | Declarado |
|---|---|---|---|
| <situación concreta> | <resultado> | <resultado> | sí (body) / no ⚠ |

## Flujo                              ← gatillo: el flujo cambia de forma
(mermaid, antes y después lado a lado)

## Infra                              ← gatillo: recurso, permiso, migración o env var
| Recurso | Cambio | Qué implica al desplegar |

## Para juzgar                        ← gatillo: hay algo que requiere criterio
1. **<afirmación en una frase>.** <evidencia, `ruta:línea`>. **Preguntar:** <pregunta concreta al autor, si aplica>

## Orden de lectura                   ← gatillo: >8 archivos sustantivos
1. `ruta` — qué mirar ahí

## Resto
Una línea: qué es mecánico (tests espejo, wiring, renombres) y se puede leer rápido.
```

### Qué entra en "Para juzgar"

- una decisión que califica (contrato de borde, cara de revertir, alternativa razonable),
  con la alternativa descartada si se puede inferir
- un cambio de comportamiento **no declarado**
- una decisión de producto implícita: el código elige una regla de negocio que nadie
  escribió
- un riesgo de despliegue: orden entre unidades, migración, ventana de incompatibilidad
- una contradicción entre la PR y una fuente (wiki, spec, issue)

Ordenados por cuánto cambian la decisión de aprobar, el más importante primero.

### Diagramas

Sólo si el flujo **cambia de forma**: aparece o desaparece un paso, un servicio, una cola
o una rama de decisión. Que cambie la implementación de un paso no amerita diagrama.

- Altura: servicios, colas, lambdas, tablas, actores. **Nunca clases ni funciones.**
- ≤8 nodos por lado. Lo que no cambia se dibuja igual en ambos lados, para que el ojo
  encuentre la diferencia; lo nuevo, con `:::nuevo` o un color.
- `mermaid` con `flowchart LR`, dos `subgraph` (Antes / Después) en un solo bloque.

## La pasada de editor

Después de escribir el brief, releelo **entero** contra esta lista y cortá:

1. **Test de la decisión:** por cada oración o fila, ¿el usuario miraría o decidiría
   distinto sin ella? Si no, va al dossier (`Descartado del brief`).
2. **Fuente:** toda afirmación cita `ruta:línea`, una página o la PR. Sin fuente, se borra.
3. **Repetición:** si el body de la PR ya lo explica bien, enlazalo en vez de reescribirlo.
   El valor del brief es lo que la PR **no** dice.
4. **Paráfrasis del diff:** "se agrega el campo X en el archivo Y" no es contexto; el
   usuario lo va a ver en el diff. Sólo sobrevive si dice **por qué importa**.
5. **Preferencias del repo:** aplicá `## Preferencias del brief` del archivo de fuentes.
6. **Presupuesto:** contá las palabras de prosa (`wc -w` sobre el brief sin la tabla ni el
   bloque mermaid). Si pasa de 350, volvé al punto 1 con un criterio más duro.

Idioma: el brief y el dossier en español; los identificadores y las rutas, como están en
el código.
