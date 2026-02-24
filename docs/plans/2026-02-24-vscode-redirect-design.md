# Design: `--vscode-redirect` Flag for Azure AD Relay Authentication

## Problem

The Azure AD app registration only has `https://vscode.dev/redirect` as a registered redirect URI. The current `mcp-remote` uses `http://localhost:<port>/oauth/callback` which is rejected by Azure AD with `AADSTS50011`.

## Approach

Use the vscode.dev/redirect relay pattern: the OAuth authorization request redirects to `https://vscode.dev/redirect`, which is a static HTML page that reads the `state` parameter (containing the local callback URL), appends the `code`, and redirects the browser to the local server.

## Changes

### 1. Types (`src/lib/types.ts`)
- Add `vsCodeRedirect?: boolean` to `OAuthProviderOptions`

### 2. CLI Parsing (`src/lib/utils.ts`)
- Parse `--vscode-redirect` boolean flag
- Return `vsCodeRedirect: boolean`

### 3. NodeOAuthClientProvider (`src/lib/node-oauth-client-provider.ts`)
- Store `vsCodeRedirect` from options
- `redirectUrl` getter: return `https://vscode.dev/redirect` when enabled, otherwise existing behavior
- `state()` method: return `http://127.0.0.1:<port>/?nonce=<random>` when enabled, otherwise random UUID
- `clientMetadata`: `redirect_uris` uses `redirectUrl` (already does this)

### 4. Callback Server (`src/lib/utils.ts` or `src/lib/coordination.ts`)
- When vscode-redirect is enabled, the callback server listens on `/` (root) instead of `/oauth/callback`
- Pass `callbackPath` through to coordination layer

### 5. Entry Points (`src/proxy.ts`, `src/client.ts`)
- Wire `vsCodeRedirect` through to provider and coordination

### 6. README
- Document `--vscode-redirect` flag

## Flow

1. Auth request: `redirect_uri=https://vscode.dev/redirect&state=http://127.0.0.1:<port>/?nonce=<uuid>`
2. Azure AD redirects to: `https://vscode.dev/redirect?code=AUTH_CODE&state=http://127.0.0.1:<port>/?nonce=<uuid>`
3. vscode.dev/redirect reads state as URL, redirects to: `http://127.0.0.1:<port>/?nonce=<uuid>&code=AUTH_CODE`
4. Local server on `/` receives the code, extracts it from query params
