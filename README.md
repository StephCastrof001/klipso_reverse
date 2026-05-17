# klipso_reverse

Knowledge base de reverse engineering para extracción de APIs privadas.

Metodología, taxonomía de targets, y documentación técnica de cada portal analizado.

## Contenido

- [`_knowledge/`](./_knowledge/) — base de conocimiento central
  - [`README.md`](./_knowledge/README.md) — índice de targets + árbol de decisión
  - [`targets/`](./_knowledge/targets/) — RESEARCH.md por cada target (auth, endpoints, gotchas)
  - [`framework/`](./_knowledge/framework/) — taxonomía de tipos y proceso para nuevos CLIs
- [`_docs/`](./_docs/) — metodología consolidada original

## Targets documentados

| Target | Tipo | Auth | Anti-bot |
|--------|------|------|----------|
| Rappi | A | Bearer token (Android UA) | app-version header |
| Uber Eats | A | Session cookies (macOS UA) | CSRF hardcoded |
| WhatsApp Web | B | Browser session (QR) | AutomationControlled bypass |
| SUNAT | B | OAuth2 form-post + sesskey | Headless block, reCAPTCHA |
| UNIR | B | OAuth2 IdP + LTI | Akamai TLS fingerprint |
| Bancolombia | C | OAuth2 + device fingerprint | Imperva WAF |
| Spotify | D | OAuth2 PKCE (oficial) | Rate limiting |
| Trii | E | Bearer + sk2 cookie + RSC | Cloudflare WAF |
| Wiener | B+D | ASP form-post + Canvas PAT | Sin WAF adicional |

## Cómo contribuir

Ver [`_knowledge/framework/TEMPLATE.md`](./_knowledge/framework/TEMPLATE.md) para el proceso completo de recon → CLI → documentación.
