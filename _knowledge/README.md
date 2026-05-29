# Klipso Reverse — Knowledge Base

Base de conocimiento de reverse engineering para extracción de APIs privadas.
Cada CLI resuelve un target específico. El valor está en el RESEARCH.md de cada target.

---

## Targets documentados

| Target | Tipo | Auth Method | Anti-bot | CLI | RESEARCH |
|--------|------|-------------|----------|-----|----------|
| **Rappi** | A | Bearer token (Android UA) | app-version header | [rappi-cli](../rappi-cli/) | [targets/rappi.md](targets/rappi.md) |
| **Uber Eats** | A | Session cookies (macOS UA) | x-csrf-token hardcoded | [ubereats-cli](../ubereats-cli/) | [targets/ubereats.md](targets/ubereats.md) |
| **WhatsApp Web** | B | Browser session (QR scan) | AutomationControlled bypass | [whatsapp-cli](../whatsapp-cli/) | [targets/whatsapp.md](targets/whatsapp.md) |
| **SUNAT** | B | OAuth2 form-post → sesskey | Headless block, reCAPTCHA bypass | [sunat-cli](../sunat-cli/) | [targets/sunat.md](targets/sunat.md) |
| **UNIR** | B | OAuth2 form-post (IdP) + LTI | Akamai TLS fingerprint | [unir-cli](../unir-cli/) | [targets/unir.md](targets/unir.md) |
| **Bancolombia** | C | OAuth2 + device fingerprint | Imperva WAF, headless block | [bancolombia-cli](../bancolombia-cli/) | [targets/bancolombia.md](targets/bancolombia.md) |
| **Spotify** | D | OAuth2 PKCE (official) | Rate limiting solo | [spoti-cli](../spoti-cli/) | [targets/spotify.md](targets/spotify.md) |
| **Trii** | E | Bearer + sk2 cookie (2FA) + RSC | Cloudflare WAF | [trii-cli](../trii-cli/) | [targets/trii.md](targets/trii.md) |
| **Wiener** | B+D | ASP form-post + Canvas PAT | Sin WAF adicional | [wiener-cli](../wiener-cli/) | [targets/wiener.md](targets/wiener.md) |
| **v0** | D | API Key (Bearer official) | Rate limiting + intent tokens | [v0-cli](../v0-cli/) | [targets/v0.md](targets/v0.md) |
| **Interbank** | C híbrido | ECIES P-256 + OTP email + Cookie JWT | Cloudflare WAF (Node.js bloqueado) | [Cli-propios/interbank-cli](../Cli-propios/interbank-cli/) | [targets/interbank.md](targets/interbank.md) |
| **Cineplanet** | A | Público (cache) + usersession browser (gated) | Azure Front Door + APIM (403 gate) | [Cli-propios/cineplanet-cli](../Cli-propios/cineplanet-cli/) | [targets/cineplanet.md](targets/cineplanet.md) |
| **Plaza Vea** | B | Cookie `VtexIdclientAutCookie_plazavea` (Playwright browser) | Sin WAF notable | [Cli-propios/plazavea-cli](../Cli-propios/plazavea-cli/) | [targets/plazavea.md](targets/plazavea.md) |

---

## Framework

| Doc | Descripción |
|-----|-------------|
| [framework/TYPES.md](framework/TYPES.md) | Taxonomía de tipos A-E + B+D — cómo identificar el tipo de target |
| [framework/TEMPLATE.md](framework/TEMPLATE.md) | Proceso completo para un nuevo CLI: recon → scaffold → checklist |
| [best-practices.md](best-practices.md) | Patrones concretos con fuentes: error handling, session TTL, audit, cache-first, dry-run, Hono server |
| [_docs/MASTER_METHODOLOGY.md](../_docs/MASTER_METHODOLOGY.md) | Metodología consolidada original (7 repos) |

---

## Meta-framework

| Repo | Descripción | Doc |
|------|-------------|-----|
| [cligentic](../cligentic/) | 22 bloques copy-paste para CLI (shadcn model) — usar en todos los nuevos CLIs | [cligentic.md](cligentic.md) |

---

## Cómo agregar un nuevo target

1. Clonar en `klipso_reverse/<nombre>-cli/`
2. Leer `framework/TEMPLATE.md` → seguir el proceso
3. Crear `RESEARCH.md` en el repo del CLI (recon primero, código después)
4. Copiar `RESEARCH.md` → `_knowledge/targets/<nombre>.md`
5. Agregar fila a esta tabla

---

## Árbol de decisión: identificar tipo de target

```
¿Tiene Akamai/PerimeterX/MFA obligatoria?
  → Tipo C (bancolombia)

¿Next.js con Server Actions (header Next-Action)?
  → Tipo E (trii)

¿Dos backends con auth diferente?
  → Tipo B+D (wiener)

¿API Key o OAuth2 PKCE documentado?
  → Tipo D (spotify, v0)

¿Login devuelve Bearer token (SPA + fetch)?
  → Tipo A (rappi, ubereats)

¿Login es <form> HTML con POST / Moodle / portal legacy?
  → Tipo B (sunat, whatsapp, unir)
```
