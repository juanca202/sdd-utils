# Proyecto nuevo Angular (desde plantilla upstream)

## Alcance

Referencia ejecutable para **project-create** con `stack_id = angular`. Describe cómo crear un proyecto **Angular** nuevo para el usuario **a partir del proyecto base** del equipo, dentro de un repositorio git **ya existente**.

- **Aplica:** usuario pide proyecto Angular nuevo desde plantilla.
- **No aplica:** mantener/publicar el repo del proyecto base; ese flujo es distinto.

## Proyecto base (plantilla)

| Concepto | Valor |
| -------- | ----- |
| URL del repo | **Preguntar al usuario; no asumir ninguna URL** |
| Nombre del remote | `upstream` |
| Rama a integrar | `upstream/main` |
| Mensaje de merge | `Merge <repo-name> template` — inferir `<repo-name>` del segmento final de la URL sin `.git` |

---

## Requisitos previos

1. **Git:** el directorio de trabajo debe ser la **raíz de un repo** con `.git`. Si no: `git init` o clonar antes de seguir.
2. **Herramientas:** `git` y **Node/npm** en PATH para `npm install` y `npm start`.
3. **Red:** acceso al host de `templateUrl` para `git fetch` del remoto de la plantilla.

---

## Parámetros (obligatorios salvo `targetPath`)

| Parámetro     | Obligatorio | Descripción |
| ------------- | ----------- | ----------- |
| `templateUrl` | Sí          | URL git del repo plantilla (ej. `https://github.com/org/my-base.git`). |
| `projectName` | Sí          | Nombre legible (ej. `mi-app`). |
| `projectId`   | Sí          | Identificador **kebab-case** (ej. `mi-app`). |
| `targetPath`  | No          | Raíz del repo; si vacío: directorio actual. |

### Validación (aplicar antes de editar)

- **`templateUrl`:** no vacío; debe ser una URL git válida.
- **`projectName`:** no vacío; máx. 64 caracteres; regex `^[a-zA-Z0-9][a-zA-Z0-9._-]*$`.
- **`projectId`:** no vacío; máx. 128 caracteres; regex `^[a-z][a-z0-9]*(-[a-z0-9]+)*$` (kebab-case estricto).

### Si la validación falla

**Parar** sustituciones masivas; pedir valores corregidos al usuario.

---

## Orden de ejecución (no alterar)

1. [Instalar remote y merge](#1-remote-fetch-y-merge)
2. [Personalización `projectName` / `projectId`](#2-personalización-projectname-y-projectid)
3. [Dependencias `npm install`](#3-dependencias-npm-install)

---

## 1. Remote, fetch y merge

Ejecutar desde la **raíz del repo del usuario** (`targetPath` si se definió). Inferir `<repo-name>` del segmento final de `templateUrl` sin `.git` (ej. `https://github.com/org/my-base.git` → `my-base`).

### 1.1 Añadir remote `upstream`

```bash
git remote add upstream <templateUrl>
```

- Si Git indica que el remote **ya existe** y apunta a la **misma** URL de la tabla, **continuar** sin duplicar.
- Si existe pero apunta a **otra** URL, **no** sobrescribir; preguntar al usuario.

### 1.2 Obtener la plantilla

```bash
git fetch upstream
```

### 1.3 Merge con historias no relacionadas

```bash
git merge upstream/main --allow-unrelated-histories -m "Merge <repo-name> template"
```

- Si la plantilla cambiara la rama por defecto en el futuro, ajustar la ref (`upstream/<rama>`) **en este mismo archivo** y ejecutar según lo actualizado.
- Si hay **conflictos**, **parar** el flujo de personalización hasta resolución.

---

## 2. Personalización `projectName` y `projectId`

Tras merge **sin conflictos pendientes**, aplicar **`projectName`** y **`projectId`** en los sitios siguientes (rutas relativas a la raíz del repo). Usar **`projectId`** en **kebab-case** en todos los archivos donde corresponda identificador técnico.

| Archivo | Acción |
| ------- | ------ |
| `src/index.html` | `<title>` → **`projectName`**. |
| `angular.json` | Reemplazar **`angular-base-project`** por **`projectId`** donde aparezca. |
| `package.json` | `"name"` → **`projectId`**. |
| `package-lock.json` | Alinear `name` raíz y `packages[""].name` con **`projectId`**. |
| `.cursor/hooks/telemetry/scripts/send.js` | Reemplazar **`angular-base-project`** por **`projectId`**. |
| `public/manifest.webmanifest` | `name` y `short_name` → **`projectName`**. |
| `src/app/app.ts` | Valor de **`title`** del app según plantilla → **`projectId`**. |
| `src/environments/environment.ts` | Si existe: `appName` → **`projectName`**; `appId` → **`projectId`** (mantener estilo de comillas del archivo). |
| `src/environments/environment.development.ts` | Igual que `environment.ts` si los campos existen. |
| Otros `src/environments/environment.*.ts` | Misma convención si incluyen `appName` / `appId`. |

---

## 3. Dependencias (`npm install`)

En la raíz donde quede el `package.json` tras el merge.

1. **Preguntar** al usuario si debe ejecutarse `npm install`. **No** ejecutar sin confirmación afirmativa.
2. Si **sí**:

```bash
npm install
```

3. Si **no**, indicar el comando para ejecución manual.

---

## Errores habituales

| Situación | Acción |
| --------- | ------ |
| Sin `.git` | `git init` o clonar antes del [paso 1](#1-remote-fetch-y-merge). |
| `git remote add` falla (distinto de "already exists" con URL correcta) | Revisar stderr; no forzar. |
| `fetch` / `merge` fallan | Red, permisos, rama, o conflictos. |
| `npm install` falla | Node, lockfile, registry. |
| Nombres inválidos | Volver a [Validación](#validación-aplicar-antes-de-editar). |