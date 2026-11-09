# Revisión y composición (fase 7)

Dos cosas a la vez, **en un solo mensaje con dos acciones**: el subagente revisor
arranca mientras vos armás la tabla. Si las serializás, pagás dos esperas por nada.

Esta fase corre sobre el **árbol sucio**: nada está commiteado todavía, y por eso los
hallazgos se funden sin dejar rastro en la historia. Es el motivo de que la fase exista
acá y no después de la PR.

## 1. El subagente revisor

`subagent_type: "general-purpose"`, `model: "opus"`. Es el único subagente del flujo.
Existe porque vos escribiste este código: no sos un buen revisor de tu propio trabajo.

**Read-only.** No edita, no commitea, no corre el gate (ya corrió en la fase 6).

Prompt:

```markdown
Revisá un diff que todavía NO está commiteado. Sos read-only: no edites ningún
archivo, no commitees, no corras tests. Sólo reportás.

Repo: <ruta>. Diff: `git diff HEAD` y `git status --short` para los archivos nuevos.

Contexto de lo que se quiso hacer:
<el bloque ## Contrato>

Buscá, en este orden:
1. Bugs de corrección: un caso de entrada que produce un resultado incorrecto o una
   excepción. Describí el escenario concreto, no la categoría.
2. Seguridad: input sin validar, secreto expuesto, permiso que se amplía, inyección.
3. Tests que no pueden fallar: un mock que hace verde el test por construcción, una
   aserción derivada del mismo valor que dice verificar, un caso que no se ejercita.
4. Cobertura faltante: comportamiento nuevo sin test que lo ejercite.
5. Código que duplica algo que el repo ya resuelve. Citá dónde está lo existente.

Por cada hallazgo: severidad (alta/media/baja), `archivo:línea`, qué está mal, el
escenario de falla concreto, y la corrección propuesta.

Máximo 8 hallazgos, el más grave primero. No reportes preferencias de estilo, no
elogies, no resumas el diff de vuelta. Si no encontrás nada de severidad alta,
decilo explícitamente.
```

## 2. La tabla de commits

En paralelo. Si el usuario ya tiene una skill de descomposición en commits, usala;
si no, el formato es este:

| # | Commit | Archivos | Qué hace |
|---|---|---|---|
| 1 | `feat(scope): ...` | `a.ts`, `b.ts` | 1–2 líneas |

- Cada fila es **un cambio coherente y revisable por separado**, en el orden en que
  se aplicarían: primero la base (tipos, interfaces, scaffolding), después la
  implementación, después los tests, al final docs y limpieza.
- Conventional Commits, subject en inglés e imperativo (`add`, no `added`).
- Sin commits `wip` ni `fix typo`: no hay historia previa que arreglar, esta es la
  primera y única versión.
- Si el trabajo es genuinamente un solo commit, decilo en una línea con su nombre.

**La tabla es la unidad de revisión del usuario.** Que cada fila se lea sola es más
importante que minimizar la cantidad de filas.

## 3. La fusión

Presentá las dos cosas juntas, en este orden: hallazgos primero (numerados, con
severidad y `archivo:línea`), tabla después.

Al usuario le preguntás **qué hallazgos tomar**. Los que toma, los arreglás ahí mismo
sobre el árbol sucio. Los que descarta, quedan registrados en la bitácora con su razón
— no se vuelven a proponer en esta feature.

Si los arreglos cambian el diff lo suficiente como para que la tabla ya no lo
describa, **rehacé la tabla**. Es barato: todavía no hay commits.
