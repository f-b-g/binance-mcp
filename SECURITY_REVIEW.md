# Binance MCP Security & Trustworthiness Review

## Scope & Methodology
- Reviewed source TypeScript implementation (`src/index.ts`) for the MCP server entry point and tool definitions.
- Examined build/runtime configuration (`package.json`, `tsconfig.json`, `Dockerfile`).
- Looked for non-source assets or hidden scripts and verified no additional AGENTS instructions were present.
- Assessed external dependencies and network interactions for potential security or privacy concerns.

## Architecture Overview
- Single entry point (`src/index.ts`) that instantiates an `McpServer`, registers tool handlers, and serves over stdio. No background daemons or network listeners are created.【F:src/index.ts†L12-L441】
- Each tool handler performs a single HTTPS GET request against the public Binance REST API base URL (`https://api.binance.com`). Responses are stringified and returned to the MCP client without post-processing.【F:src/index.ts†L8-L399】
- Optional proxy support reads `HTTP_PROXY`/`HTTPS_PROXY`; values are parsed via `URL` and passed to axios. No other environment variables are consumed except `BINANCE_API_KEY` for historical trades where an API key is required.【F:src/index.ts†L10-L411】

## Dependency Review
- Runtime dependencies limited to:
  - `@modelcontextprotocol/sdk` for MCP primitives.
  - `axios` for HTTP requests.
  - `zod` for declarative tool input validation.
- Build toolchain uses `typescript` and `ts-node`; Dockerfile builds artifacts with `npm ci` and runs compiled output with Node 18+ on Alpine Linux.【F:package.json†L1-L46】【F:Dockerfile†L1-L33】
- No custom scripts beyond `npm run build/start`; no postinstall hooks due to `--ignore-scripts` usage in Docker build.【F:Dockerfile†L10-L27】

## Data Flow & Privacy Considerations
- Tool inputs are validated via `zod`, ensuring required symbols/intervals are strings or numbers as expected, reducing malformed request risk.【F:src/index.ts†L16-L398】
- Requests are outbound-only to Binance endpoints. No local file access, subprocess execution, or arbitrary command invocation is performed.
- Returned data mirrors Binance API responses. No sensitive data is stored or logged aside from console messages on server start/stop or error reporting.【F:src/index.ts†L425-L438】
- Optional `BINANCE_API_KEY` is inserted into the `X-MBX-APIKEY` header only when historical trades are requested; the key is not logged or persisted.【F:src/index.ts†L85-L95】

## Error Handling & Resilience
- Each tool handler wraps axios calls in `try/catch` and returns structured MCP errors when requests fail, preventing unhandled promise rejections from crashing the process.【F:src/index.ts†L20-L398】
- Global `main()` uses `process.on` handlers to close transports gracefully on SIGINT/SIGTERM and exits with non-zero code on fatal errors.【F:src/index.ts†L414-L441】

## Identified Risks / Areas for Improvement
1. **Proxy Handling:** When no proxy is configured, `getProxy()` returns an empty object; axios treats `{}` as a proxy definition and may attempt to connect to `http://:0`. Returning `undefined` when no proxy is set would avoid potential connection failures or confusing error messages. This is a reliability concern rather than a security issue.【F:src/index.ts†L403-L412】
2. **Rate Limiting Awareness:** The server does not enforce rate limiting or caching. Calling clients must ensure compliance with Binance API rate limits to avoid IP bans. Consider documenting expected usage patterns.
3. **Input Validation for Lists:** `symbols` arrays are JSON-stringified without additional validation (length, symbol format). While this matches Binance API expectations, adding regex validation could prevent accidental malformed requests.【F:src/index.ts†L236-L388】

## Conclusion
- The codebase exhibits transparent, minimal logic focused on proxying Binance REST endpoints. No hidden network calls, credential harvesting, or persistence mechanisms were observed.
- With the minor reliability improvement around proxy handling, the MCP server appears trustworthy for fetching public Binance market data, assuming the environment supplies any required API keys securely and clients respect Binance API usage policies.
