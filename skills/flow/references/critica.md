# Crítica externa del plan (fase 4)

Una llamada a `codex`, que es otro modelo y corre fuera del contexto de Claude. Es la
forma más barata de tener una opinión que no sea la tuya: cuesta un prompt de ida y un
resumen de vuelta.

**No lo reemplaces por un subagente de Claude si `codex` falta.** El valor era la
diversidad de proveedor; un subagente de Claude criticando un plan de Claude cuesta
contexto y coincide contigo más de lo que debería. Si no está, decílo y seguí.

## Cómo se corre

Escribí el prompt a un archivo del scratchpad y pasalo por `$(cat ...)`. Un solo
comando, en una línea:

```bash
timeout 330 codex exec -s read-only -C "$PWD" "$(cat /tmp/flow-critica.md)" -c 'model_reasoning_effort="high"' < /dev/null
```

`-s read-only` no puede escribir. `< /dev/null` es obligatorio: sin eso se cuelga
esperando stdin. `timeout` evita que una corrida larga trabe el flujo.

Verificá que existe antes: `command -v codex`.

## El prompt

Lo que le mandás es el **contrato y el plan**, no el repo entero — `codex` ya corre
con `-C` sobre el repo y lee lo que necesite.

```markdown
Sos un ingeniero senior revisando un plan de implementación ANTES de que se escriba
código. El repo está en tu directorio de trabajo; leé lo que necesites.

## Contrato
<el bloque ## Contrato del archivo de estado>

## Plan propuesto
<el bloque ## Plan>

Respondé SÓLO con lo que encontraste, sin resumir el plan de vuelta. Buscá, en este
orden de prioridad:

1. Un supuesto del plan que el código del repo contradice. Citá archivo y línea.
2. Un caso que el plan no cubre y que rompería en producción.
3. Un acoplamiento u orden de despliegue que el plan no ve: código nuevo que depende
   de un esquema, índice, variable o servicio que este mismo cambio crea.
4. Algo que el plan hace de una forma que el repo ya resuelve de otra. Citá dónde.
5. Un test que el plan declara y que no ejercitaría lo que dice ejercitar.

Reglas: máximo 5 hallazgos, el más grave primero. Si no encontrás nada en una
categoría, saltala. Para cada hallazgo: qué está mal, por qué importa, y la
corrección concreta. Nada de elogios ni de preámbulo.
```

## Qué hacer con la respuesta

Triageá vos antes de mostrarla — el usuario no debería leer la salida cruda.

En `## Crítica externa`, por cada hallazgo: **una línea de qué levantó** y **tomado /
descartado + por qué**. Si tomás uno, actualizá el plan antes de la Puerta 1.

Un hallazgo que contradice algo que el usuario ya decidió en la fase 1 se descarta
citando esa decisión, no se re-litiga.
