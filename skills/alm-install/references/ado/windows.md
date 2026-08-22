# Azure DevOps MCP — Windows + Cursor

Pasos para que Cursor reciba cada `ADO_PAT_{ALIAS}` de forma persistente tras reiniciar el equipo.

## Requisitos

- Node.js 20+
- Windows 10/11
- PowerShell 5.1+ o PowerShell 7+
- PAT ya creado en Azure DevOps

---

## Por cada cuenta ADO

Repetir estos pasos una vez por cada par organización+correo. Sustituir `{ALIAS}` por el alias calculado (ej. `FABRIKAM_MARIA`).

### Paso 1 — Codificar y guardar variable de usuario (usuario)

El agente **muestra solo este paso** al usuario y espera confirmación antes de continuar. **No** mostrar comandos de codificación aparte ni pasos de verificación.

En **PowerShell**, sustituir `PAT_EN_CRUDO` por el token tal como lo copió de Azure DevOps:

```powershell
$base64Pat = [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes(":PAT_EN_CRUDO"))
[System.Environment]::SetEnvironmentVariable("ADO_PAT_{ALIAS}", $base64Pat, "User")
```

> El PAT **no** debe pegarse en el chat.

---

### Paso 2 — Perfil de PowerShell (agente, automático)

Tras la confirmación del usuario, el agente **ejecuta** este paso sin pedir intervención adicional. Antes de continuar, el agente **verifica** internamente que la variable existe (ver sección «Verificación del agente»).

Para que nuevas ventanas de PowerShell también tengan la variable disponible:

```powershell
# Crear perfil si no existe
if (!(Test-Path $PROFILE)) { New-Item -Path $PROFILE -ItemType File -Force }
```

Añadir a `$PROFILE` (si la línea no existe ya):

```powershell
$env:ADO_PAT_{ALIAS} = [Environment]::GetEnvironmentVariable("ADO_PAT_{ALIAS}", "User")
```

> Las variables de usuario en Windows persisten en el registro y son heredadas por Cursor al abrirse después de cerrar y reabrir sesión.

---

## Verificación del agente

El agente ejecuta estas comprobaciones **antes** de pedir reiniciar Cursor. **No** mostrar estos comandos al usuario salvo que falle algo y haga falta diagnosticar.

```powershell
# Variable de usuario persistente
[Environment]::GetEnvironmentVariable("ADO_PAT_{ALIAS}", "User").Length  # > 0

# API Azure DevOps
$pat = [Environment]::GetEnvironmentVariable("ADO_PAT_{ALIAS}", "User")
$headers = @{ Authorization = "Basic $pat" }
(Invoke-WebRequest -Uri "https://dev.azure.com/{ORG}/_apis/projects?api-version=7.1&`$top=1" -Headers $headers).StatusCode
# Esperado: 200
```

> **Cuidado: HTTP 200 no garantiza éxito.** Azure DevOps puede responder `200` con una **página de inicio de sesión** (HTML) si el PAT es inválido o expiró. Verificar que la respuesta sea **JSON real** (que su contenido contenga `"value"` o `"count"`), no solo el código 200. Mapear `401`/`203` como **PAT inválido** y `404` como **organización inexistente / incorrecta**.

Si alguna comprobación falla, el agente diagnostica y corrige antes de continuar.

---

## Reiniciar Cursor

1. Cerrar Cursor por completo (Archivo → Salir o desde bandeja).
2. Cerrar sesión de Windows y volver a entrar (o reiniciar el equipo) para que Cursor herede las variables de usuario actualizadas.
3. Reabrir Cursor.
4. **Settings → MCP** → cada servidor `{SERVER_KEY}` debe aparecer **Connected** y sin aviso de *naming issues*.

> Si no se quiere cerrar sesión, lanzar Cursor desde una ventana de PowerShell donde se haya seteado `$env:ADO_PAT_{ALIAS}` manualmente (solo válido para esa sesión).

---

## Desinstalar / rotar token

```powershell
# Eliminar la variable de usuario
[Environment]::SetEnvironmentVariable("ADO_PAT_{ALIAS}", $null, "User")

# Quitar línea de $PROFILE si se añadió
```

Revocar el PAT en: `https://dev.azure.com/{ORG}/_usersSettings/tokens` (con el usuario de esa cuenta).
