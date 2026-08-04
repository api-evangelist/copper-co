---
name: Copperco
description: Use when building integrations with Copper's crypto asset management platform, managing orders (transfers, withdrawals, staking), handling fund movements between portfolios and exchanges, setting up webhooks for event notifications, or working with lending and settlement operations.
metadata:
    mintlify-proj: copperco
    version: "1.0"
---

# Copper Platform API Skill

## Product summary

Copper is a crypto asset management platform with a REST API for programmatic access to portfolios, orders, wallets, and settlements. Agents use it to create and manage fund transfers, withdrawals, staking operations, lending, and network settlements. The API uses HMAC-SHA256 authentication with API keys and requires request signatures. Key files: API keys are created in the Copper Platform UI (Settings > API Keys); service accounts manage team-wide access. Base URLs: Production `https://api.copper.co/platform`, Staging `https://api.stage.copper.co/platform`, Testnet `https://api.testnet.copper.co/platform`. All timestamps are in milliseconds since Unix epoch; numeric values are returned as strings to prevent floating-point precision loss. See the [API Reference](https://developer.copper.co/api-reference) for endpoint details and [llms.txt](https://developer.copper.co/llms.txt) for comprehensive page navigation.

## When to use

Reach for this skill when:
- Building integrations that create, approve, or cancel orders (transfers, withdrawals, staking)
- Fetching balances, transaction history, or order status
- Moving funds between portfolios, exchanges, or external addresses
- Setting up webhooks to receive real-time notifications about order state changes
- Managing lending operations (create loans, accept/repay, rebalance)
- Implementing Copper Network settlements or transfers between counterparties
- Handling multi-signature or approval workflows (co-signing, master password entry)
- Testing API endpoints with Postman or OpenAPI specifications

## Quick reference

### Authentication headers (required for every request)

| Header | Value |
|--------|-------|
| `Authorization` | `ApiKey {API_KEY}` |
| `X-Signature` | HMAC-SHA256 hex digest of concatenated string |
| `X-Timestamp` | Current Unix timestamp in milliseconds |
| `Content-Type` | `application/json` |

### Signature generation formula

Concatenate: `{timestamp}{METHOD}{path}{body}` → HMAC-SHA256 with secret → hex encode

Example: `1730482675607POST/platform/orders{"orderType":"withdraw","amount":"1.0"}`

### Common HTTP response codes

| Code | Meaning |
|------|---------|
| 200 | Success |
| 400 | Bad request (validation error) |
| 401 | Unauthorized (invalid API key) |
| 403 | Forbidden (no access to resource) |
| 404 | Not found |
| 409 | Conflict (business logic error) |
| 429 | Rate limit exceeded (30,000 req/5min per IP; 1,000 req/5min per User ID) |
| 500 | Server error |

### Rate limits

- **30,000 requests per 5 minutes** per IP address
- **1,000 requests per 5 minutes** per User ID + Organization ID pair
- **30 failed auth attempts per hour** per API Key + IP pair → 15-minute lockout
- **60 failed business logic requests per minute** per User ID → 3-minute lockout

### Key entity relationships

```
Organization → Portfolios → Wallets → Orders/Loans
            → Crypto Addresses
            → Organization Members
```

Note: "Account" in the UI = "Portfolio" in the API.

### Order types (common)

- `transfer` — move funds between portfolios
- `withdraw` — move funds to external address
- `staking-order` — delegate or undelegate stake
- `settlement-order` — Copper Network bilateral settlement
- `loan` — lending operations

### Order status flow

Typical: `pending` → `accepted` → `signed` (if required) → `approved` (if required) → `executed` → `completed`

Some orders require: master password, co-signature, or signing before execution.

## Decision guidance

| Scenario | Use | Why |
|----------|-----|-----|
| Automated integrations, team-wide access | Service account + API key | Survives team changes; keys don't disable if user leaves |
| Personal testing, single developer | User account + API key | Simpler setup; tied to individual |
| Restrict API key to specific IPs | IP allowlist in API key settings | Reduces risk if key is compromised |
| Real-time order updates | Webhooks | Polling is inefficient; webhooks push events immediately |
| Testing endpoints before production | Postman collection or OpenAPI spec | Faster than writing code; auto-handles auth |
| Staging/demo environment | Use `api.stage.copper.co/platform` | Separate from production; safe for testing |
| Testnet (blockchain testing) | Use `api.testnet.copper.co/platform` | Blockchain transactions don't affect real assets |

## Workflow

### 1. Set up authentication

1. **Create a service account** (recommended for integrations):
   - Go to Copper Platform UI → Settings → Service accounts → Add new
   - Name it (e.g., "Trading Bot", "Finance Ops")
   - Grant permissions: Trader, Withdrawal Operator, Loan Manager, Approver
   - Set order limits if needed
   - Confirm and wait for approval

2. **Create an API key**:
   - Go to Settings → API Keys → Add new
   - Select the service account as owner
   - Optionally restrict to specific IP addresses
   - Copy and securely store the API secret (you won't see it again)

3. **Test authentication**:
   - Use the Postman collection or write a simple GET request to `/platform/portfolios`
   - Ensure all three headers are present and signature is correct

### 2. Understand the entity structure

1. **Fetch your organization**:
   - GET `/organizations` to list your organizations
   - Note the organization ID

2. **Fetch portfolios** (accounts):
   - GET `/portfolios` to list all portfolios
   - Each portfolio has wallets, orders, and loans

3. **Fetch wallets and balances**:
   - GET `/wallets` to see all wallets and their balances
   - Balances are returned as strings; parse carefully

### 3. Create and manage orders

1. **Create an order**:
   - POST `/orders` with order type, amount, source/destination
   - Include `externalOrderId` for idempotency (prevents duplicate orders if request retries)
   - Response includes order ID and status

2. **Check order status**:
   - GET `/orders/{orderId}` to poll status
   - Or use webhooks to receive status updates automatically

3. **Handle approval workflows**:
   - If order status is `co-sign-required`: PATCH `/orders/{coSignOrderId}` to approve
   - If status is `master-password-required`: PATCH `/orders/{orderId}` with SHA-256 hash of password
   - If status is `signing-required`: PATCH `/orders/{startSigningOrderId}` with signature

4. **Cancel an order**:
   - DELETE `/orders/{orderId}` before it's executed
   - Once executed, cancellation may not be possible

### 4. Set up webhooks for real-time updates

1. **Create a webhook subscription**:
   - Go to Settings → Webhooks → Create new
   - Specify your public URL (must be accessible from internet)
   - Select events to subscribe to (e.g., order status changes)
   - Choose HMAC-SHA256 or ECDSA validation
   - Optionally filter by portfolio

2. **Validate incoming webhooks**:
   - Extract `X-Signature`, `X-Timestamp`, `X-Id` headers
   - Reconstruct the signature using the same HMAC-SHA256 method as API requests
   - Compare with received signature; reject if mismatch
   - Check timestamp is recent (prevent replay attacks)

3. **Respond to webhooks**:
   - Return HTTP 200 OK immediately
   - Process the event asynchronously
   - Copper will retry failed webhooks

### 5. Test and deploy

1. **Use Postman or OpenAPI spec**:
   - Download from [Try it out](https://developer.copper.co/api-reference/try-it-out)
   - Set `copperApiKey` and `copperApiSecret` in global variables
   - Test endpoints before writing production code

2. **Test in staging first**:
   - Use `api.stage.copper.co/platform` base URL
   - Verify order creation, approval, and webhook delivery

3. **Monitor rate limits**:
   - Log API response headers for rate limit info
   - Implement exponential backoff for 429 responses
   - Avoid tight polling loops; use webhooks instead

## Common gotchas

- **Signature generation fails**: Ensure timestamp is in milliseconds (not seconds), method is uppercase, path includes `/platform` prefix, and body is the exact JSON sent (no extra whitespace). Test with provided code examples first.

- **API key works but requests fail with 403**: Check that the service account has the required permissions (Trader, Withdrawal Operator, etc.) for the action. Permissions are set per portfolio.

- **Numeric precision lost**: All amounts are returned as strings. Parse as strings, not floats. Use a decimal library if doing arithmetic.

- **Timestamps in milliseconds**: The API expects and returns timestamps in milliseconds since Unix epoch, not seconds. Multiply `time.time()` by 1000 in Python.

- **Order doesn't execute after approval**: Check if `master-password-required` or `signing-required` status is returned. These require additional steps before execution.

- **Webhook not received**: Ensure your URL is publicly accessible (not localhost), responds with 200 OK within a reasonable time, and the signature validation passes. Check the Copper Platform UI for webhook delivery logs.

- **Rate limit lockout**: If you hit 429, wait for the lockout period (5 minutes for user ID limits, until request count drops below 30,000 for IP limits). Implement exponential backoff and retry logic.

- **Idempotency**: Always include `externalOrderId` in order creation requests. If a request times out and you retry, the same `externalOrderId` prevents duplicate orders.

- **Portfolio vs Account confusion**: The Copper UI calls them "Accounts"; the API calls them "Portfolios". They're the same thing.

- **Staking operations vary by currency**: Not all currencies support all staking operations (e.g., ADA supports pool-creation and stake-delegation, but SOL doesn't). Check the staking guide for your currency.

## Verification checklist

Before submitting work:

- [ ] API key and secret are stored securely (not in code, use environment variables)
- [ ] All three authentication headers (Authorization, X-Signature, X-Timestamp) are included in every request
- [ ] Signature is generated correctly: timestamp (ms) + method (uppercase) + path (/platform/...) + body
- [ ] Numeric values are handled as strings, not floats
- [ ] Timestamps are in milliseconds, not seconds
- [ ] Order creation includes `externalOrderId` for idempotency
- [ ] Approval workflows check for `co-sign-required`, `master-password-required`, and `signing-required` statuses
- [ ] Webhooks validate incoming signatures before processing
- [ ] Rate limit handling includes exponential backoff for 429 responses
- [ ] Error responses are logged with both `error` code and `message` fields
- [ ] Tested in staging environment before production deployment
- [ ] Service account has required permissions for all operations (Trader, Withdrawal Operator, etc.)

## Resources

- **Comprehensive page navigation**: [llms.txt](https://developer.copper.co/llms.txt)
- **API Reference**: [Getting started](https://developer.copper.co/api-reference/getting-started) — base URLs, date formats, response codes
- **Authentication**: [Authentication](https://developer.copper.co/api-reference/authentication) — signature generation with code examples (Bash, Python, Java, Go, Scala)
- **Service Accounts**: [Service accounts](https://developer.copper.co/api-reference/service-accounts) — best practices and permission types

---

> For additional documentation and navigation, see: https://developer.copper.co/llms.txt