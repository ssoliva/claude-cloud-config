---
paths:
  - "**/*.{ts,tsx,js,jsx,mjs,cjs}"
  - "**/*.py"
---

# Nombres: casos y ejemplos

Los invariantes están en `~/.claude/policies/agent-policies.md` (§Nombres). Acá va cómo aplicarlos.

## Cómo elegir el nombre

1. Di qué **es** el valor en términos del dominio, no de dónde vino: "la decisión sobre la respuesta a
   la plantilla", no "lo que devolvió la llamada anterior".
2. Busca con `grep` cómo llama el repo a ese concepto y reutiliza esa palabra.
3. Agrega el discriminante que lo vuelve exacto: `active`, `pending`, `customerFacing`.
4. Agrega unidad o marco si el tipo no los lleva: `Ms`, `Sec`, `Clp`, `Utc`.
5. Recorta hasta el largo que el alcance justifica.

## Forma según lo que contiene

| Contiene | Forma | Ejemplo |
|---|---|---|
| colección | plural | `orders` |
| mapa o índice | `<valor>By<clave>` | `ordersById`, `countByStatus` |
| conteo | `<sustantivo>Count` | `retryCount` |
| cota | `max*` / `min*` | `maxAgeMs` |
| booleano | predicado afirmativo | `hasPrescription` |
| función con efecto | frase verbal | `sendTemplateMirror()` |
| predicado | `is*` / `has*` | `isTemplateMirror()` |
| conversión | `to*` / `as*` | `toSeconds()` |
| construcción | `create*` / `build*` | `buildConversationHistory()` |

## Antipatrones, con el arreglo

```ts
const result = await geocodeAddress(address);    // ✗ nombra la procedencia
const geocoded = await geocodeAddress(address);  // ✓ nombra el referente

const data = await executeGraphQL<NextOrdersData>(...);   // ✗ dos veces ruido
const nextOrders = await executeGraphQL<NextOrders>(...); // ✓

let filtered = orders;      // ✗ nombra la operación, y además muta
const ordersInWindow = ...; // ✓ nombra el resultado

nowMs - sent.createdAt * 1000     // ✗ el `* 1000` se lee como número mágico
nowMs - sent.createdAtSec * 1000  // ✓ se lee como conversión de unidades
```

El caso más caro es un concepto con varios nombres: el lector no puede saber si son lo mismo. Si el
dominio dice `derive`, no aparecen `handoff`, `escalate` ni `transferToHuman` como sinónimos.

## Nombres cortos que sí corresponden

- índices y acumuladores de bucle: `i`, `acc`
- matemática y geometría donde la letra es la convención: `dx`, `dy`, `xi`, `yj`
- callbacks de una línea donde el tipo es evidente: `messages.filter((msg) => ...)`
- el error de un `catch`

## Dónde no aplica

- Campos que replican un contrato externo (`created_at`, `reply_policy`, `content_attributes`): se
  escriben como los define el contrato.
- Tests: el sujeto del `expect` igual se nombra (`decision`, `httpStatus`, `reply`), pero un `result`
  que ya existe en un test no se toca solo por esto.

## Si dudas

Renombrar preserva el comportamiento y cuesta poco, así que escribe el nombre explícito. Si queda
demasiado largo, casi siempre es señal de que la variable junta dos cosas y hay que partirla.
