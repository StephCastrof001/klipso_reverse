# Crafter Framework — Metodología Compartida

> Patrón identificado en rappi-cli (camilocbarrera) y sunat-cli (crafter-research)
> Fecha: 2026-03-30

---

## El Framework: "Live Recon → Document → Automate"

### Fase 1: Live Recon (con agent-browser via CDP)
- Abrir portal con sesión named: agent-browser --headed --session target open "https://..."
- Snapshot del DOM para descubrir elementos: agent-browser snapshot -i
- Interceptar requests de red para descubrir endpoints reales
- Ejecutar JS en contexto de la página para explorar

### Fase 2: Document (RESEARCH.md)
Artifact central: RESEARCH.md
- Portal Map (URLs, auth, CAPTCHA, contenido)
- Auth flows con bypass discoveries
- Workflows paso a paso con gotchas numerados
- Key URLs / endpoints descubiertos
- agent-browser tips específicos del portal

### Fase 3: Automate (CLI)
Arquitectura estandarizada:
- browser/client.ts — wrapper typed de agent-browser
- browser/cdp.ts — raw CDP WebSocket para edge cases (cross-origin iframes, input masks)
- commands/ — commander.js commands
- workflows/ — lógica de negocio (form automation)
- services/ — API calls cuando hay REST disponible
- validation/ — input hardening con Zod

---

## Stack técnico compartido

| Componente | rappi-cli | sunat-cli |
|---|---|---|
| Runtime | Bun | Bun |
| Lenguaje | TypeScript | TypeScript |
| Browser automation | Playwright (login) | agent-browser (todo) |
| Schema validation | Zod | Zod + JSON Schema |
| CLI framework | custom dispatcher | commander.js |

---

## Principios de diseño

1. Agent-first I/O: --json input + --output json/table/auto. Diseñado para AI agents, no solo humanos.
2. RESEARCH.md como memoria persistente: el conocimiento vive en el repo.
3. --dry-run obligatorio: cualquier operación mutante tiene preview.
4. Audit trail: operaciones loggeadas en ~/.app/audit/YYYY-MM-DD.jsonl
5. Session management: sesiones nombradas para mantener estado entre comandos.
6. CDP como escape hatch: cuando agent-browser no puede (cross-origin iframes), raw CDP WebSocket resuelve.

---

## Qué hace único este framework

vs. scrapers tradicionales:
- Usa refs efímeros buscados por text/type — sobrevive cambios de DOM (no hardcodea XPaths)
- Documenta TODO en RESEARCH.md — reproducible por futuros agentes AI
- Raw CDP para casos imposibles — sin bloqueos técnicos
- I/O estructurado — usable por humanos y por AI igualmente

---

## Aplicabilidad a otros targets tributarios LATAM

| Target | Complejidad | Anti-bot | API Alternativa |
|---|---|---|---|
| SUNAT Peru | Alta | Headless block + reCAPTCHA | OAuth2 parcial (no RHE) |
| SAT México | Alta | CAPTCHA complejo | Parcial |
| AFIP Argentina | Alta | Similar a SUNAT | Parcial |
| SRI Ecuador | Media | Similar | Limitada |
| MercadoLibre | Baja | Ninguno relevante | API oficial completa |
| Rappi | Media | Headers específicos | No existe |

