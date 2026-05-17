rappi-cli vs sunat-cli

Tipo A (rappi): API REST privada - HTTP puro post-login
Tipo B (sunat): Forms HTML - browser automation permanente

PROS rappi: stateless, serverless, http.ts reutilizable, MCP support
CONTRAS rappi: app-version fragil, sin refresh token, Colombia-only

PROS sunat: CDP escape hatch, headed mode, audit trail, agent-first IO
CONTRAS sunat: sesion 20min, refs efimeros, requiere Chrome GUI

Marco comun: recon->document->automate, Bun+TS+Zod, UI layer, local config
