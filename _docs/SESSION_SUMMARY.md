# Resumen completo de la sesion
# Analisis rappi-cli, sunat-cli, framework Crafter, sectores
# Fecha: 2026-03-30

---

## 1. rappi-cli (camilocbarrera)

Que es: CLI que reverse engineera la API privada de Rappi Colombia
Tipo de RE: Tipo A - API-based

Flujo tecnico:
  Playwright abre rappi.com.co -> intercepta Bearer token del request a /ms/application-user/auth
  Guarda token + deviceId en .rappi-config.json
  Todos los requests siguientes son HTTP puro (sin browser)

Endpoints clave descubiertos:
  GET  /ms/application-user/auth              (whoami)
  POST /api/pns-global-search-api/v1/unified-search  (buscar)
  POST /api/restaurant-bus/stores/catalog-paged/home (restaurantes)
  PUT  /api/ms/shopping-cart/v2/{type}/store   (agregar al carrito)
  POST /api/ms/shopping-cart-proxy/{type}/checkout  (place order)

Headers fragiles (hardcodeados):
  app-version: e1de6be43aa29091011474615d7ac0810051c36a (commit hash del frontend)
  x-application-id: rappi-microfront-web/<mismo hash>
  accept-language: es-CO (solo Colombia)
  origin: https://www.rappi.com.co (solo Colombia)

Reutilizable: http.ts, config.ts, ui/, dispatcher pattern

---

## 2. sunat-cli (crafter-research)

Que es: CLI que automatiza SUNAT Peru (RHE + F616)
Tipo de RE: Tipo B - Form-based

Flujo tecnico:
  agent-browser abre SOL viejo (sin CAPTCHA)
  Navega a Nueva Plataforma via redirect con state OAuth (bypass de reCAPTCHA)
  Automatiza formularios via CDP
  Edge cases: raw CDP WebSocket para cross-origin iframes con Angular masks

Portales mapeados:
  SOL viejo: e-menu.sunat.gob.pe/cl-ti-itmenu/
  Nueva Plataforma: e-menu.sunat.gob.pe/cl-ti-itmenu2/
  e-renta DyP: e-renta.sunat.gob.pe (solo F709 Renta Anual)

Tecnica CDP clave:
  setInputValueInIframe() en cdp.ts
  Crea isolated worlds en cada frame para bypassear cross-origin restrictions
  Usa nativeInputValueSetter para bypassear Angular input masks

Diferencial vs rappi-cli:
  RESEARCH.md como artefacto central
  SKILL.md para integracion con Claude Code
  --dry-run obligatorio para mutaciones
  --output json/table/auto (agent-first IO)
  Audit trail JSONL diario
  Monorepo con Astro landing

---

## 3. El framework comun: Crafter RE Framework

Metodologia: recon -> document -> automate

Fase 1 - RECON (con agent-browser via CDP):
  Abrir portal real con sesion named
  Snapshot del DOM para descubrir elementos
  Interceptar network requests para descubrir endpoints
  Explorar con JS eval en contexto de la pagina

Fase 2 - DOCUMENT (RESEARCH.md):
  Portal map (URLs, auth, CAPTCHA)
  Auth flows con bypass discoveries documentados
  Workflows paso a paso con gotchas numerados
  Key URLs y agent-browser tips especificos

Fase 3 - AUTOMATE (CLI):
  browser/client.ts: wrapper typed de agent-browser
  browser/cdp.ts: raw CDP para edge cases
  commands/ + workflows/ + services/
  validation/ con Zod

Lo que homologa: filosofia de recon, RESEARCH.md, stack Bun+TS+Zod, UI layer
Lo que varia: tecnica especifica segun tipo de target (API vs Form vs Hybrid)

---

## 4. Matriz de tecnicas por tipo de target

Tipo A (API privada): interceptar token -> HTTP client puro
  Ejemplos: Rappi, Uber Eats, delivery apps

Tipo B (Forms HTML): browser automation permanente + CDP
  Ejemplos: SUNAT, SAT Mexico, AFIP Argentina, SRI Ecuador

Tipo C (Hybrid): API para lectura, forms para escritura
  Ejemplos: MercadoLibre, e-commerce con checkout protegido

Tipo D (API oficial): OAuth2 directo, sin reverse engineering
  Ejemplos: MercadoLibre MPE, Falabella, cualquier portal con API publica

---

## 5. MercadoLibre Peru (plan)

Tipo: D - API oficial documentada
Site ID: MPE
Base URL: https://api.mercadolibre.com
Auth: OAuth2 con refresh_token (6 meses)

Lo que se reutiliza de rappi-cli: http.ts, config.ts, ui/, dispatcher
Lo nuevo: OAuth2 flow, token refresh automatico, endpoints MELI

Caso de uso propuesto: monitor de precios Peru
  meli search, meli item, meli watch --below <precio>, meli check

---

## 6. Bancos - cuenta propia

Legal: 100% verde (tu propia cuenta)
Tecnico: mas dificil del framework (fingerprinting, behavioral analysis, OTP)

Opciones tecnicas:
  A: headed Chrome con perfil real (evita login si hay sesion activa)
  B: interceptar app movil via mitmproxy (endpoints mas limpios)
  C: Open Banking oficial (SBS Peru, BCP y BBVA ya tienen API)

Combo de mayor valor:
  sunat-cli + bank-cli = flujo completo freelancer peruano
  Cliente paga -> bank-cli detecta deposito -> sunat-cli emite RHE automaticamente

---

## 7. Sectores recomendados por orden de ataque

1. sunat-cli Peru -> productizar multi-tenant (base ya existe)
2. SAT Mexico -> mismo framework, mercado 5x mas grande
3. AFP Peru -> mismo usuario que SUNAT, complemento natural
4. SUNAT Aduanas -> segmento empresarial, ticket mas alto
5. Open Banking bancos -> cuando SBS Peru finalice regulacion

---

## 8. Modelo de negocio potencial

sunat-cli como SaaS:
  Target: 2M freelancers peruanos de 4ta categoria
  Pain: 15-30 min por declaracion mensual manual
  Modelos:
    SaaS self-service: S/. 29-49/mes
    API as a service: S/. 0.50 por RHE emitida
    White-label para contadores: S/. 200-500/mes

Lo que falta para productizar:
  Multi-tenant (hoy asume .env local)
  Credential vault seguro
  Webhook/callback para operaciones async
  UI para no-tecnicos
  Compliance legal (terminos, privacidad)