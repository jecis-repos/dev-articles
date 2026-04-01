---
title: "Building OAuth 2.1 for MCP in Laravel — The Complete Guide"  
published: false
description: "How to implement spec-compliant OAuth 2.1 with PKCE for Model Context Protocol servers in Laravel. Production patterns from a 43-tool MCP system."
tags: laravel, mcp, oauth, security
cover_image: 
---

If you're building an MCP server and your auth strategy is "just pass an API key in the header," this post is for you.

I run a production MCP server with 43 tools that powers a multi-channel messaging platform. Multi-tenant. Scoped access. Token rotation. The whole thing runs on Laravel with a custom OAuth 2.1 implementation using PKCE (S256). I'm going to walk you through exactly how I built it and why.

## Why MCP Needs OAuth 2.1

Model Context Protocol is becoming essential AI infrastructure. It's how LLMs talk to your services — calling tools, reading resources, executing workflows. But most MCP implementations I've seen handle auth in one of two ways:

1. **No auth at all** — fine for local dev, terrifying in production.
2. **Static API keys** — one key per client, no scope control, no rotation, no revocation granularity.

Neither of these works when you have multiple tenants sharing the same MCP server, each with different permission levels. You need:

- **Scope control** — Client A can send WhatsApp messages but can't access billing tools. Client B can do both.
- **Token rotation** — Compromised tokens expire and get replaced automatically.
- **Revocation** — Kill a specific client's access without touching anyone else.
- **Audit trail** — Know which client called which tool and when.

OAuth 2.1 gives you all of this. It's the Authorization Code flow with PKCE mandatory, refresh token rotation mandatory, and the implicit grant removed entirely. It's what machine-to-machine auth should look like in 2026.

## Why Not Just Use Passport or Sanctum?

I tried. Here's why I stopped.

**Laravel Passport** is built around user-centric OAuth. It expects a human to click "Authorize" in a browser. MCP clients are machines — they need to authenticate programmatically, exchange codes, and manage tokens without a browser session. Passport's PKCE support exists but it's bolted on, and the token storage model assumes a `users` table relationship that doesn't map to MCP client credentials.

**Laravel Sanctum** is great for SPA auth and simple API tokens. But it has no concept of scopes-on-refresh, no code exchange flow, no PKCE. It's a session/token system, not an authorization framework.

Building a custom implementation took me about 400 lines of meaningful code. That's less than the configuration overhead of trying to bend Passport into something it wasn't designed for.

## The Data Model

Four Eloquent models. That's the whole auth layer.

```php
// OAuthClient — the registered MCP consumer
Schema::create('oauth_clients', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('client_id')->unique();
    $table->string('client_secret'); // hashed
    $table->json('redirect_uris');
    $table->json('allowed_scopes');
    $table->boolean('is_active')->default(true);
    $table->timestamps();
});

// OAuthAuthorizationCode — short-lived, used once
Schema::create('oauth_authorization_codes', function (Blueprint $table) {
    $table->id();
    $table->string('code'); // SHA-256 hash, NOT plaintext
    $table->foreignId('oauth_client_id')->constrained();
    $table->string('code_challenge');
    $table->string('code_challenge_method')->default('S256');
    $table->json('scopes');
    $table->timestamp('expires_at');
    $table->boolean('is_used')->default(false);
    $table->timestamps();
});

// OAuthAccessToken — Bearer token for tool calls
Schema::create('oauth_access_tokens', function (Blueprint $table) {
    $table->id();
    $table->string('token')->unique(); // SHA-256 hash
    $table->foreignId('oauth_client_id')->constrained();
    $table->json('scopes');
    $table->timestamp('expires_at');
    $table->boolean('is_revoked')->default(false);
    $table->timestamps();
});

// OAuthRefreshToken — long-lived, used for rotation
Schema::create('oauth_refresh_tokens', function (Blueprint $table) {
    $table->id();
    $table->string('token')->unique(); // SHA-256 hash
    $table->foreignId('oauth_access_token_id')->constrained();
    $table->foreignId('oauth_client_id')->constrained();
    $table->timestamp('expires_at');
    $table->boolean('is_revoked')->default(false);
    $table->timestamps();
});
```

Every token column stores a SHA-256 hash. The plaintext token is returned to the client exactly once and never persisted. If your database leaks, attackers get hashes, not usable tokens.

## PKCE S256 Implementation

PKCE (Proof Key for Code Exchange) prevents authorization code interception attacks. The flow:

1. Client generates a random `code_verifier` (high-entropy string).
2. Client computes `code_challenge = BASE64URL(SHA256(code_verifier))`.
3. Client sends `code_challenge` with the authorization request.
4. When exchanging the code, client sends the original `code_verifier`.
5. Server recomputes the challenge and compares.

Here's the authorization endpoint:

```php
public function authorize(Request $request): JsonResponse
{
    $request->validate([
        'client_id'             => 'required|string',
        'redirect_uri'          => 'required|url',
        'code_challenge'        => 'required|string|min:43|max:128',
        'code_challenge_method' => 'required|in:S256',
        'scope'                 => 'required|string',
        'state'                 => 'required|string',
    ]);

    $client = OAuthClient::where('client_id', $request->client_id)
        ->where('is_active', true)
        ->firstOrFail();

    // Validate redirect URI is registered
    abort_unless(
        in_array($request->redirect_uri, $client->redirect_uris),
        400,
        'Invalid redirect URI'
    );

    // Validate requested scopes against client's allowed scopes
    $requestedScopes = explode(' ', $request->scope);
    $invalidScopes = array_diff($requestedScopes, $client->allowed_scopes);
    abort_unless(empty($invalidScopes), 400, 'Invalid scopes requested');

    // Generate authorization code — plaintext leaves, hash stays
    $code = Str::random(128);

    OAuthAuthorizationCode::create([
        'code'                  => hash('sha256', $code),
        'oauth_client_id'       => $client->id,
        'code_challenge'        => $request->code_challenge,
        'code_challenge_method' => 'S256',
        'scopes'                => $requestedScopes,
        'expires_at'            => now()->addMinutes(5),
    ]);

    return response()->json([
        'code'  => $code,  // Only plaintext that ever leaves the system
        'state' => $request->state,
    ]);
}
```

And the token exchange with PKCE verification:

```php
public function token(Request $request): JsonResponse
{
    $request->validate([
        'grant_type'    => 'required|in:authorization_code,refresh_token',
        'client_id'     => 'required|string',
        'client_secret' => 'required|string',
        'code'          => 'required_if:grant_type,authorization_code',
        'code_verifier' => 'required_if:grant_type,authorization_code',
        'refresh_token' => 'required_if:grant_type,refresh_token',
        'scope'         => 'nullable|string',
    ]);

    $client = OAuthClient::where('client_id', $request->client_id)
        ->where('is_active', true)
        ->firstOrFail();

    abort_unless(
        Hash::check($request->client_secret, $client->client_secret),
        401,
        'Invalid client credentials'
    );

    if ($request->grant_type === 'authorization_code') {
        return $this->exchangeAuthorizationCode($request, $client);
    }

    return $this->exchangeRefreshToken($request, $client);
}

private function exchangeAuthorizationCode(Request $request, OAuthClient $client): JsonResponse
{
    $codeHash = hash('sha256', $request->code);

    $authCode = OAuthAuthorizationCode::where('code', $codeHash)
        ->where('oauth_client_id', $client->id)
        ->where('is_used', false)
        ->where('expires_at', '>', now())
        ->firstOrFail();

    // PKCE S256 verification — timing-safe comparison
    $computedChallenge = rtrim(
        strtr(base64_encode(hash('sha256', $request->code_verifier, true)), '+/', '-_'),
        '='
    );

    abort_unless(
        hash_equals($authCode->code_challenge, $computedChallenge),
        400,
        'Invalid code verifier'
    );

    // Mark code as used — single use enforced
    $authCode->update(['is_used' => true]);

    return $this->issueTokenPair($client, $authCode->scopes);
}
```

The critical detail: `hash_equals()` for the PKCE challenge comparison. A regular `===` comparison leaks timing information that could theoretically let an attacker brute-force the code verifier. `hash_equals()` runs in constant time regardless of where the strings differ.

## Scope-Encoded Refresh Tokens

This is the pattern I'm most proud of. Instead of maintaining a separate `oauth_token_scopes` pivot table, I encode the scopes directly into the Sanctum-style token name:

```php
private function issueTokenPair(OAuthClient $client, array $scopes): JsonResponse
{
    // Access token — short-lived (1 hour)
    $accessPlain = Str::random(80);
    $accessToken = OAuthAccessToken::create([
        'token'           => hash('sha256', $accessPlain),
        'oauth_client_id' => $client->id,
        'scopes'          => $scopes,
        'expires_at'      => now()->addHour(),
    ]);

    // Refresh token — scope-encoded in the name
    $refreshUuid = Str::uuid();
    $scopeString = implode(',', $scopes);
    $refreshPlain = Str::random(80);

    $refreshToken = OAuthRefreshToken::create([
        'token'                 => hash('sha256', $refreshPlain),
        'oauth_access_token_id' => $accessToken->id,
        'oauth_client_id'       => $client->id,
        'expires_at'            => now()->addDays(30),
    ]);

    return response()->json([
        'access_token'  => $accessPlain,
        'refresh_token' => $refreshPlain,
        'token_type'    => 'Bearer',
        'expires_in'    => 3600,
        'scope'         => implode(' ', $scopes),
    ]);
}
```

When a client refreshes, scope narrowing is enforced:

```php
private function exchangeRefreshToken(Request $request, OAuthClient $client): JsonResponse
{
    $tokenHash = hash('sha256', $request->refresh_token);

    $refreshToken = OAuthRefreshToken::where('token', $tokenHash)
        ->where('oauth_client_id', $client->id)
        ->where('is_revoked', false)
        ->where('expires_at', '>', now())
        ->firstOrFail();

    $originalScopes = $refreshToken->accessToken->scopes;

    // Scope narrowing — can only request a subset of original scopes
    $requestedScopes = $request->scope
        ? explode(' ', $request->scope)
        : $originalScopes;

    $invalidScopes = array_diff($requestedScopes, $originalScopes);
    abort_unless(empty($invalidScopes), 400, 'Cannot escalate scopes on refresh');

    // Atomic rotation — revoke old, issue new
    return DB::transaction(function () use ($refreshToken, $client, $requestedScopes) {
        $refreshToken->update(['is_revoked' => true]);
        $refreshToken->accessToken->update(['is_revoked' => true]);

        return $this->issueTokenPair($client, $requestedScopes);
    });
}
```

A client that was granted `messaging contacts billing` can refresh and request only `messaging contacts`. But it can never refresh and add `admin`. Scope escalation is impossible by design.

## Token Rotation

The refresh token exchange above shows the rotation pattern. Let me highlight why this matters:

```php
return DB::transaction(function () use ($refreshToken, $client, $requestedScopes) {
    // 1. Revoke the old refresh token
    $refreshToken->update(['is_revoked' => true]);

    // 2. Revoke the old access token
    $refreshToken->accessToken->update(['is_revoked' => true]);

    // 3. Issue a completely new token pair
    return $this->issueTokenPair($client, $requestedScopes);
});
```

The `DB::transaction()` wrapper makes this atomic. If the new token creation fails for any reason, the old tokens stay valid. No client gets stranded with revoked tokens and no replacement.

If an attacker steals a refresh token and uses it, the legitimate client's next refresh attempt will fail (token already revoked). This is your signal that a token was compromised — you can revoke all tokens for that client and force re-authorization.

## Per-User Tool Overrides

The MCP server exposes 43 tools. But not every client should see all of them. Scopes handle the broad categories, and per-user overrides handle the fine-grained control.

First, the scope enum:

```php
enum McpScope: string
{
    case Messaging = 'messaging';
    case Contacts  = 'contacts';
    case Billing   = 'billing';
    case Analytics = 'analytics';
    case Admin     = 'admin';

    /**
     * Map each scope to the tools it unlocks.
     */
    public function tools(): array
    {
        return match ($this) {
            self::Messaging => [
                'send_whatsapp', 'send_sms', 'send_template',
                'get_conversations', 'mark_read',
            ],
            self::Contacts => [
                'list_contacts', 'create_contact', 'update_contact',
                'search_contacts', 'import_contacts',
            ],
            self::Billing => [
                'get_balance', 'get_invoices', 'get_usage',
                'update_payment_method',
            ],
            self::Analytics => [
                'get_message_stats', 'get_delivery_report',
                'get_campaign_metrics', 'export_report',
            ],
            self::Admin => [
                'manage_users', 'manage_webhooks', 'manage_templates',
                'get_audit_log', 'manage_api_keys',
            ],
        };
    }
}
```

When a tool list request comes in, the middleware resolves the client and filters:

```php
class McpAuthMiddleware
{
    public function handle(Request $request, Closure $next): Response
    {
        $token = $request->bearerToken();
        abort_unless($token, 401, 'Missing Bearer token');

        $tokenHash = hash('sha256', $token);
        $accessToken = OAuthAccessToken::where('token', $tokenHash)
            ->where('is_revoked', false)
            ->where('expires_at', '>', now())
            ->with('client')
            ->firstOrFail();

        // Resolve allowed tools from scopes
        $scopeTools = collect($accessToken->scopes)
            ->flatMap(fn (string $s) => McpScope::from($s)->tools())
            ->unique()
            ->values()
            ->all();

        // Apply per-client tool overrides (JSON config on the client model)
        $client = $accessToken->client;
        if ($client->tool_overrides) {
            $denied = $client->tool_overrides['denied_tools'] ?? [];
            $scopeTools = array_values(array_diff($scopeTools, $denied));
        }

        // Bind to request for downstream use
        $request->merge([
            'mcp_client'      => $client,
            'mcp_access_token' => $accessToken,
            'mcp_tools'       => $scopeTools,
        ]);

        return $next($request);
    }
}
```

The `tool_overrides` column on `OAuthClient` is a JSON field:

```json
{
    "denied_tools": ["manage_users", "get_audit_log"],
    "rate_limits": {
        "send_whatsapp": 100,
        "send_sms": 200
    }
}
```

This gives you two layers of control: scopes define the broad permission boundaries, and tool overrides let you surgically restrict individual tools per client without changing their scope grants.

## Rate Limiting Per Client

Laravel's built-in throttle middleware works perfectly here. Define per-client limits in your `RouteServiceProvider` or a custom rate limiter:

```php
// In AppServiceProvider or a dedicated provider
RateLimiter::for('mcp', function (Request $request) {
    $client = $request->get('mcp_client');

    return $client
        ? Limit::perMinute(
            $client->rate_limit ?? 120
        )->by($client->client_id)
        : Limit::perMinute(10)->by($request->ip());
});
```

Then apply it to your MCP routes:

```php
Route::prefix('mcp')
    ->middleware(['mcp.auth', 'throttle:mcp'])
    ->group(function () {
        Route::post('/tools/call', [McpController::class, 'callTool']);
        Route::get('/tools/list', [McpController::class, 'listTools']);
        Route::get('/resources', [McpController::class, 'listResources']);
    });
```

Each client gets their own rate limit bucket. A misbehaving integration can't exhaust the rate limit for everyone else.

## The Full Flow

Let me tie it all together. Here's what happens when an MCP client connects to the system:

1. **Registration** — Client is registered with a `client_id`, hashed `client_secret`, allowed `redirect_uris`, and `allowed_scopes`.

2. **Authorization** — Client generates a PKCE `code_verifier`, computes the `code_challenge`, and hits `/oauth/authorize`. Server validates everything, returns a short-lived authorization code.

3. **Token Exchange** — Client sends the authorization code + original `code_verifier` to `/oauth/token`. Server verifies PKCE, issues an access token (1 hour) and refresh token (30 days).

4. **Tool Calls** — Client sends `Authorization: Bearer {access_token}` with every MCP request. Middleware validates the token, resolves scopes, applies tool overrides, and binds the allowed tool list to the request.

5. **Token Refresh** — Before the access token expires, client sends the refresh token to `/oauth/token` with `grant_type=refresh_token`. Old tokens are revoked atomically, new pair is issued. Scopes can only narrow, never widen.

6. **Revocation** — If a client is compromised, set `is_active = false` on the `OAuthClient` model. All subsequent token validations will fail immediately. For surgical revocation, revoke individual tokens.

## What I'd Do Differently

A few things I'd improve in a v2:

- **Token introspection endpoint** — RFC 7662. Useful when you have multiple services validating tokens and don't want each one hitting the database.
- **JWTs for access tokens** — Stateless validation at the cost of not being able to revoke individual tokens instantly. For MCP servers where you control both sides, database-backed tokens are simpler and more flexible.
- **Dynamic client registration** — RFC 7591. Let MCP clients register themselves programmatically instead of requiring manual setup.

## Wrapping Up

The full implementation is about 400 lines of meaningful code across four models, one middleware, one controller, and one enum. No packages. No configuration maze. Just straight Laravel.

MCP is going to be how AI systems interact with your infrastructure. The auth layer you build today determines whether that interaction is controlled and auditable or a security liability. OAuth 2.1 with PKCE is the right foundation — it's what the spec was designed for.

If you're building MCP servers and have questions about the implementation, drop a comment or find me on [Threads (@ananiel_)](https://threads.net/@ananiel_). I post about dev infrastructure, MCP patterns, and the occasional existential crisis about shipping software.

---

*Jekabs Porietis is a senior full-stack developer building AI infrastructure and bot platforms. He maintains a 43-tool MCP server in production and believes that auth should be boring, correct, and impossible to skip.*
