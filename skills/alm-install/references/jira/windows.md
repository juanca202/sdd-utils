# Jira MCP — Windows + Cursor

Pasos para que Cursor reciba cada `JIRA_TOKEN_{ALIAS}` de forma persistente tras reiniciar el equipo.

## Requisitos

- `uv`/`uvx` (o Docker, según la variante elegida)
- Windows 10/11
- PowerShell 5.1+ o PowerShell 7+
- API token ya creado en Atlassian

---

## Por cada cuenta Jira

Repetir estos pasos una vez por cada par site+correo. Sustituir `{ALIAS}` por el alias calculado (ej. `MIEMPRESA_MARIA`).

### Paso 1 — Guardar variable de usuario (usuario)

El agente **muestra solo este paso** al usuario y espera confirmación antes de continuar. **No** mostrar pasos de verificación.

> El API token de Jira se guarda **en crudo** (sin base64).

En **PowerShell**, sustituir `API_TOKEN_EN_CRUDO` por el token tal como lo copió de Atlassian:

```powershell
[System.Environment]::SetEnvironmentVariable("JIRA_TOKEN_{ALIAS}", "API_TOKEN_EN_CRUDO", "User")
```

> El token **no** debe pegarse en el chat.

---

### Paso 2 — Perfil de PowerShell (agente, automático)

Tras la confirmación del usuario, el agente **ejecuta** este paso sin pedir intervención adicional. Antes de continuar, el agente **verifica** internamente que la variable existe (ver «Verificación del agente»).

Para que nuevas ventanas de PowerShell también tengan la variable disponible:

```powershell
# Crear perfil si no existe
if (!(Test-Path $PROFILE)) { New-Item -Path $PROFILE -ItemType File -Force }
```

Añadir a `$PROFILE` (si la línea no existe ya):

```powershell
$env:JIRA_TOKEN_{ALIAS} = [Environment]::GetEnvironmentVariable("JIRA_TOKEN_{ALIAS}", "User")
```

> Las variables de usuario en Windows persisten en el registro y son heredadas por Cursor al abrirse después de cerrar y reabrir sesión.

---

## Verificación del agente

El agente ejecuta estas comprobaciones **antes** de pedir reiniciar Cursor. **No** mostrar estos comandos al usuario salvo que falle algo y haga falta diagnosticar.

```powershell
# Variable de usuario persistente
[Environment]::GetEnvironmentVariable("JIRA_TOKEN_{ALIAS}", "User").Length  # > 0

# API Jira (Basic auth email:token)
$token = [Environment]::GetEnvironmentVariable("JIRA_TOKEN_{ALIAS}", "User")
$pair  = [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes("{EMAIL}:$token"))
$headers = @{ Authorization = "Basic $pair" }
(Invoke-WebRequest -Uri "https://{SITE}.atlassian.net/rest/api/3/myself" -Headers $headers).StatusCode
# Esperado: 200
```

> **Cuidado: HTTP 200 no garantiza éxito.** Atlassian puede responder `200` con una **página de inicio de sesión** (HTML) si el token es inválido o expiró. Verificar que la respuesta sea **JSON real** (el objeto de usuario de `myself`, con `accountId`/`emailAddress`), no solo el código 200. Mapear `401`/`403` como **token inválido / sin permisos** y `404` como **site inexistente / incorrecto**.

Si alguna comprobación falla, el agente diagnostica y corrige antes de continuar.

---

## Reiniciar Cursor

1. Cerrar Cursor por completo (Archivo → Salir o desde bandeja).
2. Cerrar sesión de Windows y volver a entrar (o reiniciar el equipo) para que Cursor herede las variables de usuario actualizadas.
3. Reabrir Cursor.
4. **Settings → MCP** → cada servidor `{SERVER_KEY}` debe aparecer **Connected**.

> Si no se quiere cerrar sesión, lanzar Cursor desde una ventana de PowerShell donde se haya seteado `$env:JIRA_TOKEN_{ALIAS}` manualmente (solo válido para esa sesión).

---

## Desinstalar / rotar token

```powershell
# Eliminar la variable de usuario
[Environment]::SetEnvironmentVariable("JIRA_TOKEN_{ALIAS}", $null, "User")

# Quitar línea de $PROFILE si se añadió
```

Revocar el token en: `https://id.atlassian.com/manage-profile/security/api-tokens`.
