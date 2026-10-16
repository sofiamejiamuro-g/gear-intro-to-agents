# Guía Terminal: Despliegue en GitHub Pages & Troubleshooting

Esta guía documenta el flujo de trabajo completo para publicar proyectos estáticos en **GitHub Pages exclusivamente desde la terminal**, junto con las soluciones a todos los problemas reales de autenticación, permisos de tokens y conflictos de cuentas encontrados durante la configuración.

---

## 📋 Índice
1. [Flujo Paso a Paso 100% Terminal](#1-flujo-paso-a-paso-100-terminal)
   - [Paso 1: Preparación del Sitio y `.nojekyll`](#paso-1-preparación-del-sitio-y-nojekyll)
   - [Paso 2: Inicialización de Git Local (Aislado)](#paso-2-inicialización-de-git-local-aislado)
   - [Paso 3: Creación del Repositorio en GitHub vía API](#paso-3-creación-del-repositorio-en-github-vía-api)
   - [Paso 4: Configuración del Remoto y Push](#paso-4-configuración-del-remoto-y-push)
   - [Paso 5: Habilitación de GitHub Pages vía API](#paso-5-habilitación-de-github-pages-vía-api)
   - [Paso 6: Verificación del Estado del Despliegue](#paso-6-verificación-del-estado-del-despliegue)
2. [Troubleshooting & Problemas Comunes](#2-troubleshooting--problemas-comunes)
   - [Problema A: Conflicto entre múltiples cuentas en la misma máquina](#problema-a-conflicto-entre-múltiples-cuentas-en-la-misma-máquina)
   - [Problema B: Fine-Grained Token (`github_pat_`) vs Classic Token (`ghp_`)](#problema-b-fine-grained-token-github_pat_-vs-classic-token-ghp_)
   - [Problema C: `403 Resource not accessible by personal access token`](#problema-c-403-resource-not-accessible-by-personal-access-token)
   - [Problema D: El `credential.helper` global sobreescribe el token del repo](#problema-d-el-credentialhelper-global-sobreescribe-el-token-del-repo)
   - [Problema E: Assets no cargan o rutas bloqueadas por Jekyll](#problema-e-assets-no-cargan-o-rutas-bloqueadas-por-jekyll)
3. [Cheat Sheet: Script Rápido para Próximos Proyectos](#3-cheat-sheet-script-rápido-para-próximos-proyectos)

---

## 1. Flujo Paso a Paso 100% Terminal

### Paso 1: Preparación del Sitio y `.nojekyll`

En la raíz de tu proyecto estático necesitas:
- `index.html`: La página principal que servirá GitHub Pages.
- `.nojekyll`: Archivo vacío crucial. GitHub Pages usa el motor Jekyll por defecto. Si no agregas `.nojekyll`, Jekyll ignorará carpetas que inicien con `_` (como `_assets` o `_next`) y romperá templates que usen llaves dobles `{{ }}`.

```bash
# Crear el archivo .nojekyll en la raíz
touch .nojekyll
```

---

### Paso 2: Inicialización de Git Local (Aislado)

Para evitar alterar la identidad de tus otros proyectos en la máquina, configura `user.name` y `user.email` **a nivel local** (`--local` es el comportamiento por defecto si no pasas `--global`):

```bash
# 1. Inicializar el repositorio con rama principal 'main'
git init -b main

# 2. Configurar usuario y correo SOLO para este repositorio
git config user.name "Sofia Mejia"
git config user.email "tu_correo@google.com"

# 3. Agregar archivos y hacer el primer commit
git add .
git commit -m "Initial commit: website files and .nojekyll"
```

---

### Paso 3: Creación del Repositorio en GitHub vía API

Puedes crear el repositorio directamente desde la terminal con `curl` hacia la API REST de GitHub:

```bash
# Variables del proyecto
GITHUB_USER="sofiamejiamuro-g"
REPO_NAME="gear-intro-to-agents"
GITHUB_TOKEN="tu_token_aqui"

# Crear repo público sin inicializar
curl -X POST \
  -H "Authorization: Bearer ${GITHUB_TOKEN}" \
  -H "Accept: application/vnd.github.v3+json" \
  https://api.github.com/user/repos \
  -d '{
    "name": "'"${REPO_NAME}"'",
    "description": "Infografía Interactiva: Agentes de IA & Gemini Enterprise",
    "private": false,
    "auto_init": false
  }'
```

> **Nota:** Si usas un **Classic Token (`ghp_`)**, esta llamada crea el repo inmediatamente con código `201 Created`. Si usas un **Fine-Grained Token (`github_pat_`)**, revisa la sección de Troubleshooting abajo.

---

### Paso 4: Configuración del Remoto y Push

Para que Git use el token **únicamente para este proyecto**, incrusta el token en la URL del remote `origin`:

```bash
# Vincular el remoto con token embebido
git remote add origin "https://${GITHUB_USER}:${GITHUB_TOKEN}@github.com/${GITHUB_USER}/${REPO_NAME}.git"

# (Si ya existía el remoto, usa 'set-url')
git remote set-url origin "https://${GITHUB_USER}:${GITHUB_TOKEN}@github.com/${GITHUB_USER}/${REPO_NAME}.git"

# Subir a la rama main
git push -u origin main
```

---

### Paso 5: Habilitación de GitHub Pages vía API

Una vez que la rama `main` contiene el código, activa GitHub Pages desde la terminal apuntando a la raíz (`/`) de `main`:

```bash
curl -X POST \
  -H "Authorization: Bearer ${GITHUB_TOKEN}" \
  -H "Accept: application/vnd.github.v3+json" \
  https://api.github.com/repos/${GITHUB_USER}/${REPO_NAME}/pages \
  -d '{
    "source": {
      "branch": "main",
      "path": "/"
    }
  }'
```

La API responderá con `201 Created` y devolverá la URL pública:
```json
{
  "html_url": "https://sofiamejiamuro-g.github.io/gear-intro-to-agents/",
  "status": "building"
}
```

---

### Paso 6: Verificación del Estado del Despliegue

Monitorea el estado del build hasta que cambie de `building` a `built`:

```bash
# Consultar estado del build en la API
curl -s \
  -H "Authorization: Bearer ${GITHUB_TOKEN}" \
  -H "Accept: application/vnd.github.v3+json" \
  https://api.github.com/repos/${GITHUB_USER}/${REPO_NAME}/pages | grep '"status"'

# Verificar que la URL responde HTTP 200
curl -Is "https://${GITHUB_USER}.github.io/${REPO_NAME}/" | head -n 5
```

Salida esperada:
```http
HTTP/2 200 
server: GitHub.com
content-type: text/html; charset=utf-8
```

---

## 2. Troubleshooting & Problemas Comunes

### Problema A: Conflicto entre múltiples cuentas en la misma máquina

#### Síntoma:
Al intentar hacer `git push` o interactuar con el repositorio, recibes:
```
ERROR: Permission to usuario-A/repo.git denied to usuario-B.
fatal: unable to access ... The requested URL returned error: 403
```

#### Causa Raíz:
- En tu máquina existen credenciales guardadas en `~/.git-credentials` o una clave SSH en `~/.ssh/id_ed25519` que corresponden a otra cuenta (`usuario-B`).
- Aunque escribas la URL como `usuario-A`, Git o SSH negocian la sesión usando las credenciales predeterminadas de la máquina.

#### Diagnóstico:
```bash
# 1. Ver qué usuario reconoce SSH actualmente
ssh -T git@github.com
# Retorna: "Hi usuario-B! You've successfully authenticated..."

# 2. Ver qué usuario tiene el token activo en la API
curl -s -H "Authorization: Bearer $GITHUB_TOKEN" https://api.github.com/user | grep '"login"'
```

#### Solución:
Incrusta la credencial del proyecto en la URL de Git para anular la global:
```bash
git remote set-url origin https://<USUARIO_CORRECTO>:<TOKEN_CORRECTO>@github.com/<USUARIO_CORRECTO>/<REPO>.git
```
Y si el helper de credenciales global sigue interfiriendo:
```bash
git config --local credential.helper ""
```

---

### Problema B: Fine-Grained Token (`github_pat_`) vs Classic Token (`ghp_`)

GitHub cuenta con dos sistemas de tokens:

| Característica | Classic Token (`ghp_...`) | Fine-Grained Token (`github_pat_...`) |
| :--- | :--- | :--- |
| **Generación** | Settings &rarr; Tokens (classic) | Settings &rarr; Fine-grained tokens |
| **Configuración** | Basta marcar el checkbox `[x] repo` | Debes configurar permisos individuales |
| **Crear repos por API** | ✅ **Soportado** (`POST /user/repos`) | ❌ **No soportado** para cuentas personales |
| **Permisos por defecto** | Hereda los scopes seleccionados | **"No access"** en todas las categorías |
| **Recomendado para** | Scripts y automatización terminal | Accesos muy restringidos / seguridad granular |

> 💡 **Lección:** Para crear repositorios y automatizar despliegues 100% desde la terminal, un **Classic Token** con permiso `repo` es el más ágil. Si usas **Fine-Grained**, el repositorio debe crearse en GitHub.com primero.

---

### Problema C: `403 Resource not accessible by personal access token`

#### Síntoma:
Al hacer `git push` o llamar al API de Pages con un Fine-Grained Token, recibes:
```
HTTPError: 403 {"message":"Resource not accessible by personal access token"}
```

#### Causa:
El token fue creado pero no tiene asignados los permisos de lectura/escritura para el contenido o para Pages.

#### Solución:
No hace falta generar un nuevo token:
1. Ve a [github.com/settings/tokens](https://github.com/settings/tokens).
2. Haz clic en el token existente.
3. En **Repository permissions**:
   - **Contents**: cámbialo a **Access: Read and write** (permite `git push`).
   - **Pages**: cámbialo a **Access: Read and write** (permite habilitar Pages vía API).
4. Si seleccionaste *"Only select repositories"*, asegúrate de agregar el nuevo repositorio a la lista.
5. Haz clic en **Update token**. La cadena del token no cambia.

---

### Problema D: El `credential.helper` global sobreescribe el token del repo

#### Síntoma:
A pesar de configurar `https://user:token@github.com/...`, Git insiste en usar la credencial vieja y arroja error 403.

#### Causa:
Si en tu sistema existe `credential.helper=store`, Git busca primero en `~/.git-credentials`. Si encuentra una entrada para `github.com`, ignora el token de la URL.

#### Solución:
Deshabilita el credential helper **únicamente dentro de la carpeta del proyecto**:
```bash
git config --local credential.helper ""
```
O fuerza el header de autorización en el comando:
```bash
git -c http.extraHeader="Authorization: Bearer ${GITHUB_TOKEN}" push -u origin main
```

---

### Problema E: Assets no cargan o rutas bloqueadas por Jekyll

#### Síntoma:
La página web carga en blanco, los estilos CSS/JS no se encuentran (404), o carpetas con prefijo `_` desaparecen.

#### Causa:
GitHub Pages asume por defecto que el proyecto es un sitio Jekyll e ignora archivos y directorios con guion bajo `_` o intenta procesar código en llaves dobles.

#### Solución:
Colocar siempre un archivo `.nojekyll` vacío en la raíz del repositorio:
```bash
touch .nojekyll
git add .nojekyll
git commit -m "fix: disable jekyll processing"
git push origin main
```

---

## 3. Cheat Sheet: Script Rápido para Próximos Proyectos

Guarda este fragmento para publicar cualquier proyecto estático futuro en un solo comando:

```bash
#!/usr/bin/env bash
set -e

# 1. Configuración de variables
GH_USER="sofiamejiamuro-g"
GH_REPO="nombre-de-tu-nuevo-repo"
GH_TOKEN="ghp_TU_CLASSIC_TOKEN_AQUI"

# 2. Archivos base
touch .nojekyll

# 3. Git local
git init -b main
git config user.name "Sofia Mejia"
git config user.email "sofiamejiamuro@google.com"
git config --local credential.helper ""
git add .
git commit -m "Initial commit"

# 4. Crear repo en GitHub vía API
curl -s -X POST \
  -H "Authorization: Bearer ${GH_TOKEN}" \
  -H "Accept: application/vnd.github.v3+json" \
  https://api.github.com/user/repos \
  -d "{\"name\":\"${GH_REPO}\",\"private\":false,\"auto_init\":false}" > /dev/null

# 5. Push autenticado
git remote add origin "https://${GH_USER}:${GH_TOKEN}@github.com/${GH_USER}/${GH_REPO}.git"
git push -u origin main

# 6. Activar GitHub Pages
curl -s -X POST \
  -H "Authorization: Bearer ${GH_TOKEN}" \
  -H "Accept: application/vnd.github.v3+json" \
  https://api.github.com/repos/${GH_USER}/${GH_REPO}/pages \
  -d '{"source":{"branch":"main","path":"/"}}' > /dev/null

echo "✅ Proyecto publicado exitosamente en: https://${GH_USER}.github.io/${GH_REPO}/"
```
