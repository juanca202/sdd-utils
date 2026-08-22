# Jira MCP — macOS + Cursor

Pasos para que Cursor (abierto desde Dock o Spotlight) reciba cada `JIRA_TOKEN_{ALIAS}` de forma persistente.

## Requisitos

- `uv`/`uvx` (o Docker, según la variante elegida)
- macOS con `zsh` (predeterminado)
- API token ya creado en Atlassian

---

## Por cada cuenta Jira

Repetir estos pasos una vez por cada par site+correo. Sustituir `{ALIAS}` por el alias calculado (ej. `MIEMPRESA_MARIA`).

### Paso 1 — Guardar el token en Keychain (usuario)

El agente **muestra solo este paso** al usuario y espera confirmación antes de continuar. **No** mostrar pasos de verificación.

> El API token de Jira se guarda **en crudo** (sin base64).

**Primera vez:**

```bash
security add-generic-password -a "$USER" -s "JIRA_TOKEN_{ALIAS}" \
  -w 'API_TOKEN_EN_CRUDO' -U
```

**Rotar token existente:**

```bash
security delete-generic-password -a "$USER" -s "JIRA_TOKEN_{ALIAS}" 2>/dev/null
security add-generic-password -a "$USER" -s "JIRA_TOKEN_{ALIAS}" \
  -w 'API_TOKEN_EN_CRUDO' -U
```

> Sustituir `API_TOKEN_EN_CRUDO` por el token tal como lo copió de Atlassian. El token **no** debe pegarse en el chat.

---

### Pasos 2 y 3 — Automáticos (agente)

Tras la confirmación del usuario, el agente **ejecuta** estos pasos sin pedir intervención adicional. Antes de continuar, el agente **verifica** internamente que el Keychain responde (ver «Verificación del agente»).

#### Paso 2 — Exportar en `~/.zshrc`

Añadir al final de `~/.zshrc` (si la línea no existe ya):

```bash
# Jira API token para Cursor MCP — {ALIAS}
export JIRA_TOKEN_{ALIAS}=$(security find-generic-password -a "$USER" -s "JIRA_TOKEN_{ALIAS}" -w 2>/dev/null)
```

Aplicar en la sesión actual:

```bash
source ~/.zshrc
```

#### Paso 3 — LaunchAgent (necesario para Cursor como app gráfica)

Cursor no hereda variables de `~/.zshrc` cuando se abre desde el Dock o Spotlight. El LaunchAgent inyecta cada variable en el entorno de usuario al iniciar sesión.

Crear `~/Library/LaunchAgents/setenv.{SERVER_KEY}.plist` (un archivo por cuenta):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>setenv.{SERVER_KEY}</string>
  <key>ProgramArguments</key>
  <array>
    <string>/bin/sh</string>
    <string>-c</string>
    <string>launchctl setenv JIRA_TOKEN_{ALIAS} "$(security find-generic-password -a $USER -s JIRA_TOKEN_{ALIAS} -w 2>/dev/null)"</string>
  </array>
  <key>RunAtLoad</key>
  <true/>
</dict>
</plist>
```

> Ejemplo: `{SERVER_KEY}` = `jira-emp-mar` → archivo `setenv.jira-emp-mar.plist`, label `setenv.jira-emp-mar`.

Activar sin reiniciar:

```bash
launchctl load ~/Library/LaunchAgents/setenv.{SERVER_KEY}.plist
launchctl setenv JIRA_TOKEN_{ALIAS} "$(security find-generic-password -a "$USER" -s "JIRA_TOKEN_{ALIAS}" -w)"
```

---

## Verificación del agente

El agente ejecuta estas comprobaciones **antes** de pedir reiniciar Cursor. **No** mostrar estos comandos al usuario salvo que falle algo y haga falta diagnosticar.

```bash
# Keychain accesible
security find-generic-password -a "$USER" -s "JIRA_TOKEN_{ALIAS}" -w 2>/dev/null | wc -c   # > 0

# Variable en shell
echo ${#JIRA_TOKEN_{ALIAS}}   # > 0

# LaunchAgent activo
launchctl getenv JIRA_TOKEN_{ALIAS} | wc -c   # > 0

# API Jira (Basic auth email:token)
VAR="JIRA_TOKEN_{ALIAS}"
TOKEN="${!VAR:-$(security find-generic-password -a "$USER" -s "$VAR" -w 2>/dev/null)}"
curl -s -o /dev/null -w "HTTP %{http_code}\n" \
  -u "{EMAIL}:$TOKEN" \
  "https://{SITE}.atlassian.net/rest/api/3/myself"
# Esperado: HTTP 200
```

> **Cuidado: HTTP 200 no garantiza éxito.** Atlassian puede responder `200` con una **página de inicio de sesión** (HTML) si el token es inválido o expiró. Verificar que la respuesta sea **JSON real** (el objeto de usuario de `myself`, con `accountId`/`emailAddress`), no solo el código 200. Mapear `401`/`403` como **token inválido / sin permisos** y `404` como **site inexistente / incorrecto**.

Si alguna comprobación falla, el agente diagnostica y corrige antes de continuar.

---

## Reiniciar Cursor

1. Cerrar con **Cmd+Q** (no solo la ventana).
2. Reabrir Cursor.
3. **Settings → MCP** → cada servidor `{SERVER_KEY}` debe aparecer **Connected**.

---

## Desinstalar / rotar token

```bash
# Descargar y eliminar el LaunchAgent de esa cuenta
launchctl unload ~/Library/LaunchAgents/setenv.{SERVER_KEY}.plist 2>/dev/null
rm ~/Library/LaunchAgents/setenv.{SERVER_KEY}.plist

# Eliminar del Keychain
security delete-generic-password -a "$USER" -s "JIRA_TOKEN_{ALIAS}"

# Quitar la línea correspondiente de ~/.zshrc manualmente
```

Revocar el token en: `https://id.atlassian.com/manage-profile/security/api-tokens`.
