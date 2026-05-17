# Crafter Framework - Como replicarlo
# Inspiracion para construir nuevos targets
# Fecha: 2026-03-30

---

## Estructura base del framework

crafter-framework/
  template/
    RESEARCH.md          <- template con secciones fijas (ver abajo)
    src/
      browser/
        client.ts        <- wrapper Playwright (copiar de rappi-cli o sunat-cli)
        cdp.ts           <- raw CDP WebSocket (copiar de sunat-cli)
      ui/                <- copiar completo de rappi-cli
      config.ts          <- copiar de rappi-cli, cambiar schema Zod
    SKILL.md             <- para integrar con Claude Code
  targets/
    sunat-pe/            <- YA EXISTE (crafter-research/sunat-cli)
    rappi-co/            <- YA EXISTE (camilocbarrera/rappi-cli)
    meli-pe/             <- POR HACER (tiene API oficial, mas simple)
    sat-mx/              <- POR HACER
    afip-ar/             <- POR HACER
    bcp-pe/              <- POR HACER (cuenta propia, dificil)

Costo de arranque: 1-2 dias para armar el scaffold base.
Costo por target nuevo: solo el tiempo de recon del portal especifico.

---

## Template de RESEARCH.md (secciones fijas)

# [NOMBRE PORTAL] Research
Findings from reverse-engineering [portal] ([fecha]).

## Portal Map
| Portal | URL | Auth | CAPTCHA | Contiene |

## Auth Flow
Como funciona el login. Bypasses descubiertos.

## Tecnica de Recon Usada
Que herramienta, que sesion, que comandos especificos.

## Workflows
Cada flujo importante paso a paso con gotchas numerados.

## Endpoints Descubiertos
| Endpoint | Metodo | Payload | Respuesta |

## Edge Cases y Soluciones
Problemas encontrados y como se resolvieron.

## Key URLs
Tabla de URLs importantes con su proposito.

## File Locations
Donde guarda config, sessions, audit.

## agent-browser / Playwright Tips
Comandos especificos que funcionan para este portal.

---

## Fuentes de inspiracion (repos existentes)

rappi-cli (camilocbarrera/rappi-cli):
  - Mejor ejemplo de Tipo A (API privada)
  - Reutilizar: http.ts, config.ts, ui/, dispatcher
  - NO reutilizar: constants.ts (Colombia-only), formatters (Rappi-specific)
  - Leccion clave: el cliente HTTP es el corazon, mantenerlo limpio

sunat-cli (crafter-research/sunat-cli):
  - Mejor ejemplo de Tipo B (Form-based)
  - Reutilizar: browser/client.ts, browser/cdp.ts, validation/, audit
  - NO reutilizar: workflows/ (SUNAT-specific), schemas/ (SUNAT-specific)
  - Leccion clave: RESEARCH.md primero, siempre. CDP es el escape hatch.

---

## Cuando vale la pena crear el framework vs usar directamente

1 target -> usa el repo existente directamente (sunat-cli o rappi-cli)
2-3 targets -> el framework se amortiza rapido
LATAM completo -> el framework ES el negocio, los targets son el inventario

---

## Checklist para un target nuevo

RECON:
  [ ] Abrir portal con agent-browser --headed --session <target>
  [ ] Mapear portales/subportales y sus URLs
  [ ] Identificar tipo de auth (OAuth2, session, token, OTP)
  [ ] Detectar anti-bot (headless detection, CAPTCHA, fingerprinting)
  [ ] Interceptar network requests para descubrir endpoints
  [ ] Snapshot de formularios para mapear field IDs
  [ ] Documentar gotchas (beforeunload, iframes, input masks)

DOCUMENT:
  [ ] Crear RESEARCH.md con todas las secciones del template
  [ ] Incluir bypass discoveries con codigo de ejemplo
  [ ] Documentar edge cases y sus soluciones

AUTOMATE:
  [ ] Copiar scaffold base (browser/client, cdp, ui, config)
  [ ] Implementar auth flow especifico del target
  [ ] Implementar workflows principales
  [ ] Agregar validation/ para inputs del dominio
  [ ] Agregar --dry-run en todas las mutaciones
  [ ] Agregar audit trail
  [ ] Crear SKILL.md para integracion con Claude

TEST:
  [ ] Smoke test: login -> operacion basica -> logout
  [ ] Dry-run de todas las mutaciones
  [ ] Verificar --output json funciona para todos los comandos
---

## Plataformas conocidas

### VTEX (Plaza Vea, Wong, Metro, Tottus, Vivanda)
- **itemId != productId** — `cart add` necesita `itemId` (SKU), no `productId`
- **1 item por producto** — las presentaciones (1kg, 2kg, 5kg) son productos separados, no `items[]` del mismo producto
- **Stock en**: `sellers[0].commertialOffer.AvailableQuantity` — siempre presente
- **3 precios en**: `sellers[0].commertialOffer.Price` (venta), `.ListPrice` (tachado), y precio tarjeta via `Teasers[].Effects.Parameters.PromotionalPriceTableItemsDiscount`
- **Auth**: cookie `VtexIdclientAutCookie_<store>` (ej: `VtexIdclientAutCookie_plazavea`), TTL ~24h
- **Sin WAF**: ninguno de los supermercados VTEX Peru tiene Cloudflare ni Akamai
- **Endpoint busqueda**: `GET /api/catalog_system/pub/products/search?ft={query}&_from=0&_to=9` (sin auth)
- **Endpoint carrito**: `GET /api/checkout/pub/orderForm` (requiere auth)
- **Cookie de carrito**: `checkout.vtex.com` vincula el carrito al browser — para checkout headless usar `context.addCookies()`
- **Nombre tarjeta de descuento**: en el sistema VTEX aparece como `TARJETA OH - PVEA`, aunque en la web se llame Tarjeta SIP

### Rappi
- **Auth**: Bearer token (extraido del browser, no OAuth)
- **Fragil**: `app-version` header usa commit hash hardcodeado, rompe con cada deploy
- **add-to-cart**: requiere `storeId` + `productId` + toppings opcionales
- **Toppings**: son variantes obligatorias u opcionales del producto (no son `items[]`)
- **Colombia-only**: BASE_URL y moneda hardcodeados en COP

---

## Lecciones generales de plaza-vea-cli

- **readline con stdin piped no funciona en bash tool** — `readline.question()` cierra el proceso cuando stdin llega a EOF antes de que los fetches terminen. Solo funciona en terminal real con TTY
- **El flujo de seleccion interactiva resuelve el problema de IDs sin gastar tokens** — en vez de pedir al usuario que copie IDs, mostrar tabla numerada y que elija. El itemId correcto se toma internamente
- **AvailableQuantity siempre presente en VTEX** — campo confiable para mostrar stock, nunca viene null
- **Python como escape hatch para escribir archivos en paths con espacios** — hooks de seguridad en Claude Code fallan con `HP SUPPORT` en el path. Usar `python3 - << PYEOF ... PYEOF` con heredoc de shell
- **Git desde el primer archivo** — sin git, iteraciones rotas no tienen rollback. Primer commit antes de cualquier logica
- **Cache-first para endpoints de historial** — `/oms/user/orders` puede tener 65+ ordenes. Guardar en JSON local con TTL 6h evita 60+ llamadas repetidas

---

## Checklist CLI (buenas practicas para cualquier CLI de e-commerce/finanzas)

### UX de seleccion
- [ ] Comando `buy` / `shop` con seleccion interactiva numerada — nunca exponer IDs al usuario
- [ ] Mostrar todos los precios disponibles en la tabla (precio venta, precio tachado, precio tarjeta)
- [ ] Mostrar stock por producto — filtrar SIN STOCK visualmente
- [ ] Modo continuo: despues de agregar, preguntar si quiere seguir comprando (loop hasta `cart show`)

### Arquitectura
- [ ] Cache-first para endpoints de historial (ordenes, transacciones) — TTL 6h en JSON local
- [ ] `itemId` / `skuId` separado de `productId` — verificar cual necesita cada endpoint antes de construir
- [ ] `--output json` en todos los comandos de lectura — para MCP y scripts
- [ ] `--dry-run` en todos los comandos mutantes — para verificar sin ejecutar

### Git y flujo de trabajo
- [ ] `git init` + primer commit antes de escribir cualquier logica
- [ ] Rama `dev` desde el inicio — nunca commitear directo a `main`
- [ ] Commit por cada comando que pasa smoke test — rollback granular
- [ ] Convencion: `feat:` / `fix:` / `docs:` / `chore:`

### Documentacion
- [ ] `RESEARCH.md` antes de codear — endpoints, auth, gotchas
- [ ] `CLAUDE.md` con stack, comandos, estructura y notas tecnicas
- [ ] `CHANGELOG.md` con historial por version
- [ ] `CONTEXT.md` para AI agents — como operar el CLI sin leer el codigo
