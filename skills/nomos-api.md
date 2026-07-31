---
name: nomos-api
description: >-
  Knowledge of the Nomos energy REST API (api.nomos.energy): base URL, OAuth
  authentication, versioning, pagination, filtering, error handling, and the
  full endpoint catalog for the German whitelabel energy platform. Use when
  building or debugging an integration that calls the Nomos API (subscriptions,
  leads, plans, customers, invoices, prices, consumption/usage, smart meter
  orders, grid fee reductions, market partners), or when an agent holds a Nomos
  OAuth token and needs to know which endpoints its scope (read:* vs write:*)
  allows.
---

# Nomos API

Nomos runs a whitelabel energy platform: partners sell electricity under their own brand while Nomos handles onboarding, supplier switching, metering, pricing, billing, and support. This skill describes the public REST API so an agent can call it correctly.

The API is REST over HTTPS only (plain HTTP is rejected). All requests share the base URL:

```
https://api.nomos.energy
```

Authoritative request and response schemas live in the OpenAPI spec, not in this file. Fetch the spec for exact field names, types, enums, and which fields are required:

```
https://api.nomos.energy/openapi.2026-05-27.curie.json
```

Swap the version segment to target another version (see Versioning). A browsable reference is at https://docs.nomos.energy.

## Authentication

The API uses OAuth 2.0. An integration holds a `client_id` and `client_secret` (created in the Nomos dashboard) and exchanges them for a short-lived bearer token.

Most integrations use the client credentials grant for server-to-server access against their own organization's data:

```bash
curl -X POST https://api.nomos.energy/oauth/token \
  -u "${CLIENT_ID}:${CLIENT_SECRET}" \
  -d grant_type=client_credentials
```

The response carries the token, its lifetime, a refresh token, and the granted scope:

```json
{
  "access_token": "eyJhbGciOiJI...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "21cc84a3ad98...",
  "scope": "read:* write:*"
}
```

Access tokens last 60 minutes. Cache the token and reuse it; when it expires, exchange the refresh token (`grant_type=refresh_token`, `refresh_token=...`) for a new pair rather than requesting a fresh token every call. Send the token on every request:

```bash
curl https://api.nomos.energy/subscriptions \
  -H "Authorization: Bearer ${ACCESS_TOKEN}"
```

If the integration acts on behalf of an end customer (for example a home energy manager reading one subscription's prices), use the authorization code grant with PKCE instead. The token usage and refresh model are identical.

## Scopes

This is the most important rule for an agent: what you can do is bounded by the token's scope. Two scopes exist today.

| Scope     | Grants                                                       |
| --------- | ------------------------------------------------------------ |
| `read:*`  | Every `GET` request across the API (the read surface below). |
| `write:*` | Every mutating request (`POST`, `PATCH`, `PUT`, `DELETE`).   |

The granted scope is in the `scope` field of the token response, space-delimited. Behavior to enforce:

- A token with only `read:*` can call every endpoint in the Read surface. Any call in the Write surface returns `403 FORBIDDEN`. Do not attempt writes with a read-only token; treat them as unavailable.
- A token with `read:* write:*` (the default, full access) can call everything.
- Scopes are fixed when the key is created and cannot be narrowed per request. To change a key's scope, a new key must be issued and rotated.

When you only need to read data (an ETL job, a BI sync, a monitoring agent), request a read-only key so a bug cannot mutate state.

## Versioning

The `X-API-Version` header selects the request and response shapes. Each Auth Client has a default version, used when the header is absent; new clients default to the latest stable release. Breaking changes ship only in a new version, so omitting the header keeps an integration stable. Build clients to ignore unrecognized response fields, since additive changes land in existing versions.

```bash
curl https://api.nomos.energy/subscriptions \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "X-API-Version: 2026-05-27.curie"
```

Every response echoes the version it was processed with in the `X-API-Version` header.

| Version             | Status                  |
| ------------------- | ----------------------- |
| `2026-05-27.curie`  | Latest stable (default) |
| `2026-01-29.edison` | Previous release        |
| `2025-12-16.batman` | Previous release        |
| `2025-12-01.chucky` | Initial stable release  |

Version-specific behavior worth knowing:

- Feed-in subscriptions (selling power back to the grid) require `2026-05-27.curie` or later. On earlier versions, leads, subscriptions, prices, and quotes for feed-in plans return `400 BAD_REQUEST`.
- Several routes were renamed in `2026-05-27.curie`. On earlier versions use the old names: `GET /subscriptions/{id}/usage` was `GET /subscriptions/{id}/consumption`, `GET /invoices` was `GET /subscriptions/{id}/invoices`, `GET`/`POST /meter-readings` was `GET`/`POST /subscriptions/{id}/meter_readings`, and `GET /market-partners` was `GET /suppliers`. On `curie` the old paths return `NOT_FOUND` pointing at the new ones.
- Validation errors are structured from `2026-05-27.curie` onward: a `400` carries an `errors` array with one entry per invalid field, alongside the usual envelope. Earlier versions return only the single `message`.

## Conventions

### Identifiers

IDs are prefixed by type: `cus_` (customer), `sub_` (subscription), `plan_` (plan), `lead_` (lead), `inv_` (invoice), `mp_` (market partner). Pass them verbatim in paths.

### Pagination

Every list endpoint returns the same envelope and uses cursor-based pagination, newest first by `created_at`.

```json
{
  "object": "list",
  "items": [],
  "has_more": true,
  "next_page": "eyJsYXN0X2NyZWF0ZWRfYXQiOi..."
}
```

- `limit`: page size, 1 to 100, default 10.
- `cursor`: pass the previous response's `next_page`. Omit on the first request.
- Stop when `has_more` is `false` (`next_page` is then `null`).
- Keep every other query parameter (filters, `limit`) identical across the walk, or the cursor is invalidated. A tampered cursor returns `400 BAD_REQUEST`.

### Filtering

List endpoints filter through bracket-notation query parameters: `filter[<field>][<operator>]=<value>`. Multiple filters combine with AND.

```bash
curl -G "https://api.nomos.energy/subscriptions" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  --data-urlencode "filter[status][eq]=active" \
  --data-urlencode "filter[created_at][gte]=2026-01-01T00:00:00Z"
```

Operators by field type:

- String: `eq`, `ne`, `in`, `is_null`, `contains`, `starts_with`, `ends_with` (the last three are case-insensitive).
- Number: `eq`, `ne`, `gt`, `gte`, `lt`, `lte`, `in`, `is_null`.
- Date (ISO 8601 with timezone): `gte`, `lte`, `is_null`.
- Boolean: `eq`, `is_null`.
- Enum: `eq`, `ne`, `in`, `is_null`.

`in` takes a comma-separated list. An unknown field or a disallowed operator is silently ignored (it returns the unfiltered list), so a typo will not error. Malformed syntax or an invalid value returns `400 BAD_REQUEST` naming the parameter. The per-endpoint reference lists each filterable field and its allowed operators.

### Errors

Every error uses one envelope with standard HTTP status semantics:

```json
{
  "code": "BAD_REQUEST",
  "message": "invalid_type in 'customer.email': Required",
  "requestId": "37a04f8f-e791-491c-81e1-86cd304649bb",
  "docs": "https://docs.nomos.energy/api-references/errors/BAD_REQUEST"
}
```

Switch on `code`, not on the message. Log `requestId` for support. From `2026-05-27.curie`, validation failures additionally carry an `errors` array with one entry per invalid field; use it to report every invalid field at once instead of parsing `message`.

| Status | Code                    | Meaning and handling                                                           |
| ------ | ----------------------- | ------------------------------------------------------------------------------ |
| 400    | `BAD_REQUEST`           | Validation failed or unparseable input. Fix the request; do not retry.         |
| 401    | `UNAUTHORIZED`          | Missing, invalid, or expired token. Refresh and retry.                         |
| 403    | `FORBIDDEN`             | Valid token without permission, often a scope mismatch (write with `read:*`).  |
| 404    | `NOT_FOUND`             | Resource does not exist or is outside your organization. Check the ID.         |
| 405    | `METHOD_NOT_ALLOWED`    | Wrong HTTP method for the endpoint.                                            |
| 409    | `CONFLICT`              | A unique value is already taken. Change it before retrying.                    |
| 422    | `UNPROCESSABLE_ENTITY`  | Well-formed request that breaks a business rule. Read `message`; do not retry. |
| 429    | `TOO_MANY_REQUESTS`     | Rate limited. Back off with exponential delay and jitter, then retry.          |
| 500    | `INTERNAL_SERVER_ERROR` | Nomos-side error. Retry with backoff; if it persists, contact support.         |

Retry only `429` and `5xx`. Never retry a `4xx` (except `429`) without changing the request.

## Read surface (requires `read:*`)

All endpoints below are `GET`. A read-only token can call every one of them.

### Plans

- `GET /plans` - list plans (the tariffs configured on your organization).
- `GET /plans/{id}` - retrieve a plan.
- `GET /plans/{id}/quote` - retrieve a quote for a plan. Prices include VAT. Feed-in quotes require `2026-05-27.curie`.

### Leads

- `GET /leads` - list leads (pre-filled checkout links you created).
- `GET /leads/{id}` - retrieve a lead.

### Subscriptions

- `GET /subscriptions` - list subscriptions.
- `GET /subscriptions/{id}` - retrieve a subscription.

### Customers

- `GET /customers` - list customers.
- `GET /customers/{id}` - retrieve a customer.

### Usage

- `GET /subscriptions/{id}/usage` - retrieve usage data for a subscription. Usage is electricity drawn from, or fed into, the grid; values are positive in both directions and the direction comes from the plan. Named `/subscriptions/{id}/consumption` before `2026-05-27.curie`.
- `GET /meter-readings` - list meter readings. Scope to one subscription with `filter[subscription][eq]=<id>`. Was `GET /subscriptions/{id}/meter_readings` before `2026-05-27.curie`.

Smart meters: data becomes available from 4pm the following day, with preliminary values possibly replaced by final values up to the 8th working day of the next month. Analog meters: monthly values from customer or operator readings, with Nomos estimating at month-end until a reading arrives.

### Prices

- `GET /subscriptions/{id}/prices` - retrieve the price time series for a subscription to optimize consumption against. Prices include VAT. For feed-in subscriptions this is the per-kWh payout (EPEX day-ahead spot, no grid fees); feed-in requires `2026-05-27.curie`.

### Invoices

- `GET /invoices` - list monthly invoices, including all line items (energy, grid fees, taxes). Scope to one subscription with `filter[subscription][eq]=<id>`. `type: "usage"` bills metered consumption; `type: "prepayment"` charges a fixed advance. Was `GET /subscriptions/{id}/invoices` before `2026-05-27.curie`.
- `GET /invoices/{id}` - retrieve an invoice.
- `GET /invoices/{id}/file` - retrieve the invoice PDF.

### Smart meter orders

- `GET /meter-orders` - list smart meter orders.
- `GET /meter-orders/{id}` - retrieve a smart meter order.

### Grid fee reductions

- `GET /grid-fee-reductions` - list §14a EnWG grid fee reductions. Customer-scoped tokens see only their own.
- `GET /grid-fee-reductions/{id}` - retrieve a grid fee reduction.

### Market partners

- `GET /market-partners` - list German electricity retailers, used to populate a "previous supplier" dropdown in checkout. Filter by name with `filter[name][contains]=<query>`. Was `GET /suppliers` before `2026-05-27.curie`.

### Events

- `GET /events/{id}` - retrieve an event by id (the payload referenced by a webhook).

## Write surface (requires `write:*`)

All endpoints below mutate state. A token without `write:*` gets `403 FORBIDDEN`. Before calling, fetch the endpoint's request schema from the OpenAPI spec; bodies are validated strictly.

### Subscriptions

- `POST /subscriptions` - create a subscription. Consumption plans support meter types, optional VAT ID, flexible start dates, previous-supplier cancellation, and product orders (smart meters, §14a). Feed-in plans require `2026-05-27.curie`, a present smart meter, a mandatory VAT ID for companies, future start dates, and no product orders.
- `PATCH /subscriptions/{id}` - update the editable fields of a subscription: `billing_address`, `payment_method`, and `metadata`. Only allow-listed properties are accepted; any other field is ignored. A `billing_address` (all of street, house number, zip, city) replaces the subscription's billing address; the address row is mutated in place, so a row shared across a customer's subscriptions changes everywhere it is referenced. `metadata` keys are merged onto the stored metadata; set a key to `null` to remove it, or send `metadata: null` to clear all keys. An empty body is a no-op. Requires `2026-05-27.curie`.
- `POST /subscriptions/{id}/terminate` - terminate a subscription with a discriminated `reason`: `ORDINARY` (optional `intended_end_at`, otherwise the plan's cancellation period applies), `MOVE_OUT` (`intended_end_at` mandatory), or `WITHDRAWAL`. End timestamps must be exactly midnight Europe/Berlin. Requires `2026-05-27.curie`.

### Customers

- `PATCH /customers/{id}` - update a customer's name. Only `first_name` and `last_name` are accepted; both are optional so a request may update just one, and an empty body is a no-op.

### Leads

- `POST /leads` - create a lead: a secure, pre-filled checkout link returned in `link`, so a prospect finishes signup with only payment details. Feed-in leads require `2026-05-27.curie`.

### Meter readings

- `POST /meter-readings` - submit a customer-reported analog meter reading (`value` in kWh, `timestamp`, and the `subscription` id in the body) for the subscription's active meter. Was `POST /subscriptions/{id}/meter_readings` before `2026-05-27.curie`.

### Load forecasts

- `POST /load-forecasts` - transmit a load forecast for a subscription. Submit by 09:00 local time the previous day for the day-ahead auction, or at least 15 minutes before delivery for intraday. Data points must be strictly consecutive and cover full calendar days.

### Grid fee reductions

- `POST /grid-fee-reductions` - create a §14a EnWG grid fee reduction (consumption only, not feed-in). Active subscriptions create with `ordered` status and submit immediately; pending subscriptions create with `intended` and submit on confirmation.

### Smart meter orders

- `POST /meter-orders` - create a smart meter order for a subscription whose plan supports the smart-meter product. Not applicable to feed-in (a smart meter must already exist before feed-in signup).

## Working effectively against this API

- Read the granted `scope` from the token response and refuse to attempt out-of-scope calls; do not learn the scope by triggering `403`s.
- Pin `X-API-Version` explicitly so behavior does not shift when the default moves. Use `2026-05-27.curie` unless the integration was built against an earlier version's route names.
- Fetch the OpenAPI spec for the chosen version to get exact request bodies and response fields before constructing a call. Treat this file as the map, the spec as the schema.
- Paginate fully when a list endpoint is the source of truth; do not assume the first page is complete.
- On error, branch on `code`, retry only `429` and `5xx` with backoff, and surface `requestId` to the user when reporting a failure.
