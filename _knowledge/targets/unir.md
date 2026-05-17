# UNIR — Research

Tipo: B — OAuth2 form-post via SSO proxy + Moodle LMS + LTI sub-apps (CMS, Panopto)
Recon via: análisis de código (unir-cli)

## Portal Map

| Portal | URL Base | Auth Method | Anti-bot | Contenido |
|--------|----------|-------------|----------|-----------|
| Moodle LMS | campusonline.unir.net | OAuth2 form-post (crosscutting.unir.net) → sesskey | Akamai TLS fingerprint + JS challenge | Cursos, foros, módulos, LTI links |
| CMS Temario | cms.unir.net | LTI iframe launch from Moodle | LTI POST token | PDFs (temas), bloques, file IDs base64 |
| Panopto | unir.cloud.panopto.eu | LTI iframe + playlist | LTI session | Clases grabadas, MP4 download, UUID |
| IdP | crosscutting.unir.net | Form-post (email + contraseña) | CSRF estándar | OAuth2 redirect loop |

## Auth Flow

### Sistema 1: OAuth2 via crosscutting.unir.net (IdP)
1. Navegar a `https://campusonline.unir.net/my/` → redirect a crosscutting.unir.net
2. Rellenar form: `pUsuario`, `pContrasenia`
3. Redirect de vuelta → Moodle sets `MoodleSession*` cookie + `M.cfg.sesskey` en DOM

### Gotchas
- Cookie banner ("NO") debe cerrarse primero
- Akamai bot detection: TLS fingerprint + dynamic cookie → raw fetch falla, debe usarse agent-browser
- `sesskey` se resetea en cada page load, válido ~1 hora
- Extracción de sesskey: `M.cfg.sesskey` disponible en DOM como variable JS global

### Sistema 2: LTI Launches
- Endpoint: `/mod/lti/launch.php?id=<cmid>&triggerview=0`
- Auth: cookies de sesión Moodle + firma LTI 1.0
- Resultado: redirect a sub-app (CMS o Panopto) con sesión LTI propia

### Gotchas LTI
- Sesión LTI es scoped al iframe (cms.unir.net, lm.lti.unir.net)
- Para llamadas AJAX posteriores a Moodle → "bounce back" a `/my/` para resetear CORS origin
- Panopto: auto-resume del video anterior → hacer click en "Menú" para volver al hub

### Sistema 3: Moodle AJAX
- Endpoint: `/lib/ajax/service.php?sesskey=<sesskey>&info=<methodname>`
- Body: `[{ index: 0, methodname: "<name>", args: {...} }]`
- Response: array → `[{ error: bool, data: ... }]`
- `invalidsesskey` → sesión expirada

## Headers críticos

```
POST /lib/ajax/service.php?sesskey=<token>&info=<method>
Content-Type: application/json
Cookie: MoodleSession*=...
```

CMS file download:
```
GET /file/<base64-id>/esl-ES
Cookie: <LTI session cookies>
```

Panopto MP4:
```
GET /Panopto/Podcast/Download/<uuid>.mp4?mediaTargetType=videoPodcast
```

## Endpoints descubiertos

| Sistema | Endpoint | Método | Response |
|---------|----------|--------|----------|
| Moodle | `/my/` | GET | HTML + M.cfg.sesskey en DOM |
| Moodle | `/lib/ajax/service.php` | POST | JSON array `[{ error, data }]` |
| Moodle | `/mod/forum/view.php?id=<cmid>` | GET | HTML (forum discussions, `tr.discussion[data-discussionid]`) |
| Moodle | `/mod/forum/discuss.php?d=<id>` | GET | HTML (posts, `article.forum-post-container`) |
| Moodle | `/mod/lti/launch.php?id=<cmid>&triggerview=0` | GET | Form POST a sub-app |
| CMS | `/lti/hub/4/<courseId>/<shortName>?uuid=...` | GET | HTML (tabs, temas con `/file/` links) |
| CMS | `/file/<base64-id>/esl-ES` | GET | Binary (PDF) |
| Panopto | `/Panopto/Podcast/Download/<uuid>.mp4` | GET | Binary (MP4) |

Métodos Moodle AJAX descubiertos:
- `core_course_get_courses` — listar cursos matriculados
- `core_course_get_contents` — módulos y secciones del curso

## Anti-bot / Protecciones

- **Akamai EdgeGuard**: TLS fingerprint + dynamic cookie — raw Node fetch falla
- **Workaround**: agent-browser (headless Chromium) pasa el handshake TLS nativo
- **Moodle CSRF**: `sesskey` requerido para AJAX — embebido en DOM
- Headers estándar del browser requeridos: `Accept`, `Referer`, `sec-fetch-*`

## Notas críticas de implementación

- **CMS Temario tabs**: todos los tabs son server-rendered en un solo HTML → parsear primer dump, no hacer clicks
- **Panopto playlist**: UUIDs de videos NO están en HTML inicial como data-attributes — requiere click en `<li>` de `.video-playlist-list` + wait 2.5s + re-leer iframe
- **LTI cookie jar**: agent-browser mantiene cookies por `--session <name>` → CLI reutiliza sesión entre comandos
- **Temario fallback**: si no hay headers "TEMA N." → colectar todos los `/file/` links y asignar números secuenciales
- **Forum timestamps**: `<time datetime="...">` puede no estar presente en Moodle 4.1.x → fallback a parsear texto
