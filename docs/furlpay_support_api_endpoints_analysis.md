# FurlPay: Support System API Endpoints & Platform Audit

> Comparative developer analysis of the API endpoints, authentication patterns, and integration architectures for Zendesk, Chatwoot, and DevRev — verified against vendor documentation, July 2026 — plus the FurlPay-side endpoints that exist in production today.

---

## Part 0: What FurlPay already runs (production, July 2026)

Before integrating any external help desk, note that the support surface below is
live at `furlpay.com`. Any platform integration is additive, not foundational.

| Endpoint | Auth | Purpose |
| :--- | :--- | :--- |
| `POST /api/support/tickets` | Session (Bearer/cookie) | Create a ticket (Zod-validated: category, subject, body). Stored in FurlPay's own kv layer — transaction hashes and account data never leave the vault. KYC-category tickets also land in the `ops:security` compliance feed. |
| `GET /api/support/tickets` | Session | List the caller's tickets. |
| `POST /api/support/message` | Public (IP rate-limited) | Anonymous chat-bubble message, forwarded to `SUPPORT_INBOX` by email. |
| `GET /api/support/sso` | Session | Zendesk JWT SSO handoff (HS256, `iat` + `jti`, `external_id` = FurlPay user id, self-submitting form POST to `https://{subdomain}.zendesk.com/access/jwt`). Dormant — returns 404 until `ZENDESK_SUBDOMAIN` and `ZENDESK_SHARED_SECRET` are set. |
| `POST /api/chat` | Session | AI co-pilot with support tools: transaction/deposit status, card freeze/unfreeze (human-confirmed), KYC tier status, ticket creation (human-confirmed). Deterministic tool router + HITL; no agent framework dependency. |
| `/support` | Public page | Support center: category tiles (stablecoins, cards, KYC, travel, developers), ticket form, ticket list. |

The "LangGraph Support Agent" from earlier drafts of this analysis does not exist
and is not planned: the production co-pilot is a deterministic tool router with
optional LLM phrasing over grounded facts, which is the property that makes it
safe to run against a money API.

---

## Part 1: API Architecture Comparison

| System | Base API Endpoint | API Specification | Auth Protocol | Developer Testing Tools |
| :--- | :--- | :--- | :--- | :--- |
| **Zendesk** | `https://{subdomain}.zendesk.com/api/v2` | Custom REST / JSON | OAuth 2.0 Bearer, or Basic auth with `{email}/token:{api_token}` (see deprecation note) | Zendesk Public Postman Workspace |
| **Chatwoot** | `https://{domain}/api/v1` | Swagger / OpenAPI (self-hosted: `/swagger`) | `api_access_token` request header (NOT `Authorization: Bearer`) | Local Swagger UI on self-hosted instances |
| **DevRev** | `https://api.devrev.ai` | OpenAPI Specification 3.0 | `Authorization: Bearer {PAT}` (Personal Access Token) | DevRev REST Postman Collection + downloadable OpenAPI specs |

**Verified corrections to the original draft (July 2026):**

1. **Zendesk API tokens are being retired.** Zendesk has announced that API
   tokens are permanently deactivated on **April 30, 2027**. Any new
   integration must use OAuth 2.0 access tokens from day one; Basic auth with
   an API token is a dead end. This materially changes the integration cost
   comparison — Zendesk now requires an OAuth app registration, not a pasted
   token.
2. **Chatwoot does not use Bearer auth for its Application APIs.** The user
   access token (Profile Settings → Access Token) is sent as an
   `api_access_token` header. Platform API tokens come from the Super Admin
   Console on self-hosted instances, and Platform API keys can only access
   objects they created or were explicitly granted — a real constraint when
   provisioning accounts programmatically.
3. **DevRev `works.create` requires a `part`.** Tickets cannot be created
   without associating them to a product part, which means a FurlPay part
   taxonomy (rails, cards, KYC, travel, gateway) must exist in DevRev before
   the first ticket API call works.

---

## Part 2: Endpoint Layouts by Platform

### 1. Zendesk API (modular suite)

Zendesk is segmented across business capabilities rather than one spec:

- **Ticketing & Core (Support API):** `GET/POST /api/v2/tickets.json`,
  `GET /api/v2/tickets/count.json`, users, groups, macros, triggers.
- **Knowledge Base (Help Center API):** `GET /api/v2/help_center/articles.json`
  — relevant if the 15-article FurlPay help center is ever mirrored there.
- **Messaging (Conversations API):** web widget and social channels.
- **Custom Data (Custom Objects API):** could mirror FurlPay ticket categories
  and transaction references into ticket views without exporting ledger data.
- No single static endpoint count exists; the Zendesk Public Postman Workspace
  is the practical index.

### 2. Chatwoot API (3-tier partitioning)

1. **Application APIs** — agent/admin actions:
   `GET /api/v1/accounts/{account_id}/conversations`,
   `POST /api/v1/accounts/{account_id}/contacts`.
2. **Client APIs** — build custom chat UIs using `inbox_identifier` +
   `contact_identifier`, no user password involved. This is the tier a native
   in-app support chat would use.
3. **Platform APIs** — instance administration (accounts, users, roles) on
   self-hosted installs, with the created-by-this-key access restriction noted
   above.

Self-hosted instances expose the full schema at `http://localhost:3000/swagger`.

### 3. DevRev API (OpenAPI 3.0, unified graph)

- **Works API:** `POST /works.create`, `GET /works.get` — tickets, issues and
  incidents are all "works", distinguished by type. Ticket creation requires a
  `part` (see correction 3).
- **Core objects:** Accounts, Artifacts, Parts, Rev Users (customer identities
  — the natural mapping target for FurlPay `userId`).
- **Timeline API:** comments, internal notes, event history per work item.
- Auth: PAT from Settings → Account → Personal Access Token, sent as
  `Authorization: Bearer`. The PAT is shown once at creation.

---

## Part 3: FurlPay Integration Strategy

Phase 1 runs entirely on FurlPay's own endpoints (Part 0) — shipped, no
external dependency. The decision tree for Phase 2:

| Trigger | Choice | First integration steps |
| :--- | :--- | :--- |
| Enterprise partner mandates a SOC 2 help desk | **Zendesk** | (1) Set `ZENDESK_SUBDOMAIN` + `ZENDESK_SHARED_SECRET` — the SSO route goes live with no code change. (2) Register an OAuth app (API tokens die April 2027). (3) Mirror tickets with `POST /api/v2/tickets.json`, mapping FurlPay `user.id` to `external_id` — the same identifier the SSO JWT already sends, so support identity stays stable across email changes. |
| Support volume needs multi-agent inboxes + Telegram | **Chatwoot (self-hosted)** | (1) Provision the instance (Rails + Postgres + Redis — a real ops commitment). (2) Create contacts via `POST /api/v1/accounts/{id}/contacts` keyed on FurlPay `userId`. (3) Point a Chatwoot webhook (`conversation_created`, `message_created`) at a new authenticated `POST /api/webhooks/support` route, which must verify a shared-secret HMAC over the raw body per FurlPay's webhook rules before touching any state. |
| Support must unify with the product backlog | **DevRev** | (1) Model FurlPay parts (rails, cards, KYC, travel, gateway). (2) Create tickets via `works.create` with `type: ticket` and the mapped part. (3) Sync FurlPay users as Rev Users. |

Ticket-sync mapping, whichever platform wins:

| FurlPay field | Zendesk | Chatwoot | DevRev |
| :--- | :--- | :--- | :--- |
| `user.id` | `external_id` | contact `identifier` | Rev User external ref |
| `ticket.category` | ticket field / custom object | conversation label | `part` |
| `ticket.id` | external ref custom field | conversation `additional_attributes` | works display id |

Compliance rule that survives any platform choice: ticket bodies may reference
transaction hashes, but ledger data itself is never exported — the support
platform gets pointers, FurlPay's kv/ops feeds remain the system of record,
and any support action that mutates user state (card limits, MFA resets) is
executed through FurlPay's own authenticated APIs so it lands in the
`ops:security` event feed.

---

Sources: [Zendesk security and authentication](https://developer.zendesk.com/api-reference/introduction/security-and-auth/), [Zendesk API token → OAuth migration](https://developer.zendesk.com/documentation/api-basics/authentication/oauth-migration/), [Chatwoot APIs](https://developers.chatwoot.com/contributing-guide/chatwoot-apis), [Chatwoot personal access token](https://www.chatwoot.com/hc/user-guide/articles/1757445004-how-to-find-your-personal-access-token-in-chatwoot), [DevRev getting started](https://developer.devrev.ai/api-reference/getting-started), [DevRev works.create](https://developer.devrev.ai/public/api-reference/works/create).
