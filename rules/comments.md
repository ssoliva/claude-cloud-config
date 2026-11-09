---
paths:
  - "**/*.{ts,tsx,js,jsx,mjs,cjs}"
  - "**/*.py"
---

# Comentarios: casos y ejemplos

Los invariantes están en `~/.claude/policies/agent-policies.md` (§Comentarios en el código). Acá va cómo
aplicarlos mientras escribes.

## Antes de escribir un comentario

Recorre esto en orden y detente en el primer sí:

1. ¿Puedo borrarlo cambiando un nombre? → cambia el nombre.
2. ¿Puedo borrarlo extrayendo una función con nombre? → extrae la función.
3. ¿Puedo borrarlo poniendo la unidad o el marco en el nombre? → ponlo (`maxAgeMs`, `priceClp`).
4. ¿Cae en una de las cuatro categorías de la política? → escríbelo, en inglés, en una línea.
5. Si no → no va.

Un comentario que sobrevive los cuatro primeros pasos es raro. Que el diff cierre con cero comentarios es
el resultado normal, no un descuido.

## Idioma: inglés, aunque el dominio sea español

```ts
// ✗ Si la receta ya fue dispensada no se puede volver a reservar stock
// ✓ A dispensed receta cannot reserve stock again
```

El sustantivo del dominio se mantiene tal como lo nombra el repo (`receta`, `convenio`, `bodega`); la
prosa alrededor va en inglés. Nunca traduzcas el término del dominio solo para que la frase quede en
inglés: eso introduce un sinónimo (§Nombres, un término por concepto).

## Categoría 1: formato o unidad — inline, a la derecha, solo el token

El formato de un campo **no** se documenta con un bloque arriba ni con una explicación: va como anotación
inline a la derecha, y contiene únicamente el token del formato.

```ts
// ✗ bloque arriba + explicación
/** Fecha de entrega, en formato ISO sin hora (YYYY-MM-DD) */
deliveryDate: string;

// ✗ inline pero con prosa
deliveryDate: string;   // fecha en formato YYYY-MM-DD

// ✓ inline, solo el formato
deliveryDate: string;   // YYYY-MM-DD
```

```ts
interface CenabastLinkRow {
  zcenCode: string;       // ZCEN-000000
  tenderEndsAt: string;   // ISO 8601
  reportedAt: string;     // YYYY-MM-DD HH:mm
  ceilingPrice: number;   // CLP
  phone: string;          // E.164
  ttl: number;            // seconds
}
```

Tokens que califican: `YYYY-MM-DD`, `HH:mm`, `ISO 8601`, `RFC 3339`, `E.164`, `CLP`, `USD`, `ms`,
`seconds`, `bytes`, `UTC`, un ejemplo literal (`ZCEN-000000`), o un patrón (`^\d{8}-[\dkK]$`).

No califica: nada con verbo, nada que repita el nombre del campo, nada que explique para qué sirve el
campo. Si necesitas una oración, no es esta categoría: revisa si es la 2, la 3 o la 4 — o si el que falla
es el nombre.

**Primero intenta el nombre.** Si el campo se puede renombrar, la unidad va en el nombre y no hay
comentario:

```ts
createdAt: number;      // seconds        ✗ si el campo es tuyo
createdAtSec: number;   //                ✓
```

La anotación inline es sobre todo para campos que replican un contrato externo (payload de una API,
columna, evento) y que por eso no puedes renombrar.

## Antipatrones, con el arreglo

```ts
// ✗ parafrasea la línea
// Filtra las órdenes activas
const activeOrders = orders.filter((order) => order.isActive);
// ✓ (sin comentario)

// ✗ dice el qué
// Recorre los detalles y suma el total
// ✓ el nombre lo dice
const totalClp = details.reduce((acc, detail) => acc + detail.priceClp, 0);

// ✗ banner de sección: si el archivo necesita secciones, necesita módulos
// ────────────── Helpers ──────────────

// ✗ redundante con la firma
/**
 * @param orderId The order id
 * @returns The order
 */

// ✗ narra el cambio en vez del código; eso va en el commit
// Ahora también consideramos las órdenes en tránsito

// ✗ TODO sin dueño ni condición de cierre
// TODO: mejorar esto

// ✓ categoría 2: la restricción externa no se deduce del código
// EventBridge caps PutEvents at 10 entries, so callers pass the full batch and we chunk here.

// ✓ categoría 3: el lector va a querer "simplificarlo"
// Sequential on purpose: the pharmacy API rejects concurrent reservations for the same receta.

// ✓ categoría 4: el invariante vive en otro archivo
// Guaranteed non-null by the gateway authorizer (see getAuthorizer).
```

## Al tocar código ya comentado

- Dentro de la función o el bloque que estás modificando: aplica la política, borra el ruido.
- Fuera de eso: no toques nada, aunque esté sobre-comentado. Va en su propio commit.
- El archivo sobre-comentado no es precedente: tu código nuevo sigue la política, no el estilo del archivo.

## Casos que suelen confundirse

- **Docstring de API pública** de un paquete compartido (`reusable/`): vale solo si documenta el contrato
  (fail-open, unidades, qué error lanza). Si parafrasea la firma, no va.
- **Tests**: el nombre del `it()`/`test()` es la documentación. Un comentario en un test se justifica casi
  solo para explicar por qué un fixture tiene un valor raro.
- **Migraciones y SQL**: la restricción del motor o del orden de despliegue es categoría 2 legítima.
- **Regex no trivial**: un ejemplo del string que matchea es categoría 1 (un token, no una explicación).

## Antes de cerrar el turno

Relee tu propio diff y, por cada comentario que sobrevivió, di el número de categoría. Si no puedes,
bórralo. Si el diff quedó con más de 3, revisa si el problema son los nombres o el tamaño de las funciones.
