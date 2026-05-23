# CLI vs MCP — Ventajas y cobertura por repo

## Cobertura actual

| CLI | CLI commands | MCP | REST server.ts | Generación |
|-----|-------------|-----|----------------|------------|
| rappi-cli | ✅ | ✅ | ✅ | Original crafter |
| bancolombia-cli | ✅ | ✅ | ✅ | Original crafter |
| ubereats-cli | ✅ | ✅ | ✅ | Original crafter |
| whatsapp-cli | ✅ | ✅ | ✅ | Original crafter |
| trii-cli | ✅ | ✅ | ✅ | Original crafter |
| v0-cli | ✅ | ✅ | ❌ | Railly nuevo |
| **sunat-cli** | ✅ | ❌ | ❌ | Original crafter |
| **spoti-cli** | ✅ | ❌ | ❌ | Original crafter |
| **wiener-cli** | ✅ | ❌ | ❌ | Railly nuevo |
| **unir-cli** | ✅ | ❌ | ❌ | Railly nuevo |
| interbank-cli (propio) | ✅ | ✅ (parcial) | ❌ | Propio |

---

## Ventajas CLI puro (terminal)

**Quién lo usa:** humano desde terminal.

| Ventaja | Detalle |
|---------|---------|
| Funciona standalone | Sin Claude Code, sin MCP, sin servidor |
| Scripteable en shell | `interbank transactions --output json \| jq '.[] \| select(.amount < 0)'` |
| `--output json` para piping | Cualquier herramienta puede consumir el output |
| Debuggeable fácil | `bun run src/commands/accounts.ts` directo |
| Sin overhead de protocolo | Invocación directa, sin stdio MCP handshake |

**Cuándo es suficiente:** uso personal desde terminal, scripts de automatización, cronjobs.

---

## Ventajas MCP (Model Context Protocol)

**Quién lo usa:** Claude Code como agente.

| Ventaja | Detalle |
|---------|---------|
| Claude puede llamarlo como tool | `use_mcp_tool("interbank", "get_accounts")` sin que el usuario escriba nada |
| Contexto persistente en conversación | Claude lee cuentas, analiza movimientos, sugiere en el mismo hilo |
| Input validado con schema JSON | Claude no puede pasar argumentos mal formados |
| Output estructurado siempre | Claude recibe JSON, no texto para parsear |
| Composición con otros tools | `get_transactions` → `analyze_spending` → `draft_budget` en secuencia |
| Descubrimiento automático | `list_tools` expone qué puede hacer el CLI sin leer docs |

**Cuándo es indispensable:** cuando Claude Code necesita leer datos bancarios para responder preguntas, hacer análisis, o automatizar flujos sin intervención humana.

---

## Ventajas REST server.ts

**Quién lo usa:** cualquier cliente HTTP (n8n, scripts Python, frontends).

| Ventaja | Detalle |
|---------|---------|
| Sin depender de Bun/Node local | Servidor corriendo → curl desde cualquier lado |
| Integración con n8n, Zapier, etc. | HTTP node en n8n llama a `GET /api/accounts` |
| Multi-cliente | Varios consumidores simultáneos |
| Expone mismos services que CLI | No hay lógica duplicada — mismo `services/` bajo `Bun.serve` o Hono |

---

## El stack completo (los 3 juntos)

```
┌─────────────────────────────────────────┐
│  humano en terminal → CLI commands      │  rappi, bancolombia, trii, ubereats
│  Claude Code        → MCP tools         │  rappi, bancolombia, trii, ubereats, v0, interbank
│  n8n / scripts HTTP → REST server.ts    │  rappi, bancolombia, trii, ubereats
└─────────────────────────────────────────┘
```

Los CLIs originales de crafter-station (rappi, bancolombia, ubereats, trii) tienen los 3.
Los de Railly (unir, wiener, v0) tienen CLI + cligentic pero les falta MCP y/o REST.

---

## Gaps de MCP — impacto real

### sunat-cli sin MCP
Sin MCP, Claude Code no puede consultar deudas tributarias de forma autónoma.
El CLI funciona perfectamente desde terminal, pero no se puede integrar en un flujo agentico:
```
# Imposible hoy:
claude: "revisa si tengo deudas en SUNAT y dime el monto"
→ debería llamar get_debt_summary() vía MCP → no existe
```

### spoti-cli sin MCP
Sin MCP, Claude Code no puede buscar playlists, reproducir, o analizar hábitos de escucha en conversación.

### wiener-cli / unir-cli sin MCP
Sin MCP, no se pueden hacer consultas de notas, tareas, o archivos académicos dentro de Claude Code.

---

## Patrón MCP en los CLIs que sí lo tienen

Todos los que tienen MCP usan **un solo tool con schema** (patrón camilocbarrera):
```typescript
// No son múltiples tools — es 1 tool que acepta subcommand
{ name: "rappi", description: "Rappi CLI...", input_schema: { subcommand, args } }
```

Excepción: **interbank-cli** usa múltiples tools individuales:
```typescript
{ name: "get_accounts" }
{ name: "get_transactions", input_schema: { account_id } }
{ name: "get_exchange" }
{ name: "login_status" }
```

El patrón multi-tool de interbank es **mejor para agentes**: Claude puede llamar `get_accounts` sin saber que es un subcommand de "interbank".

---

## Recomendación para nuevos CLIs propios

Orden de implementación:
1. **CLI commands** — funcionalidad core, human-first
2. **MCP tools** — inmediatamente después, para uso agentico
3. **REST server** — solo si se necesita integrar con n8n o clientes HTTP externos

Para bancos peruanos (IBK, BBVA): MCP es más valioso que REST porque el caso de uso principal es que Claude analice finanzas en conversación, no que n8n consuma datos.
