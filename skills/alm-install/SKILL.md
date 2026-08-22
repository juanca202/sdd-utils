---
name: alm-install
description: Instala, configura o repara el MCP local de una herramienta ALM (Application Lifecycle Management) en Claude Code o Cursor, con autenticación por token. Soporta Azure DevOps (ADO, @azure-devops/mcp) y Jira (mcp-atlassian). Primero pregunta con la herramienta de preguntas estructuradas qué herramienta instalar (ADO o Jira) y en qué agente(s) (Claude Code y/o Cursor); luego detecta, por agente, si la integración ya existe y, de existir, prueba que esté bien configurada e informa su estado; si no existe, ejecuta el flujo de instalación creando los archivos correspondientes de cada agente. Usar cuando el usuario pida instalar, configurar, reparar o verificar el MCP de Azure DevOps o de Jira, ADO MCP, Jira MCP, dev.azure.com, *.atlassian.net, PAT o API token en Cursor o Claude Code, o agregar una cuenta/organización adicional a una integración ya existente.
license: MIT
---

# ALM Install — MCP de Azure DevOps o Jira en Claude Code / Cursor

Skill genérico para conectar una herramienta ALM al MCP **local** de un agente con autenticación por token. Dos dimensiones:

- **Plataforma** (qué instalar): **ADO** → [references/ado/install.md](references/ado/install.md) · **Jira** → [references/jira/install.md](references/jira/install.md)
- **Agente** (dónde instalar): **Cursor** y/o **Claude Code** → [references/agents.md](references/agents.md)

Filosofía común: MCP **local** (no remoto/OAuth), token guardado de forma segura fuera del chat (Keychain en macOS, variable de usuario en Windows), soporte multi-cuenta y verificación antes de reiniciar el agente. La entrada del servidor es casi idéntica entre agentes; cambia **dónde** se guarda y la **sintaxis de la variable de token** (ver [agents.md](references/agents.md)).

## Flujo del agente



### Paso 1 — Preguntar qué instalar (obligatorio)

**Antes de cualquier otra acción**, usar la **herramienta de preguntas estructuradas** (AskUserQuestion) para preguntar la **plataforma**:

- **Azure DevOps (ADO)** — work items, repos y pipelines de `dev.azure.com`.
- **Jira** — issues y proyectos de `*.atlassian.net`.

No asumir la plataforma aunque el usuario haya nombrado una; confirmar salvo que ya la haya indicado de forma inequívoca.

### Paso 2 — Preguntar en qué agente(s) instalar (obligatorio)

Con la misma herramienta de preguntas estructuradas (permitir **selección múltiple**), preguntar el/los **agentes**:

- **Cursor**
- **Claude Code**

Se puede elegir uno o ambos. Las rutas de archivo y la sintaxis de variables de cada agente están en [references/agents.md](references/agents.md).

### Paso 3 — Detectar si ya existe (por agente y plataforma)

Para **cada agente seleccionado**, revisar sus archivos de config y buscar una entrada de la plataforma (lógica de detección y rutas en [references/agents.md](references/agents.md)):

- **Cursor:** `.cursor/mcp.json` (proyecto) y `~/.cursor/mcp.json` (global).
- **Claude Code:** `.mcp.json` (proyecto) y `~/.claude.json` (usuario).

Identificación de la entrada: **ADO** → `args` contiene `@azure-devops/mcp`; **Jira** → `args` contiene `mcp-atlassian` o `env` incluye `JIRA_URL`/`JIRA_API_TOKEN`. Si un archivo existe pero **no es JSON válido**, respaldar y avisar; no sobrescribir a ciegas.

### Paso 4 — Ramificar según lo detectado

Combinar el estado de cada agente y actuar así:

- **No instalado en el agente pedido** → instalar ahí (Paso 5).
- **Ya instalado en un agente, y el usuario pide instalar otra vez:**
  - Si lo pidió para **ese mismo** agente → **no reinstalar**; probar y reportar estado (Paso 6). Informar: *«Ya está instalado en {agente}»*.
  - Si además **no** está en el otro agente → **sugerirlo**: p. ej. *«Ya está instalado en Cursor. ¿Quieres instalarlo también en Claude Code?»* y, si acepta, instalar solo en el que falta.
- **Ya instalado en AMBOS agentes** → **no instalar nada**; solo **informar** al usuario que ya está en Cursor y Claude Code (y ofrecer verificar/reparar si lo desea).

En resumen: instalar únicamente en los agentes donde falte; para los que ya lo tienen, informar y ofrecer prueba/reparación.

### Paso 5 — Instalar (agente donde falte)

Para cada agente a instalar, seguir el flujo de la plataforma elegida de principio a fin:

- **ADO** → [references/ado/install.md](references/ado/install.md)
- **Jira** → [references/jira/install.md](references/jira/install.md)

Cada flujo detecta el SO (macOS/Windows), calcula identificadores, guía el almacenamiento seguro del token (solo ese paso se muestra al usuario), y **escribe la entrada en el archivo del agente correspondiente** usando su ruta y su sintaxis de `{TOKEN_REF}` (ver [agents.md](references/agents.md)). El **secreto del token se guarda una sola vez** por cuenta (Keychain/variable de usuario) y **se reutiliza** en ambos agentes; solo se duplica la entrada del `mcp.json`/`.mcp.json`. Verificar internamente y pedir reiniciar el agente.

### Paso 5-bis — URL del proyecto y persistencia del contexto (antes de finalizar)

**Antes de dar por terminada la instalación**, preguntar al usuario la **URL del proyecto** (ADO: `https://dev.azure.com/{ORG}/{PROJECT}`; Jira: URL del site con la key del proyecto). Extraer de ella la información necesaria para conectarse y persistirla (detalle y formato en [agents.md](references/agents.md) → «Contexto del proyecto (URL)»):

- **En el MCP, si encaja** — org/site ya quedan en la entrada del `mcp.json`/`.mcp.json` (ADO: `{ORG}` en `args`; Jira: `JIRA_URL` en `env`). Validar que coincida con lo configurado.
- **En la memoria persistente del proyecto, el resto** — proyecto por defecto, URL completa, correo y nombre de la variable de token. Guardarlo en la memoria persistente del proyecto del agente y **anexar sin borrar** lo previo. **Nunca** escribir el token ni secretos ahí.



### Paso 6 — Si ya existe: probar y reportar estado

**No reinstalar.** Ejecutar la verificación de la plataforma para comprobar la configuración:

- **ADO:** «Verificación del agente» de [references/ado/macos.md](references/ado/macos.md) o [references/ado/windows.md](references/ado/windows.md).
- **Jira:** «Verificación del agente» de [references/jira/macos.md](references/jira/macos.md) o [references/jira/windows.md](references/jira/windows.md).

Validar, por cuenta: que la variable/secreto exista y que la API responda **JSON real** (no una página de login). Reportar el estado por agente y cuenta:

- ✅ *Conectada y funcional*.
- ⚠️ *Configurada pero con problema* — indicar la causa y ofrecer reparar.
- ➕ Si quería **agregar otra cuenta/organización** distinta, continuar el flujo de instalación para esa cuenta nueva sin tocar las existentes.



## Reglas transversales

- **Nunca** pedir ni almacenar el token en el chat. El token se guarda solo en Keychain (macOS) o variable de usuario (Windows), **una vez** y reutilizado por todos los agentes.
- **Fusionar, no reemplazar:** al escribir cualquier archivo de config, conservar todas las entradas previas (de ADO, Jira u otros MCP).
- **Sintaxis por agente:** Cursor usa `${env:VAR}`; Claude Code usa `${VAR}`. No mezclar (ver [agents.md](references/agents.md)).
- **MCP local únicamente:** no configurar servidores remotos/OAuth.
- **HTTP 200 no basta:** validar que la respuesta sea JSON real, no una página de login.
- Multi-cuenta: cada par (organización/site + correo) es una entrada independiente con su propia clave de servidor y variable de token.

