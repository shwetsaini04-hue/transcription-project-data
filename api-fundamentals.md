# API Fundamentals

A reference for understanding what APIs are, how they actually work over the wire, and how to read, call, design, and document them.

---

## 1. What an API is

An **API (Application Programming Interface)** is a contract that lets one piece of software use another without knowing how it is built inside.

The contract specifies four things:

1. **What operations are available** (what can you ask for)
2. **What inputs each operation takes** (shape, type, required vs optional)
3. **What outputs it returns** (shape, type, meaning)
4. **What can go wrong** (error conditions and how they are signalled)

That is the whole idea. Everything else is mechanism.

If you have written control systems code, you already know this pattern. A sensor driver exposes `read_position()` and you do not care whether it talks SPI or I2C underneath. The function signature is the API. Swap the driver for a different sensor with the same signature and your control loop does not change. That decoupling is the point.

### Local APIs vs web APIs

| | Local (library) API | Web API |
|---|---|---|
| Caller and callee | Same process, same machine | Different processes, usually different machines |
| Invocation | Function call | Message over a network |
| Cost per call | Nanoseconds | Milliseconds to seconds |
| Failure modes | Exceptions | Exceptions, plus timeouts, network loss, partial failure |
| Data passed | Python objects, pointers | Serialized bytes (JSON, protobuf) |

```python
# Local API — a function call
import numpy as np
result = np.fft.fft(signal)
```

```python
# Web API — a network request
import httpx
result = httpx.get("https://api.example.com/v1/transforms/fft?signal_id=42").json()
```

Same conceptual move (ask something else to do work), radically different engineering consequences. **Everything difficult about web APIs comes from the network being unreliable, slow, and untrusted.** Keep that sentence in mind; most design rules below are downstream of it.

The rest of this document is about web APIs.

---

## 2. The client–server model

Two roles:

- **Client** — initiates. Sends a request. Waits.
- **Server** — listens. Receives a request. Does work. Sends a response.

The roles are per-interaction, not per-machine. Your FastAPI backend is a server when React calls it, and a client when it calls PostgreSQL or an external credit bureau.

```
┌──────────┐   1. request    ┌──────────┐   3. query   ┌──────────┐
│  Client  │ ──────────────► │  Server  │ ───────────► │ Database │
│ (React)  │ ◄────────────── │(FastAPI) │ ◄─────────── │(Postgres)│
└──────────┘   2. response   └──────────┘   4. rows    └──────────┘
```

The client cannot see inside the server. It cannot read the server's variables, call its private functions, or know what language it is written in. It can only send messages that conform to the contract and interpret what comes back. This isolation is a feature: the server can be rewritten in Go tomorrow and the client will not notice.

---

## 3. HTTP: the transport underneath almost everything

Nearly all web APIs run on **HTTP**. Understanding HTTP is most of understanding web APIs.

HTTP is a **request/response, text-based, stateless** protocol.

### Anatomy of a request

```http
GET /v1/journeys?type=PA&status=offer_accepted&limit=50 HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
Accept: application/json
User-Agent: Mozilla/5.0
```

Five parts:

| Part | Example | Purpose |
|---|---|---|
| **Method** | `GET` | The verb. What kind of operation. |
| **Path** | `/v1/journeys` | Which resource. |
| **Query string** | `?type=PA&limit=50` | Filters, options, pagination. Key–value pairs after `?`, joined by `&`. |
| **Headers** | `Authorization: Bearer ...` | Metadata about the request: who you are, what format you want, caching hints. |
| **Body** | (none for GET) | The payload. Present on POST, PUT, PATCH. |

### Anatomy of a response

```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: max-age=60
X-Request-Id: 8f2a1c

{
  "items": [
    {"journey_id": "J-1001", "type": "PA", "status": "offer_accepted"}
  ],
  "total": 1,
  "page": 1
}
```

Three parts: **status code**, **headers**, **body**.

### HTTP methods

| Method | Meaning | Has body | Safe | Idempotent |
|---|---|---|---|---|
| `GET` | Retrieve a resource | No | Yes | Yes |
| `POST` | Create a resource, or trigger an action | Yes | No | No |
| `PUT` | Replace a resource entirely | Yes | No | Yes |
| `PATCH` | Update part of a resource | Yes | No | No (usually) |
| `DELETE` | Remove a resource | Rarely | No | Yes |
| `HEAD` | Like GET, headers only | No | Yes | Yes |
| `OPTIONS` | Ask what is allowed | No | Yes | Yes |

Two properties in that table matter more than people expect:

- **Safe** means the call does not change server state. A safe method can be retried, prefetched, cached, or called by a crawler with no consequences. `GET` must never mutate anything. An endpoint like `GET /users/5/delete` is a real bug, not a style preference.
- **Idempotent** means calling it N times has the same effect as calling it once. `PUT /journeys/J-1001 {status: "closed"}` twice leaves one closed journey. `POST /journeys` twice creates two journeys. This is why network retries are safe on PUT and dangerous on POST, and why the idempotency-key pattern in §9 exists.

### Status codes

The first digit is the category. Learn the categories, then the common members.

**1xx — informational.** Rare in practice.

**2xx — success**

| Code | Name | Use |
|---|---|---|
| 200 | OK | Standard success with a body |
| 201 | Created | Resource created; include a `Location` header |
| 202 | Accepted | Work queued, not finished (async jobs) |
| 204 | No Content | Success, deliberately empty body (typical for DELETE) |

**3xx — redirection**

| Code | Name | Use |
|---|---|---|
| 301 | Moved Permanently | Resource has a new permanent URL |
| 304 | Not Modified | Client's cached copy is still valid |

**4xx — client error (the caller's fault)**

| Code | Name | Use |
|---|---|---|
| 400 | Bad Request | Malformed syntax or invalid input |
| 401 | Unauthorized | Not authenticated. Misnamed; it means "unauthenticated" |
| 403 | Forbidden | Authenticated, but not allowed to do this |
| 404 | Not Found | No such resource |
| 405 | Method Not Allowed | Wrong verb for this path |
| 409 | Conflict | Violates current state (duplicate, version clash) |
| 422 | Unprocessable Entity | Syntactically valid, semantically wrong. FastAPI's default for validation failures |
| 429 | Too Many Requests | Rate limited |

**5xx — server error (your fault)**

| Code | Name | Use |
|---|---|---|
| 500 | Internal Server Error | Unhandled exception |
| 502 | Bad Gateway | Upstream service returned garbage |
| 503 | Service Unavailable | Down or overloaded; often with `Retry-After` |
| 504 | Gateway Timeout | Upstream took too long |

The 4xx/5xx split is the single most useful distinction in the whole list. A client seeing 4xx should fix its request and not retry blindly. A client seeing 5xx should retry with backoff, because the request may have been fine.

### Headers worth knowing

| Header | Direction | Purpose |
|---|---|---|
| `Content-Type` | Both | Format of the body (`application/json`) |
| `Accept` | Request | Formats the client can handle |
| `Authorization` | Request | Credentials (`Bearer <token>`) |
| `Cache-Control` | Both | Caching policy |
| `ETag` / `If-None-Match` | Both | Cheap conditional requests |
| `Retry-After` | Response | How long to wait before retrying |
| `Location` | Response | URL of a newly created resource |

### Statelessness

HTTP is stateless: **the server keeps no memory of previous requests.** Every request must carry everything needed to serve it, including identity.

This is why you send a token on every single call rather than "logging in" once and having the connection remember you. It looks wasteful and it buys something large: any server instance can handle any request, so you can put ten identical backend instances behind a load balancer and scale horizontally without sticky sessions.

---

## 4. What actually happens during one API call

End to end, when a browser calls `https://api.example.com/v1/journeys?type=PA`:

1. **DNS resolution** — `api.example.com` is resolved to an IP address.
2. **TCP connection** — a three-way handshake opens a connection to port 443.
3. **TLS handshake** — certificates are exchanged and verified, and an encrypted channel is established. This is the S in HTTPS.
4. **Request sent** — the method, path, headers, and body go over the wire as bytes.
5. **Ingress** — a load balancer, reverse proxy, or gateway receives it. It may terminate TLS, apply rate limits, and choose a backend instance.
6. **Routing** — the web framework matches the path and method to a handler function.
7. **Middleware** — cross-cutting layers run in order: logging, authentication, CORS, request ID assignment.
8. **Validation** — the framework parses the body and query params and checks them against the declared schema. Invalid input stops here with a 4xx.
9. **Handler executes** — your business logic. Usually queries a database or calls another service.
10. **Serialization** — the Python objects returned are converted to JSON bytes.
11. **Response sent** — status code, headers, and body travel back.
12. **Client parses** — JSON is deserialized, the status code is checked, and the UI updates.

Steps 1–3 are expensive, which is why clients reuse connections (keep-alive, connection pooling) instead of paying that cost per call. Steps 6–10 are the part you write.

---

## 5. REST

**REST (Representational State Transfer)** is an architectural style, not a protocol or a standard. In practice, "REST API" means an HTTP API that follows roughly these conventions.

### Resources, not procedures

Model your API around **nouns** (things) and let HTTP methods supply the **verbs**.

```
Good                          Bad
GET    /journeys              GET  /getAllJourneys
GET    /journeys/J-1001       GET  /fetchJourneyById?id=J-1001
POST   /journeys              POST /createNewJourney
PATCH  /journeys/J-1001       POST /updateJourneyStatus
DELETE /journeys/J-1001       POST /removeJourney
```

The left column has one path per resource and gets create/read/update/delete for free from the method. The right column invents a new name for every operation and grows without limit.

### Standard resource patterns

| Pattern | Path | Meaning |
|---|---|---|
| Collection | `/journeys` | All journeys |
| Item | `/journeys/J-1001` | One journey |
| Sub-collection | `/journeys/J-1001/stages` | Stages belonging to that journey |
| Sub-item | `/journeys/J-1001/stages/offer` | One specific stage |
| Filtered collection | `/journeys?type=PA&status=open` | Subset, via query params |

Path segments identify. Query parameters filter, sort, and paginate. Keep that split clean and your URLs stay predictable.

### Conventions worth following

- Plural nouns for collections (`/journeys`, not `/journey`)
- Lowercase, hyphenated paths (`/reason-codes`, not `/reasonCodes`)
- No trailing slashes
- No verbs in paths, with one pragmatic exception: genuine actions that are not CRUD on a resource, such as `POST /journeys/J-1001/retry`. Purists dislike this; it is common and readable.
- Nest at most one level deep. `/a/1/b/2/c/3` is a sign the model needs rethinking.

### Where REST is not the answer

| Style | Use when |
|---|---|
| **REST** | Resource-shaped domains, public APIs, broad client compatibility. The default. |
| **GraphQL** | Clients need wildly different subsets of a complex graph, and over-fetching is a real cost. Adds server complexity and caching difficulty. |
| **gRPC** | Service-to-service calls inside your own network where latency and payload size matter. Binary, strongly typed, fast. Poor browser support. |
| **WebSockets / SSE** | The server must push to the client (live dashboards, notifications). Not request/response at all. |
| **Webhooks** | Inverted: you register a URL and the other system calls *you* when something happens. |

---

## 6. Data formats

**JSON** is the default. Human-readable, native to JavaScript, supported everywhere.

```json
{
  "journey_id": "J-1001",
  "type": "PA",
  "offer_amount": 250000,
  "created_at": "2026-09-08T10:30:00Z",
  "stages": ["offer_generated", "offer_reviewed"],
  "rejected": false,
  "closed_at": null
}
```

JSON has six types: string, number, boolean, null, array, object. Note what is missing: no dates, no decimals, no binary, no integers distinct from floats.

Consequences you will hit:

- **Dates** go as ISO 8601 strings in UTC (`2026-09-08T10:30:00Z`). Always UTC over the wire; convert to local time in the UI.
- **Money** should be an integer in the smallest unit (paise) or a string, never a float. `0.1 + 0.2 != 0.3` in float arithmetic, and rounding errors in financial output are the kind of bug that gets escalated.
- **Large integers** beyond 2^53 lose precision in JavaScript. Send IDs as strings.
- **Binary** must be base64-encoded, which inflates size by about 33%.

Alternatives: **XML** (legacy, SOAP), **Protocol Buffers** (binary, gRPC), **MessagePack** (compact binary JSON), **CSV** (bulk export endpoints).

---

## 7. Authentication and authorization

Two different questions, often confused:

- **Authentication (authn)** — who are you? → 401 if it fails
- **Authorization (authz)** — are you allowed to do this? → 403 if it fails

### Common mechanisms

**API keys.** A shared secret in a header. Simple, no expiry, no user identity, hard to revoke selectively. Fine for server-to-server internal calls, weak for anything user-facing.

```http
X-API-Key: sk_live_a1b2c3d4
```

**HTTP Basic.** Base64 of `username:password` in the header. Base64 is encoding, not encryption, so this is plaintext credentials on every request. Only acceptable over TLS, and even then rarely the right choice.

**Bearer tokens / JWT.** The dominant pattern. The client authenticates once, receives a signed token, and sends it on every subsequent request.

```http
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ1c2VyXzQyIn0.sig
```

A **JWT (JSON Web Token)** is three base64 segments separated by dots: header, payload, signature. The payload holds claims (user ID, roles, expiry). The signature lets the server verify the token was not tampered with, without a database lookup.

Two things people get wrong about JWTs: the payload is **encoded, not encrypted**, so anyone can read it (never put secrets in it), and a token stays valid until it expires, so revocation before expiry requires extra machinery (short lifetimes plus refresh tokens, or a denylist).

**OAuth 2.0 / OIDC.** A delegation framework, not a login form. It lets a user grant an application limited access to their data on another service without handing over a password. OAuth 2.0 handles authorization; **OpenID Connect** layers identity on top so you also learn *who* the user is.

The flow, simplified:

```
User → App: "log me in"
App  → Identity Provider: redirect the user there
User → Identity Provider: authenticates directly (app never sees the password)
IdP  → App: authorization code
App  → IdP: exchanges code + client secret for tokens
App  → API: calls with the access token
```

Entra ID, Google Sign-In, and "Log in with GitHub" are all this. Register the app, get a client ID and secret, redirect, exchange, call.

### Practical rules

- Always TLS. A token over plain HTTP is a token that has been stolen.
- Short-lived access tokens (minutes), long-lived refresh tokens stored more carefully.
- Never put credentials in URLs. URLs land in browser history, server logs, and referrer headers.
- Authenticate at the edge, authorize in the handler. Knowing who someone is does not tell you whether they may read *this specific* record.

---

## 8. CORS

A browser-only restriction that confuses nearly everyone the first time.

By default, **the browser blocks JavaScript on `https://app.example.com` from reading responses from `https://api.example.com`.** Different origin (scheme + host + port), so it is blocked. This is the same-origin policy, and it exists to stop a malicious page from quietly reading your bank's API using your cookies.

**CORS (Cross-Origin Resource Sharing)** is how a server opts in. It responds with headers naming who is allowed:

```http
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PATCH
Access-Control-Allow-Headers: Authorization, Content-Type
```

For anything beyond a simple GET, the browser first sends a **preflight** `OPTIONS` request asking permission, then sends the real one.

Three things to internalise:

1. CORS is enforced **by the browser**, not the server. `curl` and Python ignore it entirely. A request that fails in your React app and succeeds in Postman is almost always CORS.
2. It protects the *user*, not the server. It is not an access control mechanism.
3. Serving the frontend and API from the **same origin** removes the problem completely. This is exactly what a reverse proxy or a platform like Azure Static Web Apps buys you: the SPA calls `/api/journeys` on its own origin, the proxy forwards to the backend, and the browser never sees a cross-origin request. It is the cleanest fix available.

---

## 9. Designing a good API

### Versioning

Once someone depends on your API, you cannot change it freely. Version from day one.

```
/v1/journeys        # URL path — most common, most visible
```

Alternatives are a header (`Accept: application/vnd.api.v1+json`) or a query param (`?version=1`). URL path wins on clarity and cacheability.

**Breaking changes** need a new version: removing a field, renaming a field, changing a type, making an optional param required, changing an error contract. **Non-breaking changes** do not: adding an optional field, adding an endpoint, adding an optional param. Design clients to ignore unknown fields so you can add freely.

### Pagination

Never return an unbounded collection. Two approaches:

**Offset-based** — simple, familiar, degrades on large offsets and can skip or duplicate rows if data changes between pages.

```
GET /journeys?limit=50&offset=100
```

**Cursor-based** — an opaque pointer to a position. Stable under concurrent writes and fast at depth. Cannot jump to page 47.

```
GET /journeys?limit=50&cursor=eyJpZCI6MTAwfQ
```

Always return the metadata the client needs:

```json
{
  "items": [...],
  "pagination": {"limit": 50, "offset": 100, "total": 4823, "has_more": true}
}
```

### Filtering, sorting, field selection

```
GET /journeys?type=PA&status=open              # filter
GET /journeys?sort=-created_at                 # sort, "-" = descending
GET /journeys?fields=journey_id,status         # sparse fields, reduce payload
GET /journeys?created_after=2026-09-01         # range
```

Pick a convention and apply it to every collection endpoint. Consistency matters more than which convention you choose.

### Error responses

Errors are part of your contract. Give them a stable, machine-readable shape.

```json
{
  "error": {
    "code": "JOURNEY_NOT_FOUND",
    "message": "No journey exists with id J-9999",
    "details": {"journey_id": "J-9999"},
    "request_id": "8f2a1c3e"
  }
}
```

- `code` is a stable string clients can branch on. Do not make them parse prose.
- `message` is for a human debugging.
- `request_id` ties the client's report to your server logs. This one field saves hours.
- Return **all** validation errors at once, not the first one.
- Never leak stack traces, SQL, or internal hostnames to the caller. Log those; return something generic.

### Rate limiting

Protects you from abuse and from one buggy client. Communicate the state:

```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 47
X-RateLimit-Reset: 1757331600
Retry-After: 30
```

Return 429 when exceeded. Clients should back off exponentially with jitter, not retry in a tight loop.

### Idempotency keys

Solves a real problem: the client sends `POST /payments`, the network drops the response, and the client does not know whether the payment happened. Retrying might charge twice.

The client generates a unique key and sends it:

```http
POST /payments
Idempotency-Key: 7c9e6679-7425-40de-944b-e07fc1f90ae7
```

The server stores the key with the result. A repeat of the same key returns the stored result instead of performing the operation again. Any API that moves money or creates records needs this.

### Other design rules

- **Consistent naming.** Pick `snake_case` or `camelCase` for JSON fields and never mix.
- **Return the created object** from POST, with a `Location` header.
- **Be liberal in what you accept, conservative in what you send** within reason. Do not silently coerce nonsense.
- **Design for the client's screen.** If every dashboard page makes six calls to render one view, consider a purpose-built aggregate endpoint. Clean resource modelling that requires a waterfall of round trips is worse than a slightly impure endpoint that returns what the page needs.

---

## 10. OpenAPI: documentation as a machine-readable artifact

**OpenAPI** (formerly Swagger) is a specification format for describing an HTTP API in YAML or JSON. Because it is machine-readable, one file gives you rendered documentation, client SDKs in a dozen languages, mock servers, and contract tests.

```yaml
openapi: 3.0.3
info:
  title: Journey Analytics API
  version: 1.0.0
paths:
  /v1/journeys/{journey_id}:
    get:
      summary: Retrieve a single journey
      parameters:
        - name: journey_id
          in: path
          required: true
          schema: {type: string}
      responses:
        '200':
          description: The journey
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Journey'
        '404':
          description: Not found
components:
  schemas:
    Journey:
      type: object
      required: [journey_id, type, status]
      properties:
        journey_id: {type: string, example: "J-1001"}
        type: {type: string, enum: [PA, TOP_UP_PA, PQ_KYC, PQ_NACH, PQ_STANDALONE_ASSET]}
        status: {type: string}
        offer_amount: {type: integer, description: "Amount in paise"}
```

FastAPI generates this for you from your type hints, then serves two interactive UIs at `/docs` (Swagger UI) and `/redoc`. You get accurate documentation as a by-product of writing typed code, which is the main reason it is worth writing typed code.

---

## 11. A worked example in FastAPI

```python
from fastapi import FastAPI, HTTPException, Query, Depends
from pydantic import BaseModel, Field
from datetime import datetime
from typing import Literal
from enum import Enum

app = FastAPI(title="Journey Analytics API", version="1.0.0")


class JourneyType(str, Enum):
    PA = "PA"
    TOP_UP_PA = "TOP_UP_PA"
    PQ_KYC = "PQ_KYC"


# Response schema — this becomes the OpenAPI definition automatically
class Journey(BaseModel):
    journey_id: str = Field(..., example="J-1001")
    type: JourneyType
    status: str
    offer_amount: int = Field(..., description="Amount in paise")
    created_at: datetime


class Page(BaseModel):
    items: list[Journey]
    total: int
    limit: int
    offset: int


@app.get("/v1/journeys", response_model=Page, tags=["journeys"])
def list_journeys(
    type: JourneyType | None = None,
    status: str | None = None,
    limit: int = Query(50, ge=1, le=200),
    offset: int = Query(0, ge=0),
    db=Depends(get_db),
):
    """
    List journeys, newest first.

    Supports filtering by type and status. Results are paginated;
    `total` reflects the full filtered set, not the current page.
    """
    rows, total = repo.query(db, type=type, status=status,
                             limit=limit, offset=offset)
    return Page(items=rows, total=total, limit=limit, offset=offset)


@app.get("/v1/journeys/{journey_id}", response_model=Journey, tags=["journeys"])
def get_journey(journey_id: str, db=Depends(get_db)):
    """Retrieve a single journey by its identifier."""
    journey = repo.get(db, journey_id)
    if journey is None:
        raise HTTPException(
            status_code=404,
            detail={"code": "JOURNEY_NOT_FOUND",
                    "message": f"No journey exists with id {journey_id}"},
        )
    return journey
```

What each part is doing:

| Code | Role |
|---|---|
| `@app.get("/v1/journeys")` | Binds method + path to this handler |
| `type: JourneyType \| None = None` | Optional query param, validated against the enum |
| `Query(50, ge=1, le=200)` | Default plus bounds; a request with `limit=5000` gets a 422 automatically |
| `journey_id: str` in the path | Path parameter, extracted and type-checked |
| `response_model=Page` | Output contract. Filters out any field not declared |
| `Depends(get_db)` | Dependency injection for connections, auth, shared setup |
| Docstring | Becomes the endpoint description in `/docs` |

The validation, the error responses, and the entire OpenAPI document all fall out of the type annotations. You did not write any of it separately.

---

## 12. Calling APIs

**curl** — universally available, good for a quick probe.

```bash
curl -X GET "https://api.example.com/v1/journeys?type=PA&limit=5" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/json"

# See status and headers
curl -i https://api.example.com/v1/journeys

# POST with a JSON body
curl -X POST https://api.example.com/v1/journeys \
  -H "Content-Type: application/json" \
  -d '{"type": "PA", "offer_amount": 250000}'
```

**Python — httpx** (async-capable) or **requests**.

```python
import httpx

with httpx.Client(base_url="https://api.example.com",
                  headers={"Authorization": f"Bearer {token}"},
                  timeout=10.0) as client:
    r = client.get("/v1/journeys", params={"type": "PA", "limit": 50})
    r.raise_for_status()          # turns 4xx/5xx into an exception
    data = r.json()
```

**JavaScript — fetch.**

```javascript
const res = await fetch(`/api/v1/journeys?type=PA&limit=50`, {
  headers: { Authorization: `Bearer ${token}` },
});
if (!res.ok) throw new Error(`HTTP ${res.status}`);
const data = await res.json();
```

`fetch` does **not** throw on 4xx or 5xx. It only rejects on network failure. Check `res.ok` yourself or you will silently render an error body as data.

**Postman / Insomnia / Bruno** — GUI clients for exploring, saving collections, and sharing requests with a team.

**Browser DevTools → Network tab** — the fastest debugging tool you own. Shows every request the page made, with full headers, payload, response, timing, and status.

---

## 13. Reading and writing API documentation

Good documentation for a single endpoint contains:

1. **Method and path** — `GET /v1/journeys/{journey_id}`
2. **One-line summary** of what it does
3. **Authentication** required, and which scope or role
4. **Parameters** — name, location (path/query/header/body), type, required, default, constraints, description
5. **Request body schema** with a realistic example
6. **Response schema** per status code, with realistic examples
7. **Error cases** — which codes, when, and what the body looks like
8. **Side effects** — what changes in the system, what else gets triggered
9. **Rate limits and idempotency** behaviour if they apply
10. **A copy-pasteable example call**

When you are *reading* unfamiliar documentation, the fast path is: find the authentication section first, then find one endpoint, then get a single call working end to end. Do not read linearly. Nothing clarifies an API like one successful response on your own machine.

Documentation that only exists as prose will drift from the code within weeks. Documentation generated from the code (OpenAPI from type hints) cannot drift. Prefer the latter and reserve hand-written prose for the things a schema cannot express: why the endpoint exists, what the business rules are, how it fits a workflow.

---

## 14. Common mistakes

| Mistake | Why it hurts |
|---|---|
| Verbs in paths (`/getJourneys`) | Discards HTTP's built-in semantics; the API grows without structure |
| `GET` that mutates state | Breaks caching, prefetching, and retry safety |
| Returning 200 with `{"error": ...}` | Clients check status codes; hiding failures behind 200 defeats every generic client, proxy, and monitor |
| Unbounded list endpoints | One call returns 400k rows and takes the service down |
| No versioning | You can never change anything without breaking someone |
| Leaking internals in errors | Stack traces and SQL are an information disclosure risk |
| No timeouts on outbound calls | One slow upstream exhausts your connection pool and cascades |
| Floats for money | Rounding errors in financial figures |
| Local times without a zone | Ambiguous timestamps, off-by-one-day bugs, DST chaos |
| Inconsistent field naming | Every client writes translation code |
| Tokens in query strings | Credentials land in logs and browser history |
| Retrying non-idempotent calls | Duplicate records, double charges |

---

## 15. Glossary

| Term | Meaning |
|---|---|
| **API** | A contract letting one program use another |
| **Endpoint** | One method + path combination |
| **Resource** | A thing the API exposes (a journey, a user) |
| **Payload / body** | Data carried in a request or response |
| **Header** | Metadata key–value pair on a request or response |
| **Query parameter** | Key–value after `?` in a URL |
| **Path parameter** | Variable segment in a path (`/journeys/{id}`) |
| **Serialization** | Converting in-memory objects to bytes (JSON) |
| **Deserialization** | The reverse |
| **Idempotent** | Repeating the call has the same effect as one call |
| **Safe method** | Does not change server state |
| **Stateless** | The server retains nothing between requests |
| **Middleware** | Code running before/after every handler |
| **CORS** | Browser rules for cross-origin requests |
| **JWT** | Signed, self-describing token |
| **OAuth 2.0** | Delegated authorization framework |
| **OIDC** | Identity layer over OAuth 2.0 |
| **OpenAPI / Swagger** | Machine-readable API description format |
| **SDK** | Client library wrapping an API |
| **Webhook** | Reverse API; the server calls your URL on an event |
| **Rate limit** | Cap on requests per client per time window |
| **Latency** | Time from request sent to response received |
| **Payload size** | Bytes on the wire; affects latency on slow networks |

---

## 16. A practice path

1. **Call a public API from the terminal.** `curl https://api.github.com/users/torvalds`. Read every header that comes back. Change the path until you get a 404 and look at the body.
2. **Call it from Python** with `httpx`. Print `r.status_code`, `r.headers`, `r.json()`.
3. **Open DevTools → Network** on any web app you use and watch the calls it makes as you click around. This is the single best way to build intuition for real-world API design.
4. **Write a three-endpoint FastAPI service** with an in-memory dict as the store: list, get by id, create. Run it and open `/docs`.
5. **Break it deliberately.** Send a wrong type, a missing required field, a `limit` above the bound. Read the 422 body carefully.
6. **Add auth** — a dependency that checks a header and raises 401.
7. **Add pagination and a stable error shape** to what you built.
8. **Call your own API from a small React page** and hit the CORS error. Fix it once with middleware, then again with a proxy, so you understand both.

Steps 4 through 8 take a weekend and will teach you more than reading another twenty pages.

---

## 17. Where to go next

| Topic | Why it matters |
|---|---|
| HTTP/2 and HTTP/3 | Multiplexing removes head-of-line blocking; changes performance advice |
| Caching (ETag, Cache-Control) | The cheapest large performance win available |
| API gateways | Centralised auth, rate limiting, routing, observability |
| Async patterns (202 + polling, webhooks) | For work that outlives a request timeout |
| Contract testing | Stops you breaking clients you cannot see |
| Observability (structured logs, tracing, RED metrics) | You cannot operate what you cannot measure |
| gRPC / protobuf | Internal service-to-service performance |
| GraphQL | If client-driven querying becomes a real need |
