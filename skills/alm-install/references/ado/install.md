# ADO Install — MCP de Azure DevOps (Claude Code / Cursor)

Configura el **servidor MCP local** (`npx @azure-devops/mcp`). **No** uses el servidor remoto (`mcp.dev.azure.com`): Cursor no soporta su OAuth con Entra ID.

Cada entrada en el archivo de config está ligada a un par **organización + correo de usuario**. Esto permite tener varias cuentas activas al mismo tiempo, cada una con su propio PAT.

> **Dónde se escribe la entrada** (`.cursor/mcp.json` en Cursor, `.mcp.json`/`~/.claude.json` en Claude Code) y **la sintaxis de la variable de token** (`{TOKEN_REF}`) dependen del agente: ver [../agents.md](../agents.md). Este flujo describe la lógica de ADO; el agente concreto define la ruta y la sintaxis.

## Flujo del agente

1. **Preguntar la organización** de Azure DevOps (ej. `Fabrikam`). Solo el nombre, sin URL completa.
2. **Preguntar el correo** del usuario ADO asociado a esa organización (ej. `juan@empresa.com`). Se usa para nombrar la variable de entorno y la entrada del servidor de forma única.
3. **Detectar SO**: macOS → [macos.md](macos.md); Windows → [windows.md](windows.md).
   - **Precondición Node/npx:** como `mcp.json` usa `command: "npx"`, verificar que `node` y `npx` estén disponibles (`node -v`, `npx -v`; idealmente Node **20+**). Si faltan, avisar al usuario que debe instalarlos antes de continuar, ya que el servidor MCP fallaría al reiniciar Cursor.
4. **Calcular identificadores** de la cuenta:
   - **Alias env** `{ALIAS}`: `{ORG}_{PARTE_CORREO}` en mayúsculas (ej. `BAYTEQDEV_JUANCA`). Se usa en `ADO_PAT_{ALIAS}` y Keychain.
   - **Clave servidor MCP** `{SERVER_KEY}`: nombre **corto** para `mcp.json` (ver sección «Límite de 60 caracteres en Cursor»). **No** usar el alias completo como clave del servidor.
5. **Indicar al usuario** que cree el PAT en Azure DevOps (con ese usuario) antes de continuar.
6. **Leer el archivo de config del agente** (ver rutas en [../agents.md](../agents.md)) si ya existe, para agregar la nueva entrada sin borrar las configuraciones previas. **Si el archivo existe pero NO es JSON válido** (corrupto o editado a mano), **no sobrescribir a ciegas**: informar al usuario, respaldarlo (ej. `mcp.json.bak`) y pedir confirmación antes de regenerarlo.
7. **Escribir o actualizar** el archivo de config del agente con la nueva entrada (plantilla abajo), usando su `{TOKEN_REF}` según [../agents.md](../agents.md). En Cursor, validar que `{SERVER_KEY}` tenga ≤ 17 caracteres antes de guardar (límite de 60 caracteres de Cursor).
8. **Mostrar al usuario únicamente** el Paso 1 de la guía del SO (codificar y guardar: Keychain en macOS, variable de usuario en Windows). El agente **no** debe pedir ni almacenar el PAT en el chat. **Esperar confirmación** de que el usuario completó ese paso.
9. **Automáticamente**, tras la confirmación, ejecutar los pasos del agente de la guía del SO:
   - **macOS:** export en `~/.zshrc` + LaunchAgent (Dock/Spotlight).
   - **Windows:** línea en `$PROFILE` de PowerShell.
   No mostrar estos pasos al usuario salvo que falle algo y haga falta intervención manual.
10. **Verificar internamente** con los comandos de «Verificación del agente» en la guía del SO. El agente ejecuta las comprobaciones; **no** mostrarlas al usuario. Solo informar el resultado (éxito o error con diagnóstico).
11. **Pedir la URL del proyecto** (ej. `https://dev.azure.com/{ORG}/{PROJECT}`). Extraer `{ORG}` (validar que coincida con el configurado) y `{PROJECT}`. Persistir el contexto: `{ORG}` ya está en `args` del `mcp.json`; el **proyecto por defecto**, la URL, el correo y el nombre de la variable van a la **memoria persistente del proyecto** (p. ej. `CLAUDE.md` en Claude Code, reglas de proyecto en Cursor; anexar sin borrar, **sin** secretos). Ver [../agents.md](../agents.md) → «Contexto del proyecto (URL)».
12. Pedir **reinicio del agente** tras configurar variables: Cursor → cierre completo (Cmd+Q); Claude Code → reiniciar la sesión. (En Claude Code, la primera vez con un `.mcp.json` de proyecto se debe **aprobar** el servidor.)

## Crear PAT (usuario)

El usuario debe hacer esto manualmente **con la cuenta de correo indicada**:

1. Iniciar sesión en Azure DevOps con ese correo.
2. Ir a `https://dev.azure.com/{ORG}/_usersSettings/tokens`
3. **New Token** → scopes mínimos según necesidad:
   - Work Items → Read (Write si creará/editará)
   - Code → Read
   - Build → Read (opcional)
4. Copiar el token **en crudo** (solo se muestra una vez).

## Codificación obligatoria

`@azure-devops/mcp` con `--authentication pat` exige `PERSONAL_ACCESS_TOKEN` como **base64 de `:PAT`** (no el PAT crudo):

```bash
# macOS / Linux / Git Bash
echo -n ":PAT_EN_CRUDO" | base64
```

```powershell
# Windows PowerShell
[Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes(":PAT_EN_CRUDO"))
```

El valor base64 resultante es lo que se guarda en `ADO_PAT_{ALIAS}`.

## Nomenclatura

Cursor impone un límite de **60 caracteres** en `{SERVER_KEY}:{tool_name}` (incluido el separador `:`). Si se supera, filtra herramientas y muestra *«Some tools have naming issues and may be filtered out»*. La herramienta más larga de `@azure-devops/mcp` mide **42 caracteres** (`repo_list_pull_requests_by_repo_or_project`), así que `{SERVER_KEY}` debe tener **≤ 17 caracteres** (`60 − 1 separador − 42 = 17`).

### Alias de entorno (`{ALIAS}`)

Usado en `ADO_PAT_{ALIAS}`, Keychain y variables de Windows. Puede ser largo.

| Correo                         | Org        | Alias env                   | Variable de env                       |
| ------------------------------ | ---------- | --------------------------- | ------------------------------------- |
| maria@fabrikam.com             | Fabrikam   | FABRIKAM_MARIA              | ADO_PAT_FABRIKAM_MARIA                |
| carlos@cliente.com             | ClienteOrg | CLIENTEORG_CARLOS           | ADO_PAT_CLIENTEORG_CARLOS             |
| juanca.altamirano@bayteq.com   | BayteqDev  | BAYTEQDEV_JUANCA_ALTAMIRANO | ADO_PAT_BAYTEQDEV_JUANCA_ALTAMIRANO   |

Reglas del alias env:
- `{ORG en mayúsculas}_{PARTE_ANTES_DE_@ en mayúsculas}`
- Reemplazar `-`, `.` y otros caracteres no alfanuméricos por `_`.

### Clave servidor MCP (`{SERVER_KEY}`)

Usada **solo** como clave en `mcp.json`. Debe ser corta y única.

| Correo                         | Org        | SERVER_KEY (≤ 17) |
| ------------------------------ | ---------- | ----------------- |
| maria@fabrikam.com             | Fabrikam   | `ado-fab-mar`     |
| carlos@cliente.com             | ClienteOrg | `ado-cli-car`     |
| juanca.altamirano@bayteq.com   | BayteqDev  | `ado-bay-jua`     |

Algoritmo para calcular `{SERVER_KEY}`:

1. Prefijo fijo `ado-` (4 caracteres).
2. `{org}` = organización en minúsculas, solo alfanuméricos, truncada para que quepa.
3. `-` separador.
4. `{user}` = parte local del correo en minúsculas, solo alfanuméricos, truncada para que quepa.
5. **Validar:** `len(SERVER_KEY) ≤ 17`. Si no cabe, acortar `{user}` primero, luego `{org}`.
6. **Colisión (otra cuenta):** si la clave ya existe en `mcp.json` para **otra** cuenta (org o correo distintos), añadir sufijo numérico (`ado-bay-jua2`) recortando `{user}` si hace falta.
7. **Re-ejecución (misma cuenta):** si la entrada corresponde a la **misma** cuenta (org **y** correo idénticos), **actualizar/sobrescribir** la entrada existente en lugar de crear una nueva — **no duplicar** la entrada. Tratar el token como **rotación**: regenerar su valor en Keychain (macOS) o variable de usuario (Windows) reutilizando el mismo `{SERVER_KEY}` y `ADO_PAT_{ALIAS}`.

> **Migración:** si ya existe una clave larga (ej. `ado-bayteq-juanca-altamirano`), renombrar la entrada en `mcp.json` a la clave corta. La variable `ADO_PAT_{ALIAS}` no cambia.

## Plantilla de entrada MCP (multi-cuenta)

Cada cuenta es una entrada independiente. Cuando se agrega una segunda cuenta, **conservar** las entradas previas. `{TOKEN_REF}` se resuelve según el agente (`${env:ADO_PAT_{ALIAS}}` en Cursor, `${ADO_PAT_{ALIAS}}` en Claude Code — ver [../agents.md](../agents.md)):

```json
{
  "mcpServers": {
    "{SERVER_KEY}": {
      "command": "npx",
      "args": ["-y", "@azure-devops/mcp", "{ORG}", "--authentication", "pat"],
      "env": {
        "PERSONAL_ACCESS_TOKEN": "{TOKEN_REF}"
      }
    }
  }
}
```

**Ejemplo con dos cuentas (sintaxis de Cursor):**

```json
{
  "mcpServers": {
    "ado-fab-mar": {
      "command": "npx",
      "args": ["-y", "@azure-devops/mcp", "Fabrikam", "--authentication", "pat"],
      "env": {
        "PERSONAL_ACCESS_TOKEN": "${env:ADO_PAT_FABRIKAM_MARIA}"
      }
    },
    "ado-cli-car": {
      "command": "npx",
      "args": ["-y", "@azure-devops/mcp", "ClienteOrg", "--authentication", "pat"],
      "env": {
        "PERSONAL_ACCESS_TOKEN": "${env:ADO_PAT_CLIENTEORG_CARLOS}"
      }
    }
  }
}
```

> En **Claude Code** la misma entrada va en `.mcp.json` con `"PERSONAL_ACCESS_TOKEN": "${ADO_PAT_FABRIKAM_MARIA}"` (sin `env:`).

- `{SERVER_KEY}` ≤ 17 caracteres en Cursor; `{ALIAS}` puede ser largo (solo env/Keychain).

- No commitear secretos inline; usar siempre `${env:ADO_PAT_{ALIAS}}`.
- No ejecutar remoto y local a la vez.
- Cada servidor MCP aparecerá por separado en **Settings → MCP** de Cursor.

## Verificación

La verificación previa al reinicio la ejecuta el agente (ver «Verificación del agente» en [macos.md](macos.md) o [windows.md](windows.md)). **No** mostrar esos comandos al usuario.

Tras reiniciar el agente, comprobar que el servidor `{SERVER_KEY}` aparece conectado: Cursor → **Settings → MCP** en verde y **sin** aviso de *naming issues*; Claude Code → `/mcp` (o `claude mcp list`) muestra el servidor conectado.

**Prueba funcional en chat:** listar proyectos o work items de `{ORG}` usando el servidor `{SERVER_KEY}`.

## Errores frecuentes

| Síntoma                                     | Causa                                     | Acción                                          |
| ------------------------------------------- | ----------------------------------------- | ----------------------------------------------- |
| *Some tools have naming issues…*            | `{SERVER_KEY}` demasiado largo (> 19)     | Renombrar clave en `mcp.json` según algoritmo   |
| `PERSONAL_ACCESS_TOKEN is not set or empty` | Variable `ADO_PAT_{ALIAS}` no visible     | Verificar nombre exacto del alias y reiniciar   |
| HTTP 302 / sign-in                          | PAT crudo en lugar de base64              | Recodificar y volver a guardar                  |
| MCP remoto falla                            | Cursor + OAuth remoto                     | Usar solo MCP local                             |
| Funciona en terminal, no en Cursor          | App abierta sin env de usuario            | Ver guía del SO                                 |
| Segunda cuenta no aparece en MCP            | Entrada borrada al actualizar mcp.json    | Asegurarse de fusionar, no reemplazar           |

## Referencias por plataforma

- **macOS:** almacenamiento en Keychain + LaunchAgent + `.zshrc` → [macos.md](macos.md)
- **Windows:** variable de usuario persistente → [windows.md](windows.md)

> En ambas plataformas, repetir el proceso de almacenamiento por cada variable `ADO_PAT_{ALIAS}` que se agregue.

## Seguridad

- Nunca pegar el PAT en chat, commits ni `mcp.json`.
- Cada PAT está ligado a un correo específico; un PAT de una cuenta **no** funciona en otra organización distinta donde ese usuario no tiene acceso.
- Si se expone un PAT, revocar en `https://dev.azure.com/{ORG}/_usersSettings/tokens` con el usuario correspondiente y regenerar solo ese token.
- Al rotar un PAT, actualizar únicamente la variable `ADO_PAT_{ALIAS}` afectada; las demás cuentas no se ven afectadas.
