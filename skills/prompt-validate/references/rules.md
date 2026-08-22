# Reglas de auditoría (R-1..R-11) — detalle

Las 11 reglas de auditoría. Cada una incluye el patrón a detectar y ejemplos MAL → BIEN.

> **Importante:** los códigos `R-1`, `R-2`, … son **referencia interna** de este documento. En el output al usuario usar siempre el **nombre/descripción** de la regla, nunca el código.

## R-1 · Usar verbos imperativos directos

Detectar verbos en modo no-imperativo dirigidos al agente (`puedes`, `podrías`, `deberías`, `sería bueno que`, `tendrías que`).

- MAL: `Puedes crear un middleware`
- BIEN: `Crea un middleware`

## R-2 · Evitar lenguaje conversacional

Detectar fórmulas de cortesía o desiderativas dirigidas al agente (`me gustaría que`, `te pido que`, `necesito que por favor`, `quisiera`, `por favor`).

- MAL: `Me gustaría que implementes refresh tokens`
- BIEN: `Implementa refresh tokens`

## R-3 · Usar acciones específicas

Detectar verbos genéricos sin objeto concreto: `mejorar`, `optimizar`, `arreglar`, `pulir`, `revisar` cuando no van acompañados de **qué exactamente** y **cómo**.

- MAL: `Mejora la autenticación`
- BIEN: `Implementa refresh token con expiración automática a los 15 minutos`

## R-4 · Evitar términos subjetivos

Lista cerrada de términos ambiguos para un agente. Detectar:

`limpio`, `bonito`, `elegante`, `robusto`, `escalable`, `profesional`, `moderno`, `bien hecho`, `de calidad`, `idiomático`.

- MAL: `Hazlo limpio y moderno`
- BIEN: `Usa componentes pequeños (<150 líneas) y separación feature-based bajo /features`

Cada término subjetivo detectado **debe** convertirse en una restricción observable concreta en la sugerencia.

## R-5 · Delimitar el alcance

**Solo aplica a prompts de tipo Técnico.** En prompts **Funcionales** (descripciones de comportamiento, user stories, criterios de aceptación), la falta de ruta o módulo concreto no es una oportunidad de mejora: marcar la regla como **N/A** y excluirla del cálculo de efectividad.

Detectar instrucciones globales sin ruta, módulo o entidad concreta: `refactoriza el proyecto`, `revisa todo el código`, `actualiza el sistema`.

- MAL: `Refactoriza el proyecto`
- BIEN: `Refactoriza únicamente /features/auth`

## R-6 · Usar "NO" explícitos

Detectar prohibiciones tibias o sugeridas: `preferiblemente no`, `evita en lo posible`, `trata de no`, `intenta no usar`.

- MAL: `Preferiblemente no usar Redux`
- BIEN: `No uses Redux`

## R-7 · Evitar instrucciones implícitas

Detectar apelaciones a estándares no especificados: `buenas prácticas`, `código de calidad`, `como debe ser`, `siguiendo las convenciones`, `código limpio`, `SOLID` (sin más detalle).

- MAL: `Hazlo siguiendo buenas prácticas`
- BIEN (checklist explícito):
  - Evita lógica en componentes
  - Usa hooks para estado
  - Separa dominio e infraestructura

Cada apelación implícita debe convertirse en una lista de reglas explícitas en la sugerencia (con tu mejor inferencia razonable del contexto, marcando claramente que son suposiciones a confirmar).

## R-8 · Una intención por frase

Detectar frases que mezclan múltiples objetivos con `y aprovecha para`, `y de paso`, `y también`, `mientras tanto`, `además optimiza`.

- MAL: `Implementa autenticación y aprovecha para mejorar el routing y optimizar el código`
- BIEN (descompuesto):
  - `Implementa autenticación.`
  - `No modifiques routing global.`
  - `No optimices módulos no relacionados.`

## R-9 · Usar "solo / únicamente / exclusivamente"

Detectar instrucciones de alcance que **podrían** ser absolutas pero no llevan el cuantificador exclusivo. Aplicar cuando R-5 está **cumplida** (hay un alcance concreto) **o N/A** (prompt funcional) pero falta el refuerzo de exclusividad; es decir, R-9 **no** aplica solo si R-5 está incumplida. En prompts funcionales (R-5 = N/A) R-9 puede ofrecerse igualmente como mejora opcional cuando el prompt delimita un comportamiento o entidad sin reforzar la exclusividad.

- ACEPTABLE: `Modifica archivos dentro de /auth`
- MEJOR: `Modifica únicamente archivos dentro de /auth`

Esta regla es de **refinamiento**: marcarla como `Mejora opcional · refinamiento de exclusividad` en el output y **no incluirla** en el cálculo del porcentaje de efectividad.

## R-10 · Usar nombres exactos

**Solo aplica a prompts de tipo Técnico.** En prompts **Funcionales** se usa lenguaje de dominio ("el flujo de autenticación", "el usuario", "la solicitud"), no nombres de clase, archivo o función: marcar la regla como **N/A** y excluirla del cálculo de efectividad.

Detectar sustantivos genéricos cuando el contexto permitiría un nombre específico: `un servicio`, `el componente`, `una clase`, `un método`.

- MAL: `Crea un servicio`
- BIEN: `Crea AuthService`

Si el prompt no contiene el contexto suficiente para sugerir un nombre exacto, marcar la sugerencia como `Crea <Nombre>Service` y pedir al usuario que rellene el nombre.

## R-11 · Usar formato checklist para listas de tareas

Detectar prompts que enumeran ≥2 acciones en una sola línea separadas por comas o `y` sin formato de lista.

- MAL: `Implementa middleware auth, hook useAuth y redirect a /login`
- BIEN:

  ```
  Implementa:
  - middleware auth
  - hook useAuth
  - redirect a /login
  ```

> **Desempate R-8 vs R-11:** un mismo fragmento puede disparar ambas. Si es una **lista de acciones en una sola línea / inline** (separadas por comas o `y`) → contar como **R-11**. Si son **intenciones separadas en prosa u oraciones distintas** (típicamente con `y aprovecha para`, `y de paso`, `y también`) → contar como **R-8**. No contar la misma evidencia como dos violaciones distintas.
