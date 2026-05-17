# Universidad Norbert Wiener — Research

Tipo: B+D (dual-session) — Intranet ASP.NET (form-post + CSRF) + Canvas LMS (PAT)
Recon via: análisis de código (wiener-cli)

## Portal Map

| Portal | URL Base | Auth Method | Anti-bot | Contenido |
|--------|----------|-------------|----------|-----------|
| Wienernet Intranet | intranet.uwiener.edu.pe | Form-post + CSRF token + ASPSESSIONID cookie | ASP.NET session, sin WAF adicional | Notas, horario, asistencia, plan, trámites, pagos |
| Canvas LMS | campus.uwiener.edu.pe | PAT (Personal Access Token) — generado manualmente | Bearer token + rate limiting | Cursos, tareas, anuncios, archivos, módulos |

## Auth Flow

### Sistema 1: Intranet ASP.NET (form-post + CSRF)

Flujo:
1. `GET /sso.asp` → captura CSRF token (regex: `name="csrfToken" value="..."`) + cookie `ASPSESSIONID*`
2. `POST /login/dev/autenticate.asp` con CSRF:
   ```
   pUsuario=<email>
   pContrasenia=<pwd>
   pPerfil=A|D|P    (alumno/docente/padre)
   pInstitucion=51   (código Wiener, siempre fijo)
   csrfToken=<token>
   ```
   Response: `{ estado: "0"|"1"|"9", mensaje?: string, action: "..." }`
   - `estado 0` → credenciales inválidas
   - `estado 1` → flujo normal, `action` contiene path a ValidaAcceso.asp
   - `estado 9` → POST directo a `action`
3. `POST /ValidaAcceso.asp` (o `action/ValidaAcceso.asp`):
   ```
   lgnUserName=<email>
   lgnPassword=<pwd>
   lgnddlPerfiles=A|D|P
   lgnddlInstituciones=51
   ```
   Response: 302 → `/Alumno/SiguNet.asp` + nuevo `ASPSESSIONID*` cookie

### Gotchas Intranet
- Cookie `ASPSESSIONID` tiene sufijo dinámico aleatorio: `ASPSESSIONIDCABCDEFG` — parsear dinámicamente
- `Referer` obligatorio en ValidaAcceso.asp: `https://intranet.uwiener.edu.pe/sso.asp`
- `pPerfil` debe coincidir con tipo de usuario o falla con estado=0
- `pInstitucion` siempre "51" — código Wiener
- Sesión dura ~20-30 min de inactividad
- Expiración detectada si response es HTML (no JSON) o redirige a `/sso.asp`

### Sistema 2: Canvas LMS (PAT manual)
- PAT generado en: Account → Settings → Approved Integrations → Create New Access Token
- Token format: `1234~aBcDeF...` (largo alfanumérico)
- Almacenamiento: Keychain macOS o secrets file
- No expira — solo revocación manual en Canvas UI
- Rate limit: header `x-canvas-meta` con `rlr=<remaining>` + `rce=<cost>`
- Wiener restringe algunos accesos → 403 + `"usuario no autorizado"` (no es problema del token)

## Headers críticos

Intranet `POST /login/dev/autenticate.asp`:
```
Content-Type: application/x-www-form-urlencoded
X-Requested-With: XMLHttpRequest
Referer: https://intranet.uwiener.edu.pe/sso.asp
Cookie: ASPSESSIONID*=...
```

Intranet `POST /ValidaAcceso.asp`:
```
Content-Type: application/x-www-form-urlencoded
Cookie: ASPSESSIONID*=...
```

Canvas API:
```
Authorization: Bearer <pat>
Accept: application/json+canvas-string-ids
```

## Endpoints descubiertos

### Intranet (HTML scraping con cheerio)

| Endpoint | Method | Datos extraídos |
|----------|--------|-----------------|
| `/sso.asp` | GET | CSRF token en HTML form |
| `/login/dev/autenticate.asp` | POST | `{ estado, action }` JSON |
| `/ValidaAcceso.asp` | POST | 302 + nuevo `ASPSESSIONID*` |
| `/Alumno/Datosacademicos/notas/NOTAS.asp` | GET | Tabla de notas: cursos[], periodos[] |
| `/Alumno/Datosacademicos/horario/HORARIO.asp` | GET | Horario: clases[], hoy |
| `/Alumno/Reportes/asistencia/ASISTENCIA.asp` | GET | Asistencia: porcentaje, fechas |
| `/Alumno/Datosacademicos/plandeestudios/PLAN.asp` | GET | Plan de estudios: semestres[], cursos[] |
| `/Alumno/tesoreria/pagos/PAGOS.asp` | GET | Pagos: pagos[], deudas[] |

### Canvas REST API v1

| Endpoint | Method | Datos |
|----------|--------|-------|
| `/api/v1/courses` | GET | courses[] |
| `/api/v1/courses/<id>/assignments?include[]=submission` | GET | assignments[], submissions[] |
| `/api/v1/users/self/enrollments` | GET | enrollments[] |
| `/api/v1/courses/<id>/announcements` | GET | announcements[] |
| `/api/v1/courses/<id>/discussion_topics` | GET | discussions[] |
| `/api/v1/courses/<id>/files` | GET | files[] (paginado) |
| `/api/v1/courses/<id>/modules` | GET | modules[] |
| `/api/v1/courses/<id>/assignments/<id>/submissions/self/files` | POST | `{ upload_url, upload_params }` |

File upload Canvas: proceso 2 pasos:
1. POST al endpoint anterior → obtener `upload_url` (S3 pre-signed)
2. POST a `upload_url` con form-data multipart

## Anti-bot / Protecciones

- **Intranet ASP.NET**: CSRF token (single-use), session cookie scoping, sin WAF adicional
- **Canvas API**: rate limiting via `x-canvas-meta` header (`rlr < 50` → `RateLimitedError`), sin fingerprinting
- Sin Akamai/Cloudflare detectado en ninguno de los dos portales

## Notas críticas de implementación

- **Dual-session model**: dos stores de sesión independientes — intranet y Canvas no comparten estado
- **Intranet toda HTML**: no hay JSON API — cheerio para parsear tablas HTML
- **Course aliasing**: Canvas IDs son numéricos largos → CLI soporta aliases guardados en config
- **Submission types validation**: verificar `submission_types` array antes de submit → solo los tipos permitidos por el curso
- **Rate limiting backoff**: CLI lanza error si `rlr < 50`, no tiene backoff automático — usuario debe reintentar
- **Perfil detection**: si `pPerfil` falla → error estado=0; CLI infiere desde response o config del usuario
