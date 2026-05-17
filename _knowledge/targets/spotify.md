# Spotify — Research

Tipo: D — API oficial con OAuth2 PKCE (no anti-bot, rate limiting por tier)
Recon via: análisis de código (spoti-cli)

## Portal Map

| Portal | URL Base | Auth Method | Anti-bot | Contenido |
|--------|----------|-------------|----------|-----------|
| Authorization | https://accounts.spotify.com | OAuth2 PKCE | Ninguno | User login, scope approval |
| API | https://api.spotify.com/v1 | Bearer Token | Rate limiting por tier | Search, playback, library, playlists, personalization |
| Token Endpoint | https://accounts.spotify.com/api/token | POST OAuth2 | Ninguno | access_token + refresh_token |

## Auth Flow

### Mecanismo
- Tipo de auth: **OAuth2 PKCE** (Public Client — sin client_secret)
- Redirect URI: `http://127.0.0.1:8888/callback`
- Config guardada en `~/.spoti-cli/config.json`:
  ```json
  { "client_id": "...", "access_token": "...", "refresh_token": "...", "expires_at": 1716119400000, "scopes": [...] }
  ```

### Flujo PKCE
1. Generar `code_verifier` (128 chars, base64url random)
2. Derivar `code_challenge = SHA256(code_verifier)` en base64url
3. Redirect a `/authorize` con `code_challenge` (method=S256)
4. Usuario aprueba scopes → `code` llega al callback local
5. POST `/api/token` con `code`, `code_verifier`, `client_id`
6. Response: `{ access_token, refresh_token, expires_in }`

### Gotchas descubiertos en el código
- PKCE es criptográficamente obligatorio — sin `code_challenge` → 400
- Auto-refresh cuando `Date.now() > expires_at - 60_000` (margen 60s)
- 401 en request → retry automático con refresh, una vez
- `--upgrade` hace merge de scopes, no reemplaza
- Playback control requiere Spotify Premium (→ 403 sin Premium)
- Local auth server timeout: 120 segundos

## Headers críticos

```
Authorization: Bearer {access_token}
Content-Type: application/json
```

Para token exchange:
```
Content-Type: application/x-www-form-urlencoded
```

Spotify no requiere User-Agent, deviceId, timestamps — solo Bearer.

## Endpoints descubiertos

| Method | Endpoint | Scopes | Descripción |
|--------|----------|--------|-------------|
| GET | `/search?q=...&type=track\|artist\|album&limit=10` | — | Búsqueda |
| GET | `/me` | `user-read-private` | Perfil del usuario |
| GET | `/me/player/currently-playing` | `user-read-currently-playing` | Track actual |
| PUT | `/me/player/play?device_id=...` | `user-modify-playback-state` | Reproducir (Premium) |
| PUT | `/me/player/pause` | `user-modify-playback-state` | Pausar (Premium, retorna 204) |
| POST | `/me/player/next` | `user-modify-playback-state` | Skip |
| POST | `/me/player/queue?uri=...` | `user-modify-playback-state` | Agregar a queue |
| GET | `/me/player/devices` | `user-read-playback-state` | Devices disponibles |
| GET | `/me/top/tracks?time_range=short_term\|medium_term\|long_term` | `user-top-read` | Top tracks |
| GET | `/me/top/artists` | `user-top-read` | Top artists |
| GET | `/me/player/recently-played?limit=20` | `user-read-recently-played` | Historial |
| GET | `/me/tracks` | `user-library-read` | Liked tracks |
| PUT | `/me/tracks?ids=...` | `user-library-modify` | Guardar tracks |
| GET | `/me/playlists` | `playlist-read-private` | Mis playlists |
| POST | `/users/{user_id}/playlists` | `playlist-modify-public/private` | Crear playlist |
| POST | `/playlists/{id}/tracks` | `playlist-modify-*` | Agregar tracks |
| GET | `/tracks/{id}/audio-features` | — | Energy, tempo, danceability |
| GET | `/recommendations?seed_artists=...&seed_tracks=...` | — | Recomendaciones |

Params comunes: `limit` (0-50), `offset` (paginación), `time_range`, `market` (ISO 3166-1)

## Anti-bot / Protecciones

- **Bloquea headless**: NO — CLI funciona sin browser
- **WAF**: Ninguno (API pública)
- **Rate limiting**: 429 + header `Retry-After` cuando se excede quota
- **Scope enforcement**: 403 si request sin scope requerido
- **Geographic restriction**: Playback control puede dar 403 por país

## Notas críticas de implementación

- `PUT /me/player/pause` retorna 204 (sin body) — manejar con fallback `{}`
- Crear playlist: endpoint es `/users/{user_id}/playlists`, no `/me/playlists`
- Spotify URIs: el CLI auto-convierte IDs a `spotify:track:ID`, `spotify:artist:ID`
- Token expiry: asume 1 hora (3600s) — Spotify no siempre lo dice explícitamente
