# PlaylistDL

Descargador de playlists/videos de YouTube a MP3/MP4. Basado en el repo de Angel050521 (MIT); mantener `LICENSE` y el crédito al autor. El README sigue siendo el original. Ya se eliminó un `public/sw.js` de anuncios (`3nbf4.com`); no reintroducir scripts de terceros.

Lo usan otras personas (familia y amigos), así que no debe depender de Music System: `/api/auto-download` es una integración opcional para el organizador personal del autor.

## Arranque
- `npm run dev` → `server.js` en el puerto `PORT` (3000 por defecto).
- Requiere `yt-dlp` y `ffmpeg`. Si yt-dlp no está en el PATH, usar `YTDLP_PATH`.
- `npm run electron` / `npm run build`: versión de escritorio (`main.js`), poco usada.
- Dockerfile, `fly.toml`, `netlify.toml`, `start.sh`: despliegue en la nube; no hacen falta en local.

## Variables de entorno
| Variable | Uso |
|---|---|
| `PORT` | Puerto (3000) |
| `HOST` | Interfaz de escucha. Por defecto `127.0.0.1` (solo esta PC); `0.0.0.0` para compartir en la red local. El Dockerfile la pone en `0.0.0.0` |
| `YTDLP_PATH` | Ruta a yt-dlp |
| `YOUTUBE_COOKIES` | Contenido de cookies.txt; se escribe en `/tmp/cookies.txt` |
| `MUSIC_SYSTEM_SECRET` | Clave obligatoria para `/api/auto-download` (debe ser igual a `PLAYLISTDL_SECRET` de Music System) |

## Estructura
- `server.js`: toda la lógica.
  - `POST /api/playlist-info`: lista de canciones (`yt-dlp --flat-playlist -J`).
  - Socket `start-download` → `downloadWithConcurrency` (6 en paralelo) → `downloadSong`. Emite `download-started`, `song-progress`, `global-stats`, `download-complete`, `download-error`.
  - `GET /api/download/:sessionId/:filename` y `GET /api/download-zip/:sessionId`.
  - `POST /api/auto-download`: integración con Music System (3 en paralelo; luego copia a `/sdcard/Download/` en Termux).
  - `cleanOldDownloads`: cada 30 min borra carpetas de `downloads/` con más de 1 hora.
- `public/`: interfaz (`index.html`, `app.js`, `styles.css`), sin framework.

## Reglas de seguridad (no romper)
- Todo valor que venga del cliente y llegue a yt-dlp o al disco se valida:
  - URLs: deben empezar con `http(s)://` (`isValidUrl`) y se pasan después de `--`, para que yt-dlp nunca las lea como opciones (riesgo de `--exec`).
  - `sessionId` debe cumplir `SESSION_ID_REGEX`; los nombres de archivo pasan por `isSafeFilename`.
  - `filenameTemplate` no puede contener `/`, `\` ni `..`.
- Usar siempre `spawn` con array de argumentos, nunca `exec` con texto que venga del cliente.
- No volver a añadir CORS abierto ni scripts externos de anuncios o analítica.
