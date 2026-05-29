# klipso_reverse

Fábrica de CLIs de extracción de APIs privadas. Cada CLI expone un target como comandos de terminal + servidor MCP para Claude Code.

Stack invariante: **Bun + TypeScript strict + Zod + Hono + MCP SDK + cligentic**.

## Contenido

- [`_knowledge/`](./_knowledge/) — base de conocimiento central
  - [`README.md`](./_knowledge/README.md) — índice de targets + árbol de decisión de tipos
  - [`targets/`](./_knowledge/targets/) — RESEARCH.md por cada target (auth, endpoints, gotchas)
  - [`framework/`](./_knowledge/framework/) — proceso completo para nuevos CLIs (TEMPLATE.md + TYPES.md)
  - [`cligentic.md`](./_knowledge/cligentic.md) — 22 bloques copy-paste + cómo instalar desde monorepo
  - [`best-practices.md`](./_knowledge/best-practices.md) — patrones concretos de producción
- [`Cli-propios/`](./Cli-propios/) — CLIs propios en producción
- [`cligentic/`](./cligentic/) — fuente de los 22 bloques cligentic
- [`_docs/`](./_docs/) — metodología consolidada original

## Targets documentados

| Target | Tipo | Auth | Anti-bot | CLI |
|--------|------|------|----------|-----|
| Rappi | A | Bearer token (Android UA) | app-version header | [rappi-cli](./rappi-cli/) |
| Uber Eats | A | Session cookies (macOS UA) | CSRF hardcoded | [ubereats-cli](./ubereats-cli/) |
| WhatsApp Web | B | Browser session (QR) | AutomationControlled bypass | [whatsapp-cli](./whatsapp-cli/) |
| SUNAT | B | OAuth2 form-post + sesskey | Headless block, reCAPTCHA | [sunat-cli](./sunat-cli/) |
| UNIR | B | OAuth2 IdP + LTI | Akamai TLS fingerprint | [unir-cli](./unir-cli/) |
| Bancolombia | C | OAuth2 + device fingerprint | Imperva WAF | [bancolombia-cli](./bancolombia-cli/) |
| Spotify | D | OAuth2 PKCE (oficial) | Rate limiting | [spoti-cli](./spoti-cli/) |
| Trii | E | Bearer + sk2 cookie + RSC | Cloudflare WAF | [trii-cli](./trii-cli/) |
| Wiener | B+D | ASP form-post + Canvas PAT | Sin WAF adicional | [wiener-cli](./wiener-cli/) |
| Cineplanet | A | Público (cache) + usersession browser | Azure Front Door + APIM | [Cli-propios/cineplanet-cli](./Cli-propios/cineplanet-cli/) |
| Plaza Vea | B | Cookie `VtexIdclientAutCookie_plazavea` (VTEX) | Sin WAF notable | [Cli-propios/plazavea-cli](./Cli-propios/plazavea-cli/) |
| Interbank | C híbrido | ECIES P-256 + OTP email + Cookie JWT | Cloudflare WAF | [Cli-propios/interbank-cli](./Cli-propios/interbank-cli/) |

## Nuevo CLI — inicio rápido

```bash
# 1. Leer _knowledge/framework/TEMPLATE.md
# 2. Identificar tipo en _knowledge/framework/TYPES.md
# 3. Crear RESEARCH.md ANTES de codear
# 4. Scaffold + bloques cligentic desde monorepo:
REGISTRY="C:/Users/HP SUPPORT/klipso_reverse/cligentic/registry"
mkdir -p src/cli/{agent,foundation,platform,safety}
cp "$REGISTRY/agent/json-mode.ts" src/cli/agent/
# ... ver TEMPLATE.md §2.2 para lista completa
```

## Cómo contribuir

Ver [`_knowledge/framework/TEMPLATE.md`](./_knowledge/framework/TEMPLATE.md) — proceso completo: recon → scaffold → cligentic → MCP → checklist de publicación.
