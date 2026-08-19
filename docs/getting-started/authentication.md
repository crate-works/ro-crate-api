---
sidebar_position: 3
title: Authentication
---

# Authentication

The RO-Crate API supports multiple authentication methods to accommodate
different use cases, from public data access to secure server-to-server
integrations.

## Authentication Methods

### 1. No Authentication (Public Access)

Many RO-Crate API endpoints are publicly accessible for read-only operations:

```bash
# Public access - no authentication required
curl https://data.ldaca.edu.au/api/entities
```

**Use Cases:**

- Browsing public collections
- Accessing open research data
- Building public discovery interfaces

### 2. API Key Authentication

API keys provide a simple authentication method for server-to-server
integrations:

```bash
# Using API key in header
curl -H "X-API-Key: your-api-key-here" \
  https://data.ldaca.edu.au/api/entities
```

**Use Cases:**

- Automated scripts and batch processing
- Server-to-server integrations
- CI/CD pipelines
- Research data harvesting

**Getting an API Key:**

1. Contact the API provider (implementation-specific)
2. Provide details about your use case
3. Receive your API key via secure channel
4. Store securely (environment variables, key management service)

**Best Practices:**

- Never commit API keys to version control
- Use environment variables: `X_API_KEY=your-key`
- Rotate keys regularly
- Use different keys for different environments
- Monitor key usage and set up alerts

### 3. OpenID Connect (Recommended)

OpenID Connect builds on OAuth2 to provide user identity information:

#### Discovery

Most implementations provide an OpenID Connect discovery document:

```bash
curl https://api-provider.com/.well-known/openid-configuration
```

**Example Response:**

```json
{
  "issuer": "https://api-provider.com",
  "authorization_endpoint": "https://api-provider.com/oauth/authorize",
  "token_endpoint": "https://api-provider.com/oauth/token",
  "userinfo_endpoint": "https://api-provider.com/oauth/userinfo",
  "jwks_uri": "https://api-provider.com/.well-known/jwks",
  "scopes_supported": ["openid", "read", "write"],
  "response_types_supported": ["code"]
}
```

#### ID Token

In addition to access tokens, you'll receive an ID token with user information:

```json
{
  "access_token": "...",
  "id_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer"
}
```

### 4. OAuth2 Authorization Code Flow

OAuth2 provides secure authentication for user-facing applications:

#### Scopes

Two coarse scopes are defined:

- **`read`** — the read surface: entities, files, search, and
  `/capabilities`.
- **`write`** — the [deposit](/docs/deposit) and RO-Crate endpoints. Only
  meaningful where the implementation declares `deposit.supported` as `true`.

Finer-grained authorisation — which collections a user may read, who may
deposit what — is implementation-defined and expressed through ordinary `403`
responses. Request the least privilege your client needs.

#### Step 1: Authorization Request

Redirect users to the authorization endpoint:

```text
https://api-provider.com/oauth/authorize?
  response_type=code&
  client_id=your-client-id&
  redirect_uri=https://yourapp.com/callback&
  scope=read&
  state=random-state-value
```

#### Step 2: Handle Authorization Response

The user is redirected back with an authorization code:

```text
https://yourapp.com/callback?
  code=authorization-code&
  state=random-state-value
```

#### Step 3: Exchange Code for Token

```bash
curl -X POST https://api-provider.com/oauth/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=authorization_code&
      code=authorization-code&
      client_id=your-client-id&
      client_secret=your-client-secret&
      redirect_uri=https://yourapp.com/callback"
```

**Response:**

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "refresh-token-here",
  "scope": "read"
}
```

#### Step 4: Use Access Token

```bash
curl -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  https://data.ldaca.edu.au/api/entities
```

## Security Best Practices

### For API Keys

- Store in environment variables, not code
- Use secure key management systems in production
- Implement key rotation policies
- Monitor usage patterns and set up alerts
- Use least-privilege principle (minimal required scopes)

### For OAuth2/OpenID Connect

- Always use HTTPS in production
- Validate the `state` parameter to prevent CSRF
- Store tokens securely (encrypted, HTTPOnly cookies)
- Implement proper token refresh logic
- Validate JWT signatures and claims
- Use PKCE (Proof Key for Code Exchange) for public clients

### General Security

- Implement proper CORS policies
- Use secure session management
- Log authentication events for monitoring
- Implement rate limiting to prevent abuse
- Keep authentication libraries up to date

## Rate Limiting

All authentication methods are subject to rate limiting:

### Rate Limit Headers

```text
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1609459200
```

### Rate Limit Exceeded (429)

```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Rate limit exceeded. Please retry after the specified time.",
    "details": {
      "retryAfter": 60
    }
  }
}
```

### Handling Rate Limits

- Implement exponential backoff
- Cache responses when possible
- Use webhooks instead of polling
- Respect the `Retry-After` header
- Consider upgrading to higher rate limits if available

## Troubleshooting

### Common Issues

#### Invalid API Key (401)

- Verify the API key is correct
- Check if the key has expired
- Ensure proper header format: `X-API-Key: your-key`

#### Access Denied (403)

- Verify required scopes/permissions
- Check if the resource requires specific access levels
- Ensure token hasn't expired

#### Rate Limited (429)

- Implement exponential backoff
- Check rate limit headers
- Consider caching or reducing request frequency

#### OAuth2 Flow Issues

- Verify redirect URIs match exactly
- Check client ID/secret configuration
- Validate state parameter consistency
- Ensure proper scope requests
