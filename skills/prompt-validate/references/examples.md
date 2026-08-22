# Formato de salida, ejemplos y notas operativas

Detalle de la estructura de salida, reglas de formato de las listas de reglas, antipatrones y notas de ejecución. La forma resumida está en `SKILL.md`.

## Estructura de salida completa

El skill **siempre** responde con esta estructura:

```
## Análisis del prompt

**Prompt analizado:**
> [transcripción del prompt; secretos como `[REDACTED]`]

**Tipo detectado:** Funcional | Técnico

**Efectividad: XX%**

**Reglas no evaluables:** [descripciones, ver reglas de formato abajo]

**Reglas cumplidas:** [descripciones, ver reglas de formato abajo]

### Sugerencias de mejora

**1. [Título corto de la mejora]**
- Texto actual: "<fragmento citado; secretos como `[REDACTED]`>"
- Propuesta: "<reemplazo concreto>"
- Por qué mejora: <explicación en 1 línea>
- *Regla: [nombre de la regla]*

[... una entrada por cada sugerencia, numeradas en orden de regla ...]

### Prompt reescrito (propuesta)

> [prompt completo aplicando todas las sugerencias]
```

Si el prompt **no tiene oportunidades de mejora**, omitir la sección "Sugerencias de mejora" y la propuesta reescrita; el bloque de cabecera (Tipo, Efectividad, Reglas no evaluables, Reglas cumplidas) se mantiene seguido de:

```
Sin sugerencias.
```

Si el usuario pidió **solo sugerencias** o **solo prompt reescrito**, devolver únicamente esa sección manteniendo el bloque de cabecera arriba.

## Reglas de formato para listas de reglas

Aplicar a `Reglas no evaluables`, `Reglas cumplidas` y a la cita `Regla:` dentro de cada sugerencia:

- **No mostrar códigos** (`R-1`, `R-2`, …): usar siempre el **nombre/descripción** de la regla (p. ej. `Usar verbos imperativos directos`).
- **No numerar** los elementos (sin `1.`, `2.`, …): el orden no aporta valor.
- **Siempre presentar como lista con viñetas**, una regla por línea, independientemente de la cantidad de elementos.
- **Separar cada bloque de cabecera con una línea en blanco** (`Tipo detectado`, `Efectividad`, `Reglas no evaluables`, `Reglas cumplidas`): sin línea en blanco el renderizador los fusiona en un mismo párrafo.
- En `Reglas no evaluables` se puede añadir un paréntesis breve aclarando el motivo cuando sea no obvio: p. ej. `Delimitar el alcance (N/A en prompt funcional)`.
- **R-9** se reporta siempre como bloque aparte `Mejora opcional · refinamiento de exclusividad` (no en `Reglas cumplidas` ni en `Sugerencias de mejora`), sin afectar el porcentaje.

**Ejemplo de formato:**

> **Reglas no evaluables:**
> - Delimitar el alcance (N/A en prompt funcional)
> - Usar nombres exactos (N/A en prompt funcional)
>
> **Reglas cumplidas:**
> - Usar verbos imperativos directos
> - Evitar lenguaje conversacional
> - Usar acciones específicas
> - Evitar términos subjetivos
> - Usar "NO" explícitos
> - Evitar instrucciones implícitas
> - Una intención por frase
> - Usar formato checklist para listas de tareas

## Cálculo de efectividad

- **Efectividad = (Reglas cumplidas / Reglas evaluables) × 100**, redondeado al entero más cercano.
- **Si Evaluables (N) = 0** (todas las reglas resultan N/A), **no** reportar porcentaje (evita la división por cero): mostrar `Efectividad: N/A` con la nota *"Sin reglas evaluables aplicables"*.
- **Reglas evaluables** = de las 11, las que aplican al prompt. Descontar las marcadas como **N/A**:
  - R-5 y R-10 son **N/A** si el tipo de prompt es **Funcional**.
  - R-11 es **N/A** si el prompt no enumera ≥2 acciones.
  - Cualquier regla cuyo patrón no aplica al contenido del prompt.
- **R-9** (refinamiento opcional) **no entra** en el cálculo del porcentaje. Si aplica, se muestra como sugerencia separada bajo el título `Mejora opcional · refinamiento de exclusividad`, sin afectar la efectividad.

### Cómo contar "reglas evaluables"

No todas las 11 reglas aplican a todos los prompts. En el bloque de efectividad:

- **Evaluables (N)** = reglas que aplican al prompt (entre 1 y 10, ya que R-9 nunca cuenta).
- **Cumplidas (X)** = evaluables sin sugerencia.
- **Efectividad** = `round((X / N) × 100)`.

Reglas que pueden ser **N/A** (y se excluyen de N):

- **R-5** y **R-10** si el tipo de prompt es **Funcional**.
- **R-11** si el prompt no enumera ≥2 acciones.
- Cualquier regla cuyo patrón no aparece en el contenido del prompt.

**R-9** nunca cuenta en N ni en X; si aplica, se reporta como sugerencia separada bajo `Mejora opcional · refinamiento de exclusividad` sin afectar el porcentaje.

## Antipatrones

Evitar al ejecutar este skill:

- **Cumplir el prompt** en lugar de auditarlo (p. ej. el usuario pega `Crea un middleware` y el agente crea el middleware).
- **Inventar sugerencias** que no derivan de las 11 reglas (no es un skill de estilo libre).
- **Aplicar R-5 o R-10 a un prompt Funcional**: en descripciones de comportamiento o user stories, la ausencia de rutas y nombres de clase es esperada, no una oportunidad de mejora. Marcar **N/A** y excluir del cálculo.
- **Penalizar el refinamiento opcional en la efectividad**: la regla de exclusividad (`solo`/`únicamente`) es refinamiento, no entra en el porcentaje.
- **Reescribir la intención** del prompt en la propuesta final (añadir features, cambiar el stack, decidir por el usuario decisiones de arquitectura no implicadas).
- **Parafrasear el "Texto actual"** de forma que el usuario no pueda localizar el fragmento en su prompt (salvo redacción de `[REDACTED]` en credenciales).
- **Reproducir credenciales** del prompt del usuario en la transcripción, en "Texto actual" o en el prompt reescrito.
- **Devolver solo la propuesta reescrita** sin las sugerencias (perdería valor pedagógico). Excepción: el usuario lo pide explícitamente.
- **Omitir la versión reescrita** salvo si el usuario lo pidió.
- **Omitir la efectividad** o sustituirla por `X/N`: el porcentaje es la métrica principal y debe aparecer siempre.
- **Auditar un prompt cuyo texto no se ha recibido** (no inferir, pedir).
- **Formular las sugerencias como acusaciones** ("incumples la regla X"): redactarlas como mejoras propuestas ("reemplaza ... por ...; reduce ambigüedad para el agente").
- **Mostrar códigos de regla (`R-1`, `R-2`, …) al usuario**: en el output usar el nombre/descripción de la regla, no el código. Los códigos son referencia interna de este documento.
- **Numerar `Reglas cumplidas` o `Reglas no evaluables`**: usar siempre viñetas, una regla por línea; no usar `1.`, `2.`, … ni separación por coma.
- **Mostrar el desglose `(X de N reglas evaluables cumplidas)`** junto a la efectividad: el porcentaje basta; la información de cuáles reglas se evaluaron ya está en `Reglas no evaluables` y `Reglas cumplidas`.

## Notas

### Tono de las sugerencias

Redactar cada sugerencia en clave de mejora, no de fallo:

- BIEN: *"Reemplaza el texto conversacional por un imperativo directo: reduce ambigüedad para el agente."*
- MAL: *"Incumples R-2 porque usas lenguaje conversacional."*

### Orden de presentación

Las sugerencias se numeran en el orden de las reglas (R-1 → R-11), no por severidad. R-9, si aplica, va al final como `Mejora opcional`.

### Detección del tipo de prompt

El parámetro `type` (`Funcional` o `Técnico`) lo declara el usuario. Solo si **no** lo declara, el agente lo infiere con esta heurística:

| Señal | Tipo |
|---|---|
| `como ... quiero ... para ...`, criterios de aceptación, "el sistema debe", "el usuario puede" | Funcional |
| Verbos imperativos de código (`implementa`, `crea`, `refactoriza`, `migra`), referencias a rutas/archivos/clases | Técnico |
| Mezcla ambigua o muy corto | Asumir Técnico |

Si el tipo fue **inferido**, declararlo en el output y ofrecer al usuario corregirlo en una nueva pasada con `type: Funcional` o `type: Técnico`.

### Prompts muy cortos

Para prompts ≤10 palabras (p. ej. `arregla esto`), priorizar R-3 (acción específica) y R-5 (alcance, si es Técnico) en las sugerencias; las demás reglas pueden ser N/A.

### Cuándo pedir contexto adicional

Pedir contexto al usuario **solo** si:

- El prompt referencia archivos, módulos o nombres que el agente no puede ver y la sugerencia depende de ellos (R-10, solo aplica a Técnico).
- El prompt apela a "buenas prácticas" (R-7) y el stack/dominio cambia radicalmente lo que esas prácticas son (p. ej. React vs Quarkus).
- El prompt llega indirectamente (captura, "el de ayer") y no se tiene el texto del prompt.

En el resto de casos, auditar con lo recibido y marcar suposiciones en las sugerencias.
