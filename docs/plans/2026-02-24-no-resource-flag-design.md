# Design: `--no-resource` CLI Flag for Entra ID v2.0 Compatibility

## Problem

Azure AD v2.0 rejects OAuth requests containing both `resource` and `scope` parameters with error `AADSTS9010010`. The `resource` concept is v1.0-only; in v2.0, the resource is embedded in the scope (e.g., `api://icmmcpapi-prod/mcp.tools`).

The `resource` parameter gets injected from two sources:
1. `NodeOAuthClientProvider.redirectToAuthorization()` sets it from the `--resource` flag (or defaults)
2. The MCP SDK's `auth()` function sets it independently during authorization URL construction

## Approach: Option C

Strip `resource` from the authorization URL in `redirectToAuthorization()` when `--no-resource` is set, regardless of who added it. This method is robust because `redirectToAuthorization` is the last stop before the URL reaches the browser.

## Changes

### 1. CLI Argument Parsing (`src/lib/utils.ts`)
- Parse `--no-resource` boolean flag
- Return `noResource: boolean` from `parseCommandLineArgs()`

### 2. Types (`src/lib/types.ts`)
- Add `noResource?: boolean` to `OAuthProviderOptions`

### 3. NodeOAuthClientProvider (`src/lib/node-oauth-client-provider.ts`)
- Store `noResource` from constructor options
- In `redirectToAuthorization()`: if `noResource`, delete `resource` param; otherwise set it when `authorizeResource` is defined

### 4. Entry Points (`src/proxy.ts`, `src/client.ts`)
- Pass `noResource` from parsed args to `NodeOAuthClientProvider`
- Fix existing bug in `client.ts` where `authorizeResource` is not passed through

### 5. Token Exchange
- Check if `resource` is sent during token exchange POST and strip it when `--no-resource` is set

### 6. README
- Document `--no-resource` flag with Entra ID / Azure AD v2.0 usage context

## Non-Goals
- Does not modify the MCP SDK source or bundle
- Does not change behavior when `--no-resource` is not specified (fully backwards compatible)

## Verification

```bash
npx mcp-remote https://icm-mcp-prod.azure-api.net/v1/ \
  --static-oauth-client-info '{"client_id":"aebc6443-996d-45c2-90f0-388ff96faa56"}' \
  --static-oauth-client-metadata '{"scope":"api://icmmcpapi-prod/mcp.tools"}' \
  --no-resource \
  --debug
```

The authorization URL should contain `scope=api://icmmcpapi-prod/mcp.tools` and `client_id` but no `resource=` parameter.
