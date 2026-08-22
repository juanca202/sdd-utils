# Agentes — dónde y cómo instalar el MCP

El MCP se puede instalar en dos agentes. La **entrada** del servidor (`command`, `args`, `env`) es casi idéntica; lo que cambia entre agentes es **dónde** se guarda y **la sintaxis para referenciar la variable de entorno** del token.

| Agente          | Archivo (proyecto)      | Archivo (usuario/global) | Ref. a variable de entorno |
| --------------- | ----------------------- | ------------------------ | -------------------------- |
| **Cursor**      | `.cursor/mcp.json`      | `~/.cursor/mcp.json`     | `${env:NOMBRE_VAR}`        |
| **Claude Code** | `.mcp.json` (raíz repo) | `~/.claude.json`         | `${NOMBRE_VAR}`            |

En ambos, la clave raíz es `mcpServers` y cada servidor es una entrada `{ "command", "args", "env" }`.

## `{TOKEN_REF}` — placeholder por agente

Los flujos de instalación (ADO/Jira) usan el placeholder **`{TOKEN_REF}`** para el valor de la variable de token en `env`. Resolverlo según el agente:

| Agente          | ADO (`ADO_PAT_{ALIAS}`)        | Jira (`JIRA_TOKEN_{ALIAS}`)      |
| --------------- | ------------------------------- | -------------------------------- |
| **Cursor**      | `${env:ADO_PAT_{ALIAS}}`        | `${env:JIRA_TOKEN_{ALIAS}}`      |
| **Claude Code** | `${ADO_PAT_{ALIAS}}`            | `${JIRA_TOKEN_{ALIAS}}`          |

> Claude Code expande `${VAR}` y `$VAR` en `.mcp.json`. Cursor usa `${env:VAR}`. **No** mezclar sintaxis: cada archivo usa la de su agente.

## Ejemplo de entrada por agente (ADO)

**Cursor** (`.cursor/mcp.json`):

```json
{
  "mcpServers": {
    "ado-bay-jua": {
      "command": "npx",
      "args": ["-y", "@azure-devops/mcp", "BayteqDev", "--authentication", "pat"],
      "env": { "PERSONAL_ACCESS_TOKEN": "${env:ADO_PAT_BAYTEQDEV_JUANCA}" }
    }
  }
}
```

**Claude Code** (`.mcp.json`):

```json
{
  "mcpServers": {
    "ado-bay-jua": {
      "command": "npx",
      "args": ["-y", "@azure-devops/mcp", "BayteqDev", "--authentication", "pat"],
      "env": { "PERSONAL_ACCESS_TOKEN": "${ADO_PAT_BAYTEQDEV_JUANCA}" }
    }
  }
}
```

Para Jira es igual, cambiando el bloque `env` por `JIRA_URL` / `JIRA_USERNAME` / `JIRA_API_TOKEN` (ver [jira/install.md](jira/install.md)).

## Detección de instalación existente (por agente y plataforma)

Para cada agente seleccionado, revisar sus archivos (proyecto **y** usuario/global si existen) y buscar dentro de `mcpServers` una entrada de la plataforma:

- **ADO:** `args` contiene `@azure-devops/mcp`.
- **Jira:** `args` contiene `mcp-atlassian`, o `env` incluye `JIRA_URL` / `JIRA_API_TOKEN`.

> En Claude Code, `~/.claude.json` puede anidar la configuración MCP por proyecto; buscar la entrada tanto a nivel raíz como dentro del proyecto actual.

Estados posibles por agente:

- **No instalado** → proceder con el flujo de instalación en ese agente.
- **Instalado** → no reinstalar; ejecutar la verificación y reportar el estado (ver flujo de la plataforma).

## Scope y notas por agente

- **Cursor:** `.cursor/mcp.json` es por proyecto; `~/.cursor/mcp.json` aplica a todos los proyectos. Elegir según lo que el usuario quiera (por defecto, proyecto). Requiere **reinicio completo** (Cmd+Q) tras cambiar variables.
- **Claude Code:**
  - `.mcp.json` (proyecto) se puede commitear al repo → se comparte con el equipo. **No** poner secretos inline; usar `${VAR}`.
  - Alternativamente `claude mcp add` guarda en scope `local`/`user` (`~/.claude.json`), que **no** se sincroniza.
  - La primera vez que se usa un `.mcp.json` de proyecto, Claude Code **pide aprobar** el/los servidores del proyecto. Avisar al usuario.
  - Tras cambiar variables de entorno, reiniciar la sesión de Claude Code para que las tome.

## Contexto del proyecto (URL) — persistencia

Antes de finalizar la instalación, pedir al usuario la **URL del proyecto** y de ahí extraer la información necesaria para conectarse:

- **ADO** — `https://dev.azure.com/{ORG}/{PROJECT}` (o `https://{ORG}.visualstudio.com/{PROJECT}`). Se extrae `{ORG}` y el nombre de `{PROJECT}`.
- **Jira** — `https://{SITE}.atlassian.net/jira/software/projects/{KEY}/...` (o cualquier URL del site). Se extrae `{SITE}` y la `{KEY}` del proyecto.

Validar que el `{ORG}`/`{SITE}` de la URL coincida con el que se configuró; si difiere, avisar al usuario antes de continuar.

**Dónde almacenar:**

1. **En el MCP, si encaja** — lo que forma parte de la conexión ya vive en la entrada del `mcp.json`/`.mcp.json`: en ADO, `{ORG}` es un argumento (`args`); en Jira, `JIRA_URL` (el site) está en `env`. Eso queda persistido con la propia integración.
2. **En la memoria persistente del proyecto, el resto** — el **proyecto por defecto** y la URL completa no caben limpiamente en el JSON del MCP (no admite comentarios). Guardarlos en la memoria persistente del proyecto del agente (p. ej. `CLAUDE.md` en Claude Code, reglas de proyecto en Cursor, o el archivo de memoria que el agente use). **Anexar sin borrar** lo previo.

Formato sugerido de la entrada a persistir:

```markdown
## ALM — Integraciones configuradas

### {SERVER_KEY}  (ADO | Jira)
- Agente(s): Cursor, Claude Code
- Organización/Site: {ORG | SITE}
- Proyecto por defecto: {PROJECT | KEY}
- URL: {URL_DEL_PROYECTO}
- Usuario: {EMAIL}
- Variable de token: {ADO_PAT_{ALIAS} | JIRA_TOKEN_{ALIAS}}  (secreto en Keychain/variable de usuario, no aquí)
```

> **Nunca** escribir el token ni ningún secreto en la memoria del proyecto; solo referencias no sensibles (org/site, proyecto, URL, correo, nombre de la variable).

## Límite de nombres

El límite de **60 caracteres** en `{SERVER_KEY}:{tool_name}` es de **Cursor**. Claude Code no lo impone, pero mantener claves cortas (`{SERVER_KEY}`) es buena práctica en ambos, así que se aplica el mismo algoritmo de nomenclatura para las dos.
