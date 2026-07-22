# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm install              # install deps
npm run build            # tsc → dist/
npm run dev              # build + run server on stdio (for local testing)
npm run start            # run already-built server (node dist/index.js)
npm run auth             # one-time OAuth CLI: opens browser, stores tokens
npm run dashboard        # build + run scripts/generate-dashboard.ts + open dashboard.html
```

There is no test suite, linter, or formatter configured. `tsc` (via `npm run build`) is the only static check.

Environment: `LINKEDIN_CLIENT_ID` and `LINKEDIN_CLIENT_SECRET` are required (loaded via `dotenv` from `.env`, or set by the MCP host in `claude_desktop_config.json`). Optional `LINKEDIN_REDIRECT_URI` (default `http://localhost:3000/callback`) and `TOKEN_STORAGE_PATH` (default `~/.linkedin-ads-mcp/tokens.json`).

## Architecture

**MCP server over stdio.** `src/index.ts` instantiates `@modelcontextprotocol/sdk` `Server`, registers `ListToolsRequestSchema` and `CallToolRequestSchema` handlers, and connects a `StdioServerTransport`. The host (Claude Desktop / Claude Code) spawns `node dist/index.js` and talks JSON-RPC over stdin/stdout — anything written to stdout that isn't a protocol frame breaks the connection, so all logs go to `console.error` (stderr).

**Three layers:**

1. `src/auth/` — `LinkedInOAuth` runs a short-lived Express server on the redirect-URI port to capture the auth code; `TokenStore` persists `{access_token, refresh_token, expires_at}` to disk (mode 0600, parent dir 0700) and transparently refreshes when within 5 minutes of expiry.
2. `src/lib/linkedin-api.ts` — `LinkedInApiClient` is the single chokepoint for every LinkedIn call. It pins `LINKEDIN_VERSION = '202601'` and `X-Restli-Protocol-Version: 2.0.0`, handles 429 with `Retry-After`-aware exponential backoff (3 attempts), and parses the empty-body `201 Created` / `204` cases by reading the `x-restli-id` header back as `{ id }`. Query-string building is custom (not `URLSearchParams`) because LinkedIn expects unencoded `,` and `:` inside `fields` and `dateRange`, and array params wrapped as `List(a,b,c)`. Write operations pass `restliMethod` so the client sets `X-RestLi-Method` (e.g. `PARTIAL_UPDATE`).
3. `src/tools/` — one file per domain (`accounts`, `performance`, `demographics`, `conversions`, `analytics`, `campaign-management`). Each file exports pairs: a `Tool` schema (`xxxTool`) and a `handleXxx(client, args)` function. They're pure adapters between MCP `arguments` and `LinkedInApiClient` methods.

**Adding a tool requires three edits in `src/index.ts`:** import the `Tool` + handler, push the `Tool` into the `TOOLS` array (controls discovery), and add a `case` in the `CallToolRequestSchema` switch (controls dispatch). Missing either of the latter two silently breaks the tool. The handler dispatch also short-circuits with an `isError` response if `tokenStore.hasValidToken()` is false — there is no per-tool auth check.

**LinkedIn API quirks the code already encodes (don't re-discover them):**
- Default metric sets live as constants at the top of `linkedin-api.ts` (`DEFAULT_PERFORMANCE_METRICS`, `DEFAULT_CREATIVE_METRICS`, `VIDEO_METRICS`, `LEAD_GEN_METRICS`, `REACH_METRICS`). Max 20 metrics per request — adding to these can push other requests over the limit.
- `audiencePenetration` is only returned natively for date ranges ≤92 days; longer ranges fall back to client-side computation.
- `dateRange` and `fields` query params must not URL-encode their inner punctuation — the custom encoder in `request()` is intentional.
- Create endpoints return `201` with the new entity ID in the `x-restli-id` response header, not in the body.

**OAuth scopes** (`src/auth/oauth.ts`): `rw_ads`, `r_ads_reporting`, `r_organization_social`, `w_organization_social`. Changing scopes requires re-running `npm run auth` (delete `~/.linkedin-ads-mcp/tokens.json` first).

**`src/scripts/generate-dashboard.ts`** is a standalone Node script (not part of the MCP server) that pulls performance data via the same `LinkedInApiClient` and writes a self-contained `dashboard.html`. It uses the same on-disk tokens, so `npm run auth` must have been completed.
