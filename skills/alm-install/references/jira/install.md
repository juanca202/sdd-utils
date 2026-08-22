# Jira Install — MCP de Jira (Claude Code / Cursor)

Configura el **servidor MCP local** `mcp-atlassian` (ejecutado con `uvx mcp-atlassian`). Autenticación por **API token** de Atlassian (Basic auth `email:token`). **No** uses el servidor remoto de Atlassian con OAuth: se mantiene la misma filosofía local que el flujo de ADO.

Cada entrada en el archivo de config está ligada a un par **site (dominio Atlassian) + correo de usuario**. Esto permite tener varias cuentas activas al mismo tiempo, cada una con su propio API token.

> **Dónde se escribe la entrada** (`.cursor/mcp.json` en Cursor, `.mcp.json`/`~/.claude.json` en Claude Code) y **la sintaxis de la variable de token** (`{TOKEN_REF}`) dependen del agente: ver [../agents.md](../agents.md).

## Flujo del agente

1. **Preguntar el site** de Jira (dominio Atlassian, ej. `miempresa` de `https://miempresa.atlassian.net`). Aceptar el subdominio o la URL completa; normalizar a `https://{SITE}.atlassian.net`.
2. **Preguntar el correo** del usuario Atlassian asociado a ese site (ej. `juan@empresa.com`). Es el `JIRA_USERNAME` (parte del par Basic auth) y se usa para nombrar la variable de token y la entrada del servidor de forma única.
3. **Detectar SO**: macOS → [macos.md](macos.md); Windows → [windows.md](windows.md).
   - **Precondición `uvx`:** como `mcp.json` usa `command: "uvx"`, verificar que `uv`/`uvx` estén disponibles (`uvx --version`). Si faltan, indicar al usuario que instale `uv` (`https://docs.astral.sh/uv/`) antes de continuar, ya que el servidor MCP fallaría al reiniciar Cursor. (Alternativa: usar Docker con `command: "docker"`; ver «Variante Docker».)
4. **Calcular identificadores** de la cuenta:
   - **Alias env** `{ALIAS}`: `{SITE}_{PARTE_CORREO}` en mayúsculas (ej. `MIEMPRESA_JUAN`). Se usa en `JIRA_TOKEN_{ALIAS}` y Keychain.
   - **Clave servidor MCP** `{SERVER_KEY}`: nombre **corto** para `mcp.json` (ver «Nomenclatura»). **No** usar el alias completo como clave del servidor.
5. **Indicar al usuario** que cree el API token en Atlassian (con ese correo) antes de continuar.
6. **Leer el archivo de config del agente** (ver rutas en [../agents.md](../agents.md)) si ya existe, para agregar la nueva entrada sin borrar las configuraciones previas. **Si el archivo existe pero NO es JSON válido** (corrupto o editado a mano), **no sobrescribir a ciegas**: informar al usuario, respaldarlo (ej. `mcp.json.bak`) y pedir confirmación antes de regenerarlo.
7. **Escribir o actualizar** el archivo de config del agente con la nueva entrada (plantilla abajo), usando su `{TOKEN_REF}` según [../agents.md](../agents.md).
8. **Mostrar al usuario únicamente** el Paso 1 de la guía del SO (guardar el token: Keychain en macOS, variable de usuario en Windows). El agente **no** debe pedir ni almacenar el token en el chat. **Esperar confirmación** de que el usuario completó ese paso.
9. **Automáticamente**, tras la confirmación, ejecutar los pasos del agente de la guía del SO:
   - **macOS:** export en `~/.zshrc` + LaunchAgent (Dock/Spotlight).
   - **Windows:** línea en `$PROFILE` de PowerShell.
   No mostrar estos pasos al usuario salvo que falle algo y haga falta intervención manual.
10. **Verificar internamente** con los comandos de «Verificación del agente» en la guía del SO. El agente ejecuta las comprobaciones; **no** mostrarlas al usuario. Solo informar el resultado (éxito o error con diagnóstico).
11. **Pedir la URL del proyecto** (ej. `https://{SITE}.atlassian.net/jira/software/projects/{KEY}/...`). Extraer `{SITE}` (validar que coincida con el configurado) y la `{KEY}` del proyecto. Persistir el contexto: `{SITE}` ya está en `JIRA_URL` del `env`; el **proyecto por defecto** (`{KEY}`), la URL, el correo y el nombre de la variable van a la **memoria persistente del proyecto** (p. ej. `CLAUDE.md` en Claude Code, reglas de proyecto en Cursor; anexar sin borrar, **sin** secretos). Ver [../agents.md](../agents.md) → «Contexto del proyecto (URL)».
12. Pedir **reinicio del agente** tras configurar variables: Cursor → cierre completo (Cmd+Q); Claude Code → reiniciar la sesión. (En Claude Code, la primera vez con un `.mcp.json` de proyecto se debe **aprobar** el servidor.)

## Crear API token (usuario)

El usuario debe hacer esto manualmente **con la cuenta de correo indicada**:

1. Iniciar sesión en Atlassian con ese correo.
2. Ir a `https://id.atlassian.com/manage-profile/security/api-tokens`
3. **Create API token** → darle un nombre (ej. `Cursor MCP Jira`).
4. Copiar el token **en crudo** (solo se muestra una vez).

> A diferencia de ADO, el API token de Jira se guarda **en crudo** (sin base64). El propio MCP arma el `email:token` para el Basic auth.

## Nomenclatura

Cursor impone un límite de **60 caracteres** en `{SERVER_KEY}:{tool_name}` (incluido el separador `:`). Las herramientas de `mcp-atlassian` son moderadas (ej. `jira_get_issue`, `jira_search`), pero por seguridad mantener `{SERVER_KEY}` **≤ 20 caracteres**.

### Alias de entorno (`{ALIAS}`)

Usado en `JIRA_TOKEN_{ALIAS}`, Keychain y variables de Windows. Puede ser largo.

| Correo               | Site       | Alias env          | Variable de env          |
| -------------------- | ---------- | ------------------ | ------------------------ |
| maria@empresa.com    | miempresa  | MIEMPRESA_MARIA    | JIRA_TOKEN_MIEMPRESA_MARIA |
| carlos@cliente.com   | clienteorg | CLIENTEORG_CARLOS  | JIRA_TOKEN_CLIENTEORG_CARLOS |

Reglas del alias env:
- `{SITE en mayúsculas}_{PARTE_ANTES_DE_@ en mayúsculas}`
- Reemplazar `-`, `.` y otros caracteres no alfanuméricos por `_`.

### Clave servidor MCP (`{SERVER_KEY}`)

Usada **solo** como clave en `mcp.json`. Debe ser corta y única.

| Correo               | Site       | SERVER_KEY (≤ 20)  |
| -------------------- | ---------- | ------------------ |
| maria@empresa.com    | miempresa  | `jira-emp-mar`     |
| carlos@cliente.com   | clienteorg | `jira-cli-car`     |

Algoritmo para calcular `{SERVER_KEY}`:

1. Prefijo fijo `jira-` (5 caracteres).
2. `{site}` = site en minúsculas, solo alfanuméricos, truncado para que quepa.
3. `-` separador.
4. `{user}` = parte local del correo en minúsculas, solo alfanuméricos, truncada para que quepa.
5. **Validar:** `len(SERVER_KEY) ≤ 20`. Si no cabe, acortar `{user}` primero, luego `{site}`.
6. **Colisión (otra cuenta):** si la clave ya existe en `mcp.json` para **otra** cuenta (site o correo distintos), añadir sufijo numérico (`jira-emp-mar2`).
7. **Re-ejecución (misma cuenta):** si la entrada corresponde a la **misma** cuenta (site **y** correo idénticos), **actualizar/sobrescribir** la entrada existente — **no duplicar**. Tratar el token como **rotación**: regenerar su valor en Keychain (macOS) o variable de usuario (Windows) reutilizando el mismo `{SERVER_KEY}` y `JIRA_TOKEN_{ALIAS}`.

## Plantilla de entrada MCP (multi-cuenta)

Cada cuenta es una entrada independiente. Cuando se agrega una segunda cuenta, **conservar** las entradas previas. `{TOKEN_REF}` se resuelve según el agente (`${env:JIRA_TOKEN_{ALIAS}}` en Cursor, `${JIRA_TOKEN_{ALIAS}}` en Claude Code — ver [../agents.md](../agents.md)):

```json
{
  "mcpServers": {
    "{SERVER_KEY}": {
      "command": "uvx",
      "args": ["mcp-atlassian"],
      "env": {
        "JIRA_URL": "https://{SITE}.atlassian.net",
        "JIRA_USERNAME": "{EMAIL}",
        "JIRA_API_TOKEN": "{TOKEN_REF}"
      }
    }
  }
}
```

**Ejemplo con dos cuentas (sintaxis de Cursor):**

```json
{
  "mcpServers": {
    "jira-emp-mar": {
      "command": "uvx",
      "args": ["mcp-atlassian"],
      "env": {
        "JIRA_URL": "https://miempresa.atlassian.net",
        "JIRA_USERNAME": "maria@empresa.com",
        "JIRA_API_TOKEN": "${env:JIRA_TOKEN_MIEMPRESA_MARIA}"
      }
    },
    "jira-cli-car": {
      "command": "uvx",
      "args": ["mcp-atlassian"],
      "env": {
        "JIRA_URL": "https://clienteorg.atlassian.net",
        "JIRA_USERNAME": "carlos@cliente.com",
        "JIRA_API_TOKEN": "${env:JIRA_TOKEN_CLIENTEORG_CARLOS}"
      }
    }
  }
}
```

- `{SERVER_KEY}` ≤ 20 caracteres; `{ALIAS}` puede ser largo (solo env/Keychain).
- No commitear secretos inline; usar siempre la referencia a variable (`{TOKEN_REF}`).
- En **Claude Code** la misma entrada va en `.mcp.json` con `"JIRA_API_TOKEN": "${JIRA_TOKEN_MIEMPRESA_MARIA}"` (sin `env:`).
- Cada servidor MCP aparecerá por separado (Cursor: **Settings → MCP**; Claude Code: `/mcp`).

### Variante Docker

Si el usuario prefiere Docker en lugar de `uvx`:

```json
{
  "command": "docker",
  "args": ["run", "-i", "--rm",
    "-e", "JIRA_URL", "-e", "JIRA_USERNAME", "-e", "JIRA_API_TOKEN",
    "ghcr.io/sooperset/mcp-atlassian:latest"],
  "env": {
    "JIRA_URL": "https://{SITE}.atlassian.net",
    "JIRA_USERNAME": "{EMAIL}",
    "JIRA_API_TOKEN": "${env:JIRA_TOKEN_{ALIAS}}"
  }
}
```

Precondición: Docker instalado y corriendo.

## Verificación

La verificación previa al reinicio la ejecuta el agente (ver «Verificación del agente» en [macos.md](macos.md) o [windows.md](windows.md)). **No** mostrar esos comandos al usuario.

Tras reiniciar el agente, comprobar que el servidor `{SERVER_KEY}` aparece conectado: Cursor → **Settings → MCP** en verde; Claude Code → `/mcp` (o `claude mcp list`).

**Prueba funcional en chat:** listar proyectos o buscar issues del site usando el servidor `{SERVER_KEY}` (ej. una JQL simple).

## Errores frecuentes

| Síntoma                                     | Causa                                          | Acción                                          |
| ------------------------------------------- | ---------------------------------------------- | ----------------------------------------------- |
| `uvx: command not found`                    | `uv` no instalado                              | Instalar `uv` o usar la variante Docker          |
| HTTP 401 / 403                              | API token inválido/expirado o correo incorrecto | Rotar token y verificar `JIRA_USERNAME`          |
| HTTP 200 con HTML de login                  | Token inválido; Atlassian devuelve página web   | Recrear el API token                            |
| HTTP 404 en `myself`                        | `JIRA_URL` (site) incorrecto                    | Corregir el dominio `{SITE}.atlassian.net`       |
| Variable no visible en Cursor               | `JIRA_TOKEN_{ALIAS}` no exportada al entorno    | Ver guía del SO (LaunchAgent / variable usuario) |
| Segunda cuenta no aparece en MCP            | Entrada borrada al actualizar mcp.json          | Asegurarse de fusionar, no reemplazar            |

## Referencias por plataforma

- **macOS:** almacenamiento en Keychain + LaunchAgent + `.zshrc` → [macos.md](macos.md)
- **Windows:** variable de usuario persistente → [windows.md](windows.md)

> En ambas plataformas, repetir el proceso de almacenamiento por cada variable `JIRA_TOKEN_{ALIAS}` que se agregue.

## Seguridad

- Nunca pegar el API token en chat, commits ni `mcp.json`.
- Cada token está ligado a un correo Atlassian específico; solo da acceso a los sites donde ese usuario tiene permisos.
- Si se expone un token, revocarlo en `https://id.atlassian.com/manage-profile/security/api-tokens` y regenerarlo.
- Al rotar un token, actualizar únicamente la variable `JIRA_TOKEN_{ALIAS}` afectada; las demás cuentas no se ven afectadas.
