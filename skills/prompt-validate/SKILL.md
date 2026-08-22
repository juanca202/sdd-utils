---
name: prompt-validate
description: Validar prompts dirigidos a agentes de IA (Claude Code, Cursor, Copilot, etc.) contra reglas de redacción efectiva. Calcular un porcentaje de efectividad del prompt y devolver sugerencias de mejora concretas, más una propuesta de prompt reescrito. Cubre verbos no imperativos, lenguaje conversacional, acciones vagas, términos subjetivos, alcance difuso, prohibiciones implícitas, intenciones múltiples y nombres genéricos. Las reglas de detalle técnico (alcance, nombres exactos) se aplican solo a prompts de implementación; en prompts funcionales (user stories, descripciones de comportamiento) se marcan N/A. Usar siempre que el usuario pida validar, revisar, auditar, mejorar, corregir o "pulir" un prompt antes de enviarlo a un agente, o cuando pegue un prompt y pida feedback sobre cómo está redactado.
license: MIT
---

# Skill: Validador de Prompts

Validar y mejorar prompts dirigidos a agentes de IA aplicando un conjunto fijo de **11 reglas de redacción imperativa**.

El skill **no** ejecuta el prompt ni produce el código pedido; **solo** audita la redacción y devuelve un informe accionable.

---

## Mapa de referencias

Cargar el archivo correspondiente según la fase del análisis (rutas relativas a `prompt-validate/`):

- **`references/rules.md`** — definición íntegra de las 11 reglas R-1..R-11: patrón a detectar y ejemplos MAL → BIEN por regla. Consultar al aplicar las reglas a un prompt.
- **`references/examples.md`** — estructura de salida completa, reglas de formato de las listas de reglas (ejemplo incluido), detalle de cálculo de efectividad y conteo de evaluables, antipatrones y notas operativas (tono, orden, detección de tipo, prompts cortos, cuándo pedir contexto). Consultar al producir el output.

---

## Seguridad

- En todo el informe (transcripción, "Texto actual", prompt reescrito), **enmascarar** credenciales: API keys, PAT, contraseñas, tokens de sesión, claves privadas y valores tras `=` en variables de entorno → `[REDACTED]`.
- **Nunca** reproducir en el chat valores secretos que el usuario haya pegado en su prompt, aunque la cita sea para localizar un fragmento.
- Conservar el resto del texto del prompt tal cual (estructura, rutas, nombres de clase) para que el usuario pueda ubicar las sugerencias.

---

## Propósito

Auditar un prompt recibido del usuario y devolver:

1. Un **porcentaje de efectividad** del prompt (0–100%), basado en cuántas reglas evaluables cumple.
2. Una lista de **sugerencias de mejora** concretas, con: texto actual, propuesta de reemplazo y motivo. La regla aplicada se muestra como referencia discreta, no como acusación.
3. Una **propuesta de prompt reescrito** que aplique todas las sugerencias.

Usar cuando el usuario pida revisar, validar, mejorar o auditar un prompt; o cuando pegue un prompt en el chat y pida feedback sobre su redacción.

---

## Alcance

**Incluye:** análisis de prompts dirigidos a agentes de código (Claude Code, Cursor, Copilot, Cline, etc.); detección de las 11 oportunidades de mejora tipificadas en [Reglas](#reglas); sugerencia concreta de reemplazo por oportunidad; cálculo del porcentaje de efectividad; generación de un prompt reescrito final que integre todas las sugerencias.

**No incluye:**

- Ejecutar o cumplir el prompt (no escribir el código, middleware, refactor, etc. que el prompt pide).
- Evaluar la *corrección técnica* del contenido del prompt (si la solución pedida tiene sentido en el stack del usuario). El skill audita **redacción**, no **arquitectura**.
- Reglas distintas de las 11 listadas (estilo de redacción literaria, ortografía, gramática general).
- Validar prompts para imagen, audio o tareas no-código (el skill está calibrado a prompts de desarrollo).

---

## Tipos de prompt

- **`Funcional`** — describe comportamiento, criterios de aceptacion (`AC-XXX`), reglas de negocio o necesidad del usuario (p. ej. user stories, descripciones de feature, especificaciones de comportamiento). No requiere detalles técnicos como rutas, nombres de clase o archivos. **Las reglas de delimitar alcance (R-5) y usar nombres exactos (R-10) se marcan N/A.**
- **`Técnico`** — solicita implementación, refactor o cambios concretos de código (p. ej. "crea AuthService", "refactoriza /features/auth"). **R-5 y R-10 aplican plenamente.**

El usuario puede declarar el tipo de varias formas: `type: Funcional`, `type=Funcional`, `tipo=Funcional`, o en lenguaje natural ("es un prompt funcional", "este es técnico"). El agente reconoce ambas formas y normaliza variantes sin acento (`tecnico` → `Técnico`). Si **no se declara**, el agente lo **infiere** (ver paso 0 de [Ejecución del análisis](#ejecución-del-análisis)). La heurística de inferencia detallada está en `references/examples.md`.

---

## Entradas

- **Obligatorio:** el **texto del prompt** a auditar, completo. Si el usuario pega secretos, el agente los trata internamente pero **no** los vuelca en la salida (ver [Seguridad](#seguridad)).
- **Opcional:** `type` (ver [Tipos de prompt](#tipos-de-prompt)); lenguaje/stack del proyecto (ayuda a juzgar si un nombre es "exacto" o genérico); si el usuario quiere **solo sugerencias** o **también el prompt reescrito** (por defecto: ambos); si el prompt es **un fragmento** de uno mayor o **independiente**.

Si el prompt llega como captura, imagen o referencia indirecta (p. ej. "el prompt que te pasé ayer"), **pedir** el texto del prompt antes de auditar. No inventar el contenido.

---

## Reglas

Las 11 reglas de auditoría (nombre corto + qué detectan). El **detalle íntegro** —patrón y ejemplos MAL → BIEN por regla— está en `references/rules.md`.

- **R-1 · Usar verbos imperativos directos** — verbos no imperativos dirigidos al agente (`puedes`, `podrías`, `deberías`).
- **R-2 · Evitar lenguaje conversacional** — cortesía o desiderativos (`me gustaría que`, `por favor`, `quisiera`).
- **R-3 · Usar acciones específicas** — verbos genéricos sin objeto concreto (`mejorar`, `optimizar`, `arreglar`).
- **R-4 · Evitar términos subjetivos** — lista cerrada: `limpio`, `bonito`, `elegante`, `robusto`, `escalable`, `profesional`, `moderno`, `bien hecho`, `de calidad`, `idiomático`.
- **R-5 · Delimitar el alcance** — instrucciones globales sin ruta/módulo/entidad. **N/A en Funcional.**
- **R-6 · Usar "NO" explícitos** — prohibiciones tibias (`preferiblemente no`, `evita en lo posible`, `trata de no`).
- **R-7 · Evitar instrucciones implícitas** — apelaciones a estándares no especificados (`buenas prácticas`, `código limpio`, `SOLID`).
- **R-8 · Una intención por frase** — mezcla de objetivos (`y aprovecha para`, `y de paso`, `y también`).
- **R-9 · Usar "solo / únicamente / exclusivamente"** — alcance sin cuantificador exclusivo. **Refinamiento: no entra en el porcentaje.**
- **R-10 · Usar nombres exactos** — sustantivos genéricos (`un servicio`, `el componente`). **N/A en Funcional.**
- **R-11 · Usar formato checklist para listas de tareas** — ≥2 acciones en una sola línea sin formato de lista.

> **Desempate R-8 vs R-11:** lista de acciones en una sola línea / inline → **R-11**; intenciones separadas en prosa u oraciones distintas → **R-8**. No contar la misma evidencia como dos violaciones. Detalle en `references/rules.md`.

---

## Cálculo de efectividad y reglas N/A

- **Efectividad = (Reglas cumplidas / Reglas evaluables) × 100**, redondeado al entero más cercano.
- **Si Evaluables (N) = 0** (todas N/A), mostrar `Efectividad: N/A` con la nota *"Sin reglas evaluables aplicables"* (evita división por cero).
- **Reglas evaluables (N)** = de las 11, las que aplican al prompt. Descontar las **N/A**:
  - **R-5** y **R-10** son N/A si el tipo es **Funcional**.
  - **R-11** es N/A si el prompt no enumera ≥2 acciones.
  - Cualquier regla cuyo patrón no aparece en el contenido del prompt.
- **R-9** (refinamiento opcional) **nunca** entra en N ni en X; si aplica, se reporta como sugerencia separada bajo `Mejora opcional · refinamiento de exclusividad`, sin afectar el porcentaje.

Detalle de conteo (`Evaluables N` / `Cumplidas X`) y casos límite en `references/examples.md`.

---

## Ejecución del análisis

Para cada prompt recibido, el agente debe:

0. **Determinar el tipo de prompt** (ver [Tipos de prompt](#tipos-de-prompt)):
   - Si el usuario lo **declaró** (`type:`/`tipo=`/lenguaje natural), usar ese valor y normalizar variantes sin acento.
   - Si **no se declaró**, inferirlo: **Funcional** si usa lenguaje de comportamiento (`el sistema debe`, `el usuario puede`, `como ... quiero ... para ...`), criterios de aceptacion (`AC-XXX`) o reglas de negocio, y **no** menciona archivos/rutas/clases/verbos de codificación; **Técnico** si pide implementar/refactorizar/crear/modificar código o menciona archivos, rutas, clases, endpoints o verbos de codificación. Si es **ambiguo**, asumir **Técnico** (criterio más estricto). Heurística completa en `references/examples.md`.
   - **Declarar el tipo en el output.** Si fue **inferido**, añadir: *"Tipo inferido; declara `type: Funcional` o `type: Técnico` si quieres cambiarlo."*. Si fue **declarado**, no añadir nota.
1. **Leer el prompt completo** y separarlo en frases u oraciones.
2. **Aplicar las 11 reglas en orden**, frase por frase (ver `references/rules.md`). Una misma frase puede activar varias sugerencias (p. ej. `Me gustaría que mejores el código de forma elegante` activa R-1, R-2, R-3 y R-4). Si el tipo es Funcional, **omitir** R-5 y R-10 (marcar N/A).
3. **Citar el fragmento** en "Texto actual" de cada sugerencia, con la misma redacción que el prompt salvo secretos (`[REDACTED]`). No parafrasear el fragmento original; sí parafrasear/reescribir en la propuesta.
4. **Producir la propuesta más concreta posible.** Si falta contexto (p. ej. para nombrar `AuthService`), proponer un placeholder explícito y pedir confirmación.
5. **Calcular la efectividad** según [Cálculo de efectividad y reglas N/A](#cálculo-de-efectividad-y-reglas-na).
6. **Generar el prompt reescrito** integrando todas las sugerencias, manteniendo la intención original. No añadir requisitos nuevos. Aplicar la redacción de secretos de [Seguridad](#seguridad).

---

## Formato de salida

El skill **siempre** responde con un bloque de cabecera (`## Análisis del prompt`, `Prompt analizado`, `Tipo detectado`, `Efectividad: XX%`, `Reglas no evaluables`, `Reglas cumplidas`) seguido de `### Sugerencias de mejora` (una entrada por sugerencia: Texto actual, Propuesta, Por qué mejora, *Regla*) y `### Prompt reescrito (propuesta)`.

Reglas clave del output (estructura, plantilla exacta y ejemplo en `references/examples.md`):

- **Sin sugerencias:** si el prompt no tiene oportunidades de mejora, mantener la cabecera y escribir `Sin sugerencias.` (omitir secciones de sugerencias y reescritura).
- **Salida parcial:** si el usuario pidió solo sugerencias o solo prompt reescrito, devolver únicamente esa sección, manteniendo la cabecera.
- **Listas de reglas** (`Reglas no evaluables`, `Reglas cumplidas`, cita `Regla:`): **nunca** mostrar códigos `R-x` — usar el **nombre/descripción**; sin numerar; siempre viñetas, una por línea; línea en blanco entre bloques de cabecera; paréntesis breve de motivo cuando sea no obvio (p. ej. `Delimitar el alcance (N/A en prompt funcional)`).
- **R-9** se reporta siempre como bloque aparte `Mejora opcional · refinamiento de exclusividad`, sin afectar el porcentaje.
- **Efectividad** siempre como porcentaje; no sustituir por `X/N` ni mostrar el desglose `(X de N ...)`.

---

## Antipatrones (resumen)

Lista completa en `references/examples.md`. Lo esencial a evitar:

- **Cumplir el prompt** en lugar de auditarlo.
- **Inventar sugerencias** fuera de las 11 reglas.
- **Aplicar R-5 o R-10 a un prompt Funcional** (marcar N/A y excluir del cálculo).
- **Penalizar R-9** en la efectividad (es refinamiento).
- **Reescribir la intención** del prompt o **parafrasear "Texto actual"** de forma que no se localice el fragmento.
- **Reproducir credenciales** del usuario en cualquier parte del informe.
- **Omitir sugerencias, prompt reescrito o efectividad** (salvo que el usuario pida una salida parcial).
- **Auditar sin haber recibido el texto** del prompt (no inferir, pedir).
- **Formular sugerencias como acusaciones** ("incumples R-x"): redactarlas como mejoras propuestas.
- **Mostrar códigos de regla `R-x` al usuario**: usar siempre el nombre/descripción.
