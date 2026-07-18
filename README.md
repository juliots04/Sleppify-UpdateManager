# Sleppify Updates — panel (Astro SSR + MySQL)

Panel autohospedado para publicar las actualizaciones de la app. La app consulta
`GET /version.json` y, si el `versionCode` remoto supera al instalado, descarga el APK y lanza el
instalador.

## Qué trae de nuevo

- **Panel mejorado**: diseño oscuro con animaciones, vista previa 1:1 del popup de la app,
  validaciones de versión/código/novedades.
- **Opcional vs obligatoria**: si la marcas opcional, la app muestra un botón **“Más tarde”**; si es
  obligatoria, la única salida es actualizar.

## Estructura

```
src/
  middleware.ts            guard de sesión (/panel exige login)
  lib/{config,db,auth}.ts  configuración, MySQL (mysql2), scrypt + cookie firmada
  pages/
    index.astro            login
    panel.astro            panel (publicar + preview + historial + ajustes)
    logout.ts
    version.json.ts        manifiesto público que lee la app
    api/login.ts           valida credenciales, emite cookie
    api/publish.ts         sube APK + registra versión (validaciones + magic bytes)
    apk/[...file].ts       sirve los APKs de apk/
apk/                       APKs subidos (persisten en el hosting)
database.sql               esquema
.env / .env.example        configuración (DB, SESSION_SECRET, admin)
```

## Puesta en marcha (local)

```bash
cp .env.example .env      # y edita SESSION_SECRET
npm install
npm run dev               # http://localhost:4321
```

## Despliegue en alwaysdata (sitio Node)

1. **Base de datos**: en phpMyAdmin importa `database.sql` (crea las tablas). No crees el usuario a
   mano — se siembra solo en el primer login.

2. **En tu PC** (una vez):
   ```bash
   npm install
   npm run build          # genera dist/  (esto es lo pesado; hazlo aquí, no en el server)
   ```

3. **Sube a `~/www/`** por SFTP/rsync (FileZilla/WinSCP sirven): `dist/`, `package.json`,
   `package-lock.json`, `astro.config.mjs`, `database.sql`, `.env` y la carpeta `apk/`.

4. **En el servidor (SSH), instala SOLO runtime** (sin nativos → no hace OOM):
   ```bash
   cd ~/www
   npm install --omit=dev --no-optional --no-audit --no-fund --maxsockets 1
   ```
   Son ~270 paquetes / ~126 MB de JS, sin descargar esbuild/rollup/sharp.

5. **Configura el sitio** (Web > Sitios > Añadir un sitio):
   - Tipo: **Node.js**
   - Comando: `node ./dist/server/entry.mjs`
   - Directorio de trabajo: `~/www`

6. Apunta el dominio `sleppifymanagerupdate.alwaysdata.net` a ese sitio. La app ya pide
   `https://sleppifymanagerupdate.alwaysdata.net/version.json`.

## Publicar una versión

1. Entra al panel, rellena versión visible, version code (mayor que el actual), novedades y el
   título del popup, marca si es obligatoria u opcional, elige el `.apk` y pulsa **Publicar**.
2. La fila se guarda en `releases` y `GET /version.json` la refleja al instante. Los teléfonos la
   ven al abrir la app.

## Contrato de `version.json`

```json
{
  "versionName": "1.2.0",
  "versionCode": 5,
  "apk": "apk/sleppify-v1.2.0-5.apk",
  "size": 31510488,
  "notes": "- Novedad 1\n- Novedad 2",
  "dialogTitle": "🔥 NUEVA VERSIÓN DISPONIBLE",
  "mandatory": false,
  "publishedAt": "2026-07-18T01:00:00.000Z"
}
```

La app (`AppUpdateManager.kt`) lee `dialogTitle` (header del popup) y `mandatory` (si es `false`
muestra “Más tarde”). Los manifiestos antiguos sin esos campos se tratan como obligatorios con el
título por defecto.
