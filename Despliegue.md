# Despliegue de la Pokédex en Azure Static Web Apps

## 1. Preparación del proyecto Angular

### 1.1. Ubicación del código fuente

El proyecto Angular original de la Pokédex se ubicó en:

- `source/pokedex-angular/`

Esto incluye:

- `src/app`, `src/assets`, `src/environments`.
- Archivos de configuración (`angular.json`, `package.json`, `tsconfig.app.json`, etc.).

### 1.2. Instalación de dependencias

Desde Git Bash:

```bash
cd source/pokedex-angular
npm install
```

Durante la instalación apareció un warning de `husky` indicando que no encontraba `.git` en ese directorio, lo cual es esperado porque el repositorio Git está en la carpeta raíz (`poke-dex-lab`) y no en el subproyecto.  
El warning no afectó al funcionamiento de la aplicación.

### 1.3. Verificación en modo desarrollo

Para asegurar que el proyecto funcionaba correctamente:

```bash
cd source/pokedex-angular
ng serve
```

Angular levantó el servidor de desarrollo en `http://localhost:4200/`.  
Probé la navegación de:

- Home (listado de pokémon, filtros, ordenamiento).
- Detalle de pokémon.
- Sección About.

No se presentaron errores en la consola del navegador en modo desarrollo.

---

## 2. Generación del build estático

### 2.1. Comando de build

Para generar la versión optimizada y apta para despliegue:

```bash
cd source/pokedex-angular
npm run build
```

Angular creó la carpeta:

- `source/pokedex-angular/dist/pokedex-angular/`

con los siguientes archivos relevantes:

- `index.html`
- `main.*.js`
- `polyfills.*.js`
- `runtime.*.js`
- `styles.*.css`
- Imágenes y fuentes empaquetadas
- Chunks lazy para módulos de rutas (home, details, about, etc.)

---

## 3. Organización del repositorio para Azure

### 3.1. Relación con Azure Static Web Apps

Al crear la Static Web App en Azure, configuré:

- Repositorio: `Vasibe/pokedex`
- Rama: `main`
- **App location**: `/` (raíz del repo)

Esto significa que Azure esperaba encontrar un sitio estático listo para servir directamente en la raíz del repositorio.

### 3.2. Copia del build a la raíz

Para alinear la salida de Angular con lo que Azure sirve, copié el contenido de `dist/pokedex-angular` a la raíz del proyecto.

Comandos usados desde la raíz (`poke-dex-lab`):

```bash
cd poke-dex-lab

mkdir -p build-final
cp -r source/pokedex-angular/dist/pokedex-angular/* build-final/
cp -r build-final/* .
```

Ahora la raíz del repo contenía:

- `index.html` (generado por Angular).
- Todos los bundles JS y CSS.
- Imágenes y fuentes optimizadas.

La carpeta `source/pokedex-angular` se mantuvo intacta con el código fuente para futuros cambios.

---

## 4. Configuración del pipeline de despliegue

### 4.1. Creación del workflow de GitHub Actions

Desde el portal de Azure, al crear la Static Web App:

1. Seleccioné GitHub como fuente de código.
2. Elegí el repo `Vasibe/pokedex` y la rama `main`.
3. Al finalizar, Azure creó en el repositorio un archivo YAML en `.github/workflows`, con un nombre similar a:
   - `azure-static-web-apps-<hash>.yml`

Este workflow realiza, en cada `git push` a `main`:

- La descarga del repo.
- La publicación del contenido de la carpeta configurada como App location (`/`) en la Static Web App.

### 4.2. Ciclo de trabajo

Cada vez que hacía cambios relevantes:

```bash
git add .
git commit -m "Mensaje descriptivo"
git push
```

GitHub Actions ejecutaba el workflow y el portal de Azure mostraba el historial de despliegues en la sección **GitHub Action runs**.

---

## 5. Ajustes y problemas encontrados

### 5.1. Pantalla en blanco en el primer despliegue

En un intento inicial, se intentó desplegar un `index.html` de desarrollo (el que solo contiene `<app-root></app-root>` sin los `<script src="...">` del build).  
Resultado: la página se veía en blanco porque Angular nunca llegaba a montarse.

**Solución:**

- Asegurarse de desplegar el `index.html` generado por `ng build`, que sí incluye las referencias a `runtime.*.js`, `polyfills.*.js`, `main.*.js` y `styles.*.css`.

### 5.2. Errores 404 de imágenes en producción

Tras el primer despliegue del build, aparecían errores 404 en la consola para algunas imágenes, por ejemplo:

- `GET https://.../pokedex-angular/assets/images/pokemon-green.png 404 (Not Found)`

Esto se debía a que, en el código original, algunas rutas estaban pensadas para GitHub Pages (`/pokedex-angular/...`), pero en Azure la aplicación se sirve desde `/`.

**Solución:**

1. Revisar el código fuente en `source/pokedex-angular/src` y actualizar las rutas de assets.
2. Cambiar rutas tipo `/pokedex-angular/assets/...` por `/assets/...`.
3. Ejecutar nuevamente:

   ```bash
   cd source/pokedex-angular
   npm run build
   ```

4. Volver a copiar el contenido de `dist/pokedex-angular` a la raíz y hacer `git push`.

Con esto, las imágenes cargaron correctamente tanto en local como en Azure.

---

## 6. Configuración de seguridad (staticwebapp.config.json)

### 6.1. Creación del archivo de configuración

En la raíz del repo (`poke-dex-lab`) se creó el archivo:

- `staticwebapp.config.json`

Contenido principal:

```json
{
  "globalHeaders": {
    "Content-Security-Policy": "default-src 'self'; img-src 'self' https://raw.githubusercontent.com https://assets.pokemon.com data:; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; font-src 'self' data: https://fonts.gstatic.com; script-src 'self' 'unsafe-inline'; connect-src 'self' https://pokeapi.co https://beta.pokeapi.co",
    "Strict-Transport-Security": "max-age=31536000; includeSubDomains; preload",
    "X-Content-Type-Options": "nosniff",
    "X-Frame-Options": "DENY",
    "Referrer-Policy": "no-referrer",
    "Permissions-Policy": "camera=(), microphone=(), geolocation=(), payment=(), usb=()"
  },
  "navigationFallback": {
    "rewrite": "/index.html",
    "exclude": [
      "/assets/*",
      "/*.js",
      "/*.css",
      "/*.png",
      "/*.jpg",
      "/*.jpeg",
      "/*.gif",
      "/*.svg",
      "/*.ico"
    ]
  }
}
```

### 6.2. Iteraciones de CSP y problemas encontrados

- Primera versión de CSP:
  - Solo permitía `self` y muy pocos dominios externos.
  - Resultado: se bloquearon recursos legítimos:
    - Google Fonts (`fonts.googleapis.com`, `fonts.gstatic.com`).
    - Imágenes de Pokémon (`assets.pokemon.com`).
    - Llamadas a `https://beta.pokeapi.co/graphql/v1beta`.
  - La aplicación mostraba errores y se veía una pantalla de error de PokéAPI.

- Ajustes realizados:
  - Se añadieron explícitamente:
    - `https://fonts.googleapis.com` a `style-src`.
    - `https://fonts.gstatic.com` a `font-src`.
    - `https://assets.pokemon.com` a `img-src`.
    - `https://pokeapi.co` y `https://beta.pokeapi.co` a `connect-src`.
  - Se mantuvo `'unsafe-inline'` en `script-src` porque el código heredado utiliza scripts inline que no era posible refactorizar en el contexto de la actividad.

Después de estos ajustes, la app volvió a funcionar normalmente y la consola dejó de mostrar errores de CSP.

### 6.3. navigationFallback para rutas de SPA

Sin configuración adicional, recargar la página en rutas como `/pokemon/4` puede provocar 404 de Azure.  
El bloque `navigationFallback`:

- `rewrite: "/index.html"` indica que cualquier ruta desconocida debe responder con `index.html`, permitiendo que Angular se encargue del ruteo.
- `exclude` evita que esta regla afecte a recursos estáticos (JS, CSS, imágenes, etc.), que siguen devolviendo 404 real si no existen.

---

## 7. Verificación de seguridad con SecurityHeaders.com

Para validar los encabezados de seguridad configurados:

1. Accedí a https://securityheaders.com.
2. Analicé la URL de la Static Web App:
   - `https://gray-mushroom-0d8216610.7.azurestaticapps.net`

3. Resultado final:
   - Calificación: **A**
   - Headers en verde:
     - `Strict-Transport-Security`
     - `Referrer-Policy`
     - `X-Content-Type-Options`
     - `Content-Security-Policy`
     - `X-Frame-Options`
     - `Permissions-Policy`
   - Advertencia:
     - Indica que la política CSP incluye `'unsafe-inline'` en `script-src`.  
       Esta decisión se tomó conscientemente debido al uso de scripts inline en el código original.

La captura del reporte se guarda como evidencia en la entrega de la actividad.

---

## 8. Conclusión

El despliegue de la Pokédex en Azure Static Web Apps requirió:

- Entender que la plataforma sirve **archivos estáticos** y que, para proyectos Angular, es necesario desplegar el contenido de `dist`, no la carpeta `src`.
- Ajustar la estructura del repo para que la raíz coincida con el build.
- Resolver errores de rutas y assets (404) ocasionados por diferencias entre el despliegue original del autor y el entorno Azure.
- Configurar cabeceras de seguridad HTTP en `staticwebapp.config.json`, equilibrando seguridad y funcionalidad.
- Validar la configuración mediante un escaneo externo (`securityheaders.com`) hasta alcanzar una calificación A.

El resultado final es una aplicación funcional, accesible desde una URL pública y con una configuración de seguridad razonablemente robusta para el contexto de la actividad.
