# Pokédex – Despliegue en Azure Static Web Apps

## 1. Descripción general

Este repositorio contiene el despliegue en la nube de una aplicación tipo Pokédex desarrollada originalmente en Angular.  
La aplicación final se publica como **sitio web estático** (HTML, CSS y JavaScript) sobre **Azure Static Web Apps**, cumpliendo el requisito de que solo necesita un servidor que sirva archivos estáticos.

- URL pública de la app: https://gray-mushroom-0d8216610.7.azurestaticapps.net
- Proveedor cloud: Microsoft Azure – Static Web Apps
- Tipo de app: SPA estática (Angular compilado a archivos estáticos)

---

## 2. Creación de la cuenta y recursos en Azure

### 2.1. Registro en Azure

1. Ingresé a https://portal.azure.com con la cuenta universitaria de Microsoft.
2. Activé la suscripción gratuita de Azure para poder crear recursos básicos.
3. Verifiqué que la suscripción estuviera activa en el apartado **Subscriptions** del portal.

### 2.2. Creación de Azure Static Web App

1. En el portal de Azure seleccioné **Create a resource** → **Static Web App**.
2. Completé los campos principales:
   - **Subscription**: suscripción gratuita.
   - **Resource group**: creé un grupo nuevo (por ejemplo, `rg-pokedex-swa`).
   - **Name**: `pokedex-swa-valeria` (nombre del recurso).
   - **Region**: región cercana (en mi caso, `Central US`).
3. En la sección de **Deployment**:
   - Elegí **GitHub** como fuente de código.
   - Autoricé a Azure a acceder a mi cuenta de GitHub.
   - Seleccioné el repositorio `Vasibe/pokedex` y la rama `main`.
4. En **Build details**:
   - **App location**: `/` (raíz del repositorio).
   - API y otros campos: vacíos, porque no tengo backend aquí.
5. Confirmé la creación. Azure generó automáticamente un archivo de workflow de GitHub Actions (`.github/workflows/azure-static-web-apps-*.yml`) que despliega la app cada vez que hago `git push` a `main`.

---

## 3. Estructura del repositorio

- `/` (raíz del repo):  
  Contiene el **build de producción** generado por Angular: `index.html`, `main.*.js`, `runtime.*.js`, `polyfills.*.js`, `styles.*.css`, imágenes, fuentes, etc.  
  Esta es la carpeta que Azure sirve directamente gracias a `App location = /`.

- `/source/pokedex-angular`:  
  Código fuente original del proyecto Angular (entregado por el profesor).  
  Contiene `src/` con `app`, `assets`, `environments`, más archivos de configuración (`angular.json`, `package.json`, `tsconfig*`, etc.).  
  Aquí es donde ejecuto `npm install`, `ng serve` y `npm run build` durante el desarrollo.

- `/staticwebapp.config.json`:  
  Archivo de configuración de Azure Static Web Apps.  
  Define cabeceras HTTP de seguridad (CSP, HSTS, X-Content-Type-Options, X-Frame-Options, Referrer-Policy, Permissions-Policy).  
  Configura `navigationFallback` para que las rutas internas de la SPA se resuelvan a `index.html`.

Esta organización separa claramente:

- **Desarrollo**: proyecto Angular completo en `source/pokedex-angular`.
- **Despliegue**: artefactos estáticos que se sirven en producción desde la raíz del repositorio.

---

## 4. Proceso de despliegue (resumen)

1. **Preparar el proyecto Angular:**

   ```bash
   cd source/pokedex-angular
   npm install
   ng serve   # Verificación en http://localhost:4200
   ```

   Verifiqué que la Pokédex funcionara sin errores en local.

2. **Generar el build estático:**

   ```bash
   cd source/pokedex-angular
   npm run build
   ```

   Esto creó `dist/pokedex-angular` con el `index.html` y todos los bundles estáticos.

3. **Copiar el build a la raíz del repo:**

   ```bash
   cd poke-dex-lab
   mkdir -p build-final
   cp -r source/pokedex-angular/dist/pokedex-angular/* build-final/
   cp -r build-final/* .
   ```

   De esta forma, la raíz del repo coincidió con el contenido de `dist`, que es lo que entiende Azure como sitio estático.

4. **Conectar con Azure Static Web Apps:**
   - Configuré el recurso para usar el repositorio `Vasibe/pokedex`, rama `main` y `App location = /`.
   - El workflow de GitHub Actions se encarga de desplegar cada nueva versión cuando hago `git push`.

5. **Configurar cabeceras de seguridad y navegación:**
   - Creé y ajusté `staticwebapp.config.json` para añadir headers de seguridad y `navigationFallback` hacia `index.html`.
   - Redeplegué la app y validé los cambios.

---

## 5. Seguridad: encabezados HTTP y análisis con SecurityHeaders.com

Después de tener la app funcionando en Azure, apliqué buenas prácticas de seguridad web a nivel de cabeceras HTTP.

### 5.1. Encabezados configurados

En `staticwebapp.config.json` configuré:

- `Content-Security-Policy`:
  - Restringe orígenes de recursos:
    - `default-src 'self'`
    - `img-src 'self' https://raw.githubusercontent.com https://assets.pokemon.com data:`
    - `style-src 'self' 'unsafe-inline' https://fonts.googleapis.com`
    - `font-src 'self' data: https://fonts.gstatic.com`
    - `script-src 'self' 'unsafe-inline'`
    - `connect-src 'self' https://pokeapi.co https://beta.pokeapi.co`
  - Con esto limito scripts, estilos, imágenes y peticiones XHR a dominios específicos.

- `Strict-Transport-Security`:
  - `max-age=31536000; includeSubDomains; preload`
  - Obliga a los navegadores a usar siempre HTTPS con HSTS.

- `X-Content-Type-Options: nosniff`:
  - Evita que el navegador intente adivinar tipos de contenido (previene ciertos ataques).

- `X-Frame-Options: DENY`:
  - Impide que la app se muestre en iframes externos (mitiga clickjacking).

- `Referrer-Policy: no-referrer`:
  - Reduce la fuga de información de la URL de origen en peticiones externas.

- `Permissions-Policy`:
  - `camera=(), microphone=(), geolocation=(), payment=(), usb=()`
  - Deniega explícitamente permisos del navegador que la app no utiliza.

Además, configuré:

- `navigationFallback` para que las rutas internas de la SPA (`/pokemon/4`, `/about`, etc.) se redirijan a `index.html`, evitando 404 de Azure cuando se recarga la página.

### 5.2. Resultado del escaneo en SecurityHeaders.com

- Ejecuté el escaneo en https://securityheaders.com contra la URL:
  - `https://gray-mushroom-0d8216610.7.azurestaticapps.net`
- Resultado: **calificación A**, con todos los encabezados principales en verde.
- El sitio indica una advertencia sobre el uso de `'unsafe-inline'` en `script-src`.  
  En este contexto lo acepté como trade-off porque el código heredado de la Pokédex utiliza scripts inline y no puedo refactorizar la app completa, pero la CSP sigue limitando scripts a mi propio dominio y a los orígenes estrictamente necesarios.

La captura del reporte de Security Headers forma parte del material entregado.

---

## 6. Análisis personal

### 6.1. ¿Qué vulnerabilidades ayudan a prevenir los encabezados implementados?

- **Content-Security-Policy:**
  - Reduce el riesgo de ataques de Cross-Site Scripting (XSS) al limitar de dónde se pueden cargar scripts, estilos e imágenes.
  - Evita la carga de recursos desde dominios no autorizados, lo que dificulta la inyección de contenido malicioso.

- **Strict-Transport-Security:**
  - Obliga el uso de HTTPS, evitando ataques de tipo downgrade y reduciendo la posibilidad de sniffing de tráfico en HTTP plano.

- **X-Content-Type-Options (nosniff):**
  - Evita que el navegador interprete archivos con tipos de contenido incorrectos, mitigando ciertos vectores de ataque que se aprovechan de esa “adivinación”.

- **X-Frame-Options (DENY):**
  - Previene que la aplicación se incruste en iframes de terceros, reduciendo el riesgo de clickjacking.

- **Referrer-Policy (no-referrer):**
  - Minimiza la exposición de la URL interna de la app en peticiones a otros dominios, lo que protege información sensible en rutas o parámetros.

- **Permissions-Policy:**
  - Bloquea funcionalidades como cámara, micrófono, geolocalización, pago o USB que la aplicación no necesita, reduciendo la superficie de ataque en el navegador.

### 6.2. ¿Qué aprendí sobre la relación entre despliegue y seguridad web?

- Desplegar una SPA (Angular) como **web estática** implica entender qué artefactos realmente consume el navegador: no se sube `src`, se sube **`dist`**.
- La elección de la carpeta de despliegue y la configuración de `App location` en Azure están muy relacionadas: si no coinciden, aparecen errores 404 y pantallas en blanco.
- Las políticas de seguridad (especialmente CSP) no se pueden aplicar de forma genérica: hay que alinearlas con los dominios y recursos que la app realmente utiliza, o se rompe la funcionalidad.
- Herramientas como `securityheaders.com` son útiles para validar y ajustar la configuración, pero también hay que interpretar sus advertencias según el contexto del proyecto.

### 6.3. ¿Qué desafíos encontré en el proceso?

- **Despliegue inicial en blanco:**
  - El primer intento de despliegue usaba un `index.html` de desarrollo (solo `<app-root>` sin bundles JS), lo que resultaba en una pantalla en blanco.
  - La solución fue generar el build con Angular (`npm run build`) y desplegar el contenido de `dist/pokedex-angular` en la carpeta que Azure sirve.

- **Rutas y assets con 404:**
  - Aparecían 404 en imágenes porque algunas rutas estaban pensadas para GitHub Pages (`/pokedex-angular/assets/...`).
  - Ajusté las rutas a `/assets/...` y volví a compilar para que coincidieran con el despliegue en Azure.

- **Ajuste fino de Content-Security-Policy:**
  - Una CSP demasiado estricta rompía la app (bloqueaba Google Fonts, assets de Pokémon y las llamadas a PokéAPI).
  - Tuve que iterar la configuración para permitir únicamente los dominios que el código heredado realmente necesita, buscando un equilibrio entre seguridad y funcionalidad.

En conjunto, la actividad me ayudó a ver que **despliegue y seguridad web están directamente conectados**: no basta con “subir la app”, también hay que entender cómo se sirven los archivos y qué controles de seguridad se aplican a nivel de servidor.
