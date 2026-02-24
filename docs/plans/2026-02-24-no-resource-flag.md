# `--no-resource` Flag Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add a `--no-resource` CLI flag that suppresses the `resource` parameter from OAuth authorization URLs, enabling compatibility with Azure AD / Entra ID v2.0.

**Architecture:** Option C approach — `redirectToAuthorization()` in `NodeOAuthClientProvider` acts as the single choke point. When `--no-resource` is set, it deletes `resource` from the URL regardless of who added it (mcp-remote or the bundled MCP SDK). The flag propagates from CLI parsing through to the provider via `OAuthProviderOptions`.

**Tech Stack:** TypeScript, Vitest, tsup bundler

---

### Task 1: Add `noResource` to types

**Files:**
- Modify: `src/lib/types.ts:33` (add field after `authorizeResource`)

**Step 1: Add `noResource` field to `OAuthProviderOptions`**

In `src/lib/types.ts`, add after line 33 (`authorizeResource?: string`):

```typescript
  /** When true, strip the resource parameter from authorization URLs (for Entra ID v2.0) */
  noResource?: boolean
```

**Step 2: Commit**

```bash
git add src/lib/types.ts
git commit -m "feat: add noResource option to OAuthProviderOptions"
```

---

### Task 2: Parse `--no-resource` CLI flag

**Files:**
- Modify: `src/lib/utils.ts:839-840` (add parsing after `--resource`)
- Modify: `src/lib/utils.ts:932-945` (add to return value)

**Step 1: Write the failing test**

In `src/lib/utils.test.ts`, add inside the `Feature: Command Line Arguments Parsing` describe block:

```typescript
  it('Scenario: Parse --no-resource flag', async () => {
    const args = ['https://example.com/sse', '--no-resource']
    const usage = 'test usage'

    const result = await parseCommandLineArgs(args, usage)

    expect(result.serverUrl).toBe('https://example.com/sse')
    expect(result.noResource).toBe(true)
  })

  it('Scenario: noResource defaults to false', async () => {
    const args = ['https://example.com/sse']
    const usage = 'test usage'

    const result = await parseCommandLineArgs(args, usage)

    expect(result.noResource).toBe(false)
  })
```

**Step 2: Run test to verify it fails**

Run: `npx vitest run src/lib/utils.test.ts -t "no-resource"`
Expected: FAIL — `noResource` is not returned by `parseCommandLineArgs`

**Step 3: Add parsing logic in `utils.ts`**

After the `--resource` parsing block (line 839), add:

```typescript
  // Parse --no-resource flag
  const noResource = args.includes('--no-resource')
  if (noResource) {
    log('Resource parameter will be suppressed from authorization URLs')
  }
```

In the return object (line 932-945), add `noResource`:

```typescript
  return {
    serverUrl,
    callbackPort,
    headers,
    transportStrategy,
    host,
    debug,
    staticOAuthClientMetadata,
    staticOAuthClientInfo,
    authorizeResource,
    noResource,
    ignoredTools,
    authTimeoutMs,
    serverUrlHash,
  }
```

**Step 4: Run test to verify it passes**

Run: `npx vitest run src/lib/utils.test.ts -t "no-resource"`
Expected: PASS

**Step 5: Commit**

```bash
git add src/lib/utils.ts src/lib/utils.test.ts
git commit -m "feat: parse --no-resource CLI flag"
```

---

### Task 3: Implement `noResource` in `NodeOAuthClientProvider`

**Files:**
- Modify: `src/lib/node-oauth-client-provider.ts:31` (add field)
- Modify: `src/lib/node-oauth-client-provider.ts:42-57` (store in constructor)
- Modify: `src/lib/node-oauth-client-provider.ts:255-280` (update `redirectToAuthorization`)

**Step 1: Write the failing tests**

In `src/lib/node-oauth-client-provider.test.ts`, add inside the `authorization URL` describe block:

```typescript
    it('should strip resource parameter when noResource is set', async () => {
      provider = new NodeOAuthClientProvider({
        ...defaultOptions,
        noResource: true,
        authorizeResource: 'https://example.com',
      })

      const authUrl = new URL('https://auth.example.com/authorize')
      // Simulate SDK adding resource before redirectToAuthorization is called
      authUrl.searchParams.set('resource', 'https://sdk-injected.example.com')
      await provider.redirectToAuthorization(authUrl)

      expect(authUrl.searchParams.has('resource')).toBe(false)
      expect(authUrl.searchParams.get('scope')).toBe('openid email profile')
    })

    it('should set resource parameter when noResource is not set', async () => {
      provider = new NodeOAuthClientProvider({
        ...defaultOptions,
        authorizeResource: 'https://example.com/resource',
      })

      const authUrl = new URL('https://auth.example.com/authorize')
      await provider.redirectToAuthorization(authUrl)

      expect(authUrl.searchParams.get('resource')).toBe('https://example.com/resource')
    })
```

**Step 2: Run test to verify it fails**

Run: `npx vitest run src/lib/node-oauth-client-provider.test.ts -t "resource"`
Expected: FAIL — first test fails because `noResource` is not handled

**Step 3: Implement the changes**

In `src/lib/node-oauth-client-provider.ts`:

Add field declaration after line 31 (`private authorizeResource: string | undefined`):

```typescript
  private noResource: boolean
```

In constructor (after line 51 `this.authorizeResource = options.authorizeResource`), add:

```typescript
    this.noResource = options.noResource ?? false
```

Replace the `redirectToAuthorization` resource logic (lines 261-263):

```typescript
    if (this.noResource) {
      authorizationUrl.searchParams.delete('resource')
    } else if (this.authorizeResource) {
      authorizationUrl.searchParams.set('resource', this.authorizeResource)
    }
```

**Step 4: Run test to verify it passes**

Run: `npx vitest run src/lib/node-oauth-client-provider.test.ts`
Expected: PASS (all tests)

**Step 5: Commit**

```bash
git add src/lib/node-oauth-client-provider.ts src/lib/node-oauth-client-provider.test.ts
git commit -m "feat: implement --no-resource in NodeOAuthClientProvider"
```

---

### Task 4: Wire `noResource` through `proxy.ts` and `client.ts`

**Files:**
- Modify: `src/proxy.ts:31-42` (add parameter to `runProxy`)
- Modify: `src/proxy.ts:67-79` (pass to provider)
- Modify: `src/proxy.ts:168-196` (destructure and pass)
- Modify: `src/client.ts:32-41` (add parameter to `runClient`)
- Modify: `src/client.ts:66-77` (pass to provider, also fix missing `authorizeResource`)
- Modify: `src/client.ts:180-204` (destructure and pass)

**Step 1: Update `proxy.ts`**

Add `noResource: boolean` parameter to `runProxy()` function signature (after `authorizeResource: string`):

```typescript
async function runProxy(
  serverUrl: string,
  callbackPort: number,
  headers: Record<string, string>,
  transportStrategy: TransportStrategy = 'http-first',
  host: string,
  staticOAuthClientMetadata: StaticOAuthClientMetadata,
  staticOAuthClientInfo: StaticOAuthClientInformationFull,
  authorizeResource: string,
  noResource: boolean,
  ignoredTools: string[],
  authTimeoutMs: number,
  serverUrlHash: string,
) {
```

Add `noResource` to the `NodeOAuthClientProvider` constructor options (after `authorizeResource`):

```typescript
    authorizeResource,
    noResource,
    serverUrlHash,
```

Add `noResource` to the destructured args and pass it to `runProxy`:

```typescript
    ({
      serverUrl,
      callbackPort,
      headers,
      transportStrategy,
      host,
      debug,
      staticOAuthClientMetadata,
      staticOAuthClientInfo,
      authorizeResource,
      noResource,
      ignoredTools,
      authTimeoutMs,
      serverUrlHash,
    }) => {
      return runProxy(
        serverUrl,
        callbackPort,
        headers,
        transportStrategy,
        host,
        staticOAuthClientMetadata,
        staticOAuthClientInfo,
        authorizeResource,
        noResource,
        ignoredTools,
        authTimeoutMs,
        serverUrlHash,
      )
    },
```

**Step 2: Update `client.ts`**

Add `authorizeResource: string` and `noResource: boolean` parameters to `runClient()` (it's currently missing `authorizeResource`):

```typescript
async function runClient(
  serverUrl: string,
  callbackPort: number,
  headers: Record<string, string>,
  transportStrategy: TransportStrategy = 'http-first',
  host: string,
  staticOAuthClientMetadata: StaticOAuthClientMetadata,
  staticOAuthClientInfo: StaticOAuthClientInformationFull,
  authorizeResource: string,
  noResource: boolean,
  authTimeoutMs: number,
  serverUrlHash: string,
) {
```

Add `authorizeResource` and `noResource` to the `NodeOAuthClientProvider` constructor options:

```typescript
    authorizeResource,
    noResource,
    serverUrlHash,
```

Add `authorizeResource` and `noResource` to the destructured args and pass to `runClient`:

```typescript
    ({
      serverUrl,
      callbackPort,
      headers,
      transportStrategy,
      host,
      staticOAuthClientMetadata,
      staticOAuthClientInfo,
      authorizeResource,
      noResource,
      authTimeoutMs,
      serverUrlHash,
    }) => {
      return runClient(
        serverUrl,
        callbackPort,
        headers,
        transportStrategy,
        host,
        staticOAuthClientMetadata,
        staticOAuthClientInfo,
        authorizeResource,
        noResource,
        authTimeoutMs,
        serverUrlHash,
      )
    },
```

**Step 3: Run all tests**

Run: `npx vitest run`
Expected: PASS (all tests)

**Step 4: Build to check for compile errors**

Run: `npx tsup`
Expected: Build succeeds with no errors

**Step 5: Commit**

```bash
git add src/proxy.ts src/client.ts
git commit -m "feat: wire --no-resource through proxy.ts and client.ts"
```

---

### Task 5: Update README documentation

**Files:**
- Modify: `README.md` (add flag docs after `--auth-timeout` section, around line 220)

**Step 1: Add documentation**

After the `--auth-timeout` section (line 220), add:

```markdown
* To suppress the `resource` parameter from OAuth authorization URLs, add the `--no-resource` flag. This is required for Microsoft Entra ID (Azure AD) v2.0 endpoints, which reject requests that include both `resource` and `scope` parameters (error `AADSTS9010010`). In v2.0, the resource should be embedded in the scope instead (e.g., `api://your-app/scope`).

```json
      "args": [
        "mcp-remote",
        "https://your-entra-protected-server.example.com/v1/",
        "--static-oauth-client-info",
        "{\"client_id\":\"your-client-id\"}",
        "--static-oauth-client-metadata",
        "{\"scope\":\"api://your-app-id/mcp.tools\"}",
        "--no-resource"
      ]
```
```

**Step 2: Commit**

```bash
git add README.md
git commit -m "docs: add --no-resource flag documentation for Entra ID v2.0"
```

---

### Task 6: Final verification

**Step 1: Run full test suite**

Run: `npx vitest run`
Expected: All tests PASS

**Step 2: Build the project**

Run: `npx tsup`
Expected: Clean build with no errors

**Step 3: Verify with type checking**

Run: `npx tsc --noEmit`
Expected: No type errors
