# 54 — Authentication & Authorization

> Part 9 — Specialized Building Blocks · Prerequisites: topics 05, 07, 09, 10, 11 (helpful: 27, 28, 34)

Every request that reaches a system raises two questions. **Who is this?** (authentication, "authn") and
**are they allowed to do this?** (authorization, "authz"). On a single server with a login form, both are
easy: check the password, remember the user in a session, check a permissions table. In a distributed
system they become design problems. Where is the "who is logged in" state kept when requests land on any
of 50 servers? How does a microservice three hops away know who the original user was? How do you let a
third-party app read a user's calendar without giving it the user's password? How do you answer "can Alice
view this document?" when the answer depends on folder membership, group nesting and sharing links, in
milliseconds, consistently, for billions of objects?

This topic covers the mechanisms behind those answers: sessions vs tokens, JWTs and their revocation
problem, OAuth 2.0 and OpenID Connect, and the main authorization models (RBAC, ABAC, and
relationship-based access control as in Google's Zanzibar).

By the end of this topic you should be able to:
1. Distinguish authentication from authorization, and explain the trade-off between server-side sessions and self-contained tokens in terms of where state lives.
2. Describe the structure of a JWT, how it's signed and verified, the common pitfalls, and why revoking a JWT is hard, along with the standard mitigations.
3. Walk through the OAuth 2.0 authorization code flow with PKCE, explain what each party learns, and explain what OpenID Connect adds.
4. Compare RBAC, ABAC and relationship-based authorization, and explain how Zanzibar models permissions and handles consistency.
5. Decide where authentication and authorization checks belong in a multi-service architecture.

---

## 1. Authentication vs authorization

| | Authentication (authn) | Authorization (authz) |
|---|---|---|
| Question | Who are you? | What may you do? |
| Inputs | Credentials: password, passkey, one-time code, client certificate, signed token | Identity + action + resource (+ context) |
| Output | A verified identity (principal) | Allow / deny |
| HTTP failure | `401 Unauthorized` (really "unauthenticated") | `403 Forbidden` |

They're separate concerns and often separate systems: an identity provider authenticates; each service
authorizes. Mixing them up causes bugs: "the request has a valid token" (authn) does not mean "the request
may access order 1234" (authz). The most common real-world vulnerability class in APIs is exactly this:
**broken object-level authorization** (also called IDOR, insecure direct object reference), where an
authenticated user changes an ID in the URL and reads someone else's data.

A brief note on credentials themselves: passwords are stored as slow, salted hashes (bcrypt, scrypt,
Argon2), never in plain text or with fast hashes, so a database leak doesn't reveal them. Multi-factor
authentication and **passkeys** (WebAuthn: a private key on the user's device signs a server challenge,
so there's no shared secret to steal or phish) are increasingly the norm. The rest of this topic is about
what happens *after* the user has proved who they are once: how that fact is carried across many
requests and many services.

---

## 2. Sessions vs tokens: where state lives

After login, the server needs to recognise subsequent requests without asking for the password every time.
There are two fundamentally different designs, distinguished by **where the authentication state lives**.

### 2.1 Server-side sessions

```
1. POST /login (password)        ──►  server verifies, creates session record:
                                         sessions["s_8f3a..."] = {user: 42, roles: [...], expires: ...}
2. ◄── Set-Cookie: sid=s_8f3a...; HttpOnly; Secure; SameSite=Lax
3. GET /orders  Cookie: sid=s_8f3a... ──► server looks up sessions["s_8f3a..."] -> user 42
```

The cookie holds only a random, unguessable **opaque identifier**. All the meaning lives on the server.

Properties:
- **Revocation is instant and trivial**: delete the session record (logout, password change, "sign out
  all devices", admin ban). The next request fails.
- **The server can change session data** (roles updated) without involving the client.
- **Every request needs a lookup** in the session store.

In a horizontally scaled system (topic 01) requests land on different app servers, so the session store
must be shared:
- **Sticky sessions** at the load balancer (topic 09): keep each user on the server holding their session
  in memory. Simple, but fights statelessness: a server restart logs everyone out, and load can't be
  rebalanced freely.
- **A shared session store**: Redis or Memcached (topic 11), or a database. App servers become stateless;
  the session store becomes a critical dependency on every request (it needs replication, and its latency
  is added to every request).

Cookie security attributes matter: `HttpOnly` (JavaScript can't read the cookie, limiting XSS theft),
`Secure` (HTTPS only, topic 05), and `SameSite` (don't send the cookie on cross-site requests, the main
defence against **CSRF**, where a malicious site makes the browser send an authenticated request).

### 2.2 Self-contained tokens

```
1. POST /login (password)  ──►  auth server verifies, builds a token:
                                   {sub: 42, roles: ["customer"], exp: 12:15}  + signature
2. ◄── token
3. GET /orders  Authorization: Bearer <token>  ──►  any server verifies the signature with the
                                                     auth server's public key; reads sub=42.
                                                     No lookup.
```

The token **carries the authentication state itself**, protected by a signature that only the issuer can
produce but anyone with the key can verify.

Properties:
- **No lookup per request**: any service with the verification key can authenticate the request locally.
  This is attractive in microservices: the identity can be checked at every hop without a shared
  session store.
- **Works across domains and services**, mobile apps, and APIs where cookies are awkward.
- **Revocation is hard** (section 3.4): the token is valid until it expires, because nobody is asked.
- **Data inside goes stale**: a role removed from the user still appears in tokens issued earlier.
- **Tokens are bigger** than session IDs (hundreds of bytes to KB) and sent on every request.

### 2.3 The real trade-off

It's the same trade-off seen throughout this plan: **state on the server (consistent, requires a
lookup/coordination) versus state carried by the client (no coordination, but stale and hard to
retract)**, exactly like strong vs eventual consistency (topic 28) or a cache with a TTL (topic 11).

| | Server-side session | Self-contained token (e.g. JWT) |
|---|---|---|
| State lives | Server (session store) | In the token (client holds it) |
| Per-request cost | Store lookup | Signature verification (CPU only) |
| Revocation | Immediate | Hard; waits for expiry without extra machinery |
| Freshness of roles/claims | Always current | As of issuance |
| Cross-service / cross-domain | Needs shared store | Natural |
| Failure dependency | Session store must be up | Only the public key needs to be known |

Most real systems use **both**: a short-lived self-contained **access token** for requests between
services, and a server-side, revocable record behind a long-lived **refresh token** or browser session.
That hybrid gets cheap verification on the hot path while bounding staleness to the access token's
lifetime.

---

## 3. JWT

### 3.1 Structure

A **JSON Web Token** (RFC 7519) is three base64url-encoded parts joined by dots:

```
eyJhbGciOiJSUzI1NiIsImtpZCI6ImsxIn0 . eyJzdWIiOiI0MiIsImV4cCI6MTc2MDUzNDkwMH0 . SflKxwRJSMeKKF2QT4fwpM...
└──────────── header ─────────────┘   └──────────────── payload ──────────────┘   └──── signature ────┘

header:  {"alg": "RS256", "typ": "JWT", "kid": "k1"}
payload: {
  "iss": "https://auth.example.com",   issuer: who created it
  "sub": "42",                          subject: the user (or client) it's about
  "aud": "orders-api",                  audience: who it's intended for
  "exp": 1760534900,                    expiry (Unix seconds)
  "iat": 1760534000,                    issued at
  "nbf": 1760534000,                    not valid before
  "jti": "a1b2c3",                      unique token ID
  "scope": "orders:read",               custom claims...
  "roles": ["customer"]
}
signature = sign( base64url(header) + "." + base64url(payload), key )
```

Important: the header and payload are **encoded, not encrypted**. Anyone holding the token can decode and
read every claim. Don't put secrets or sensitive personal data in a JWT. (JWE, the encrypted variant,
exists but is far less common.) The signature provides **integrity and authenticity**, not confidentiality.

### 3.2 Signing

Two families of algorithm:

| | HMAC (HS256) | Asymmetric (RS256, ES256, EdDSA) |
|---|---|---|
| Key | One shared secret signs **and** verifies | Private key signs; public key verifies |
| Who can verify | Anyone with the secret, who can then also forge tokens | Anyone with the public key; only the issuer can sign |
| Fit | A single service issuing and verifying its own tokens | Many services verify tokens from one issuer |

With asymmetric signing, the issuer publishes its public keys at a well-known **JWKS** (JSON Web Key Set)
URL. Each token's header carries a `kid` (key ID) naming the key that signed it. This enables **key
rotation**: publish a new key, start signing with it, keep the old public key in the JWKS until all tokens
signed with it have expired, then remove it. Verifiers cache the JWKS and refetch when they see an unknown
`kid`.

### 3.3 Verifying correctly (and the classic pitfalls)

A service receiving a JWT must check:
1. **Signature**, using a key it trusts and an algorithm **it** chooses (an allowlist), not whatever the
   token's header says.
2. **`exp` / `nbf`**, with a small allowance for clock skew between issuer and verifier (topic 27).
3. **`iss`**: issued by the expected issuer.
4. **`aud`**: intended for *this* service. Otherwise a token issued for service A can be replayed against
   service B.
5. Then authorization on the claims (`scope`, roles) and on the specific resource.

Historical pitfalls:
- **`alg: none`**: the JWT spec allows unsigned tokens. Libraries that honoured `"alg": "none"` from the
  header accepted forged tokens with no signature.
- **Algorithm confusion**: a server expecting RS256 receives a token with `"alg": "HS256"` and a library
  that then uses the RSA *public* key as the HMAC secret. Since the public key is public, an attacker can
  forge tokens. Fix: pin the expected algorithm per key.
- **Not checking `aud`**, accepting tokens meant for another service.
- **Long expiries** that turn a stolen token into long-lived access.

### 3.4 The revocation problem

A server-side session can be deleted. A JWT can't be "deleted": it's a signed statement that is valid until
`exp`, and verifiers don't ask anyone. If a user logs out, changes their password, has their account
suspended, or has a token stolen, every previously issued token remains valid until it expires.

Mitigations, from simplest to most stateful:

1. **Short expiry for access tokens** (5–15 minutes). Revocation then takes effect within one token
   lifetime. Combined with refresh tokens (below), this is the standard answer.
2. **Refresh tokens**: a long-lived credential, stored server-side (so revocable), used only against the
   auth server to obtain new short-lived access tokens. Revoking the refresh token stops new access tokens
   being issued; existing ones die within minutes.
   - **Refresh token rotation**: every refresh returns a *new* refresh token and invalidates the old one.
     If an old refresh token is ever presented again, it was stolen (or replayed): revoke the whole token
     family.
3. **Denylist** of revoked token IDs (`jti`) checked on each request, with entries kept only until the
   token's `exp`. Small (only revoked, unexpired tokens) and cacheable, but it reintroduces a lookup and a
   shared store, i.e. state. It can be distributed to services via pub/sub so each checks a local copy
   (accepting propagation delay).
4. **Per-user token version / "valid after" timestamp**: store `tokens_valid_after` for each user; reject
   any token with `iat` earlier than it. "Log out everywhere" is one write. Still a lookup, but per user and
   very cacheable.
5. **Token introspection** (RFC 7662): the resource server asks the auth server "is this token active?" for
   every request (or caches the answer briefly). Fully revocable, but that's a server-side session again.

Notice the pattern: **every way to make revocation immediate reintroduces server-side state**. The design
choice is how much staleness you accept (the access token lifetime, the cache TTL) in exchange for fewer
lookups, the same trade-off as cache invalidation (topic 11) or eventual consistency (topic 28).

### 3.5 Where to keep tokens in a browser

- **`localStorage`**: easy for JavaScript to attach as a header, but any XSS vulnerability can read and
  exfiltrate the token.
- **`HttpOnly` cookie**: unreadable from JavaScript (resistant to XSS theft), but cookies are sent
  automatically, so CSRF protection (`SameSite`, CSRF tokens) is needed.
- **Backend-for-frontend (BFF)**: the browser holds only a session cookie with a small server component
  that holds the OAuth tokens and calls APIs on the user's behalf. Tokens never reach the browser. This is
  the currently recommended pattern for browser apps handling sensitive data.

---

## 4. OAuth 2.0 and OpenID Connect

### 4.1 The problem OAuth solves

A photo-printing website wants to read your photos from a photo-hosting service. The bad old way: give the
printing site your photo-service password. Then it has *full* access to your account, *forever* (until
you change the password, which breaks every other app you gave it to), and the hosting service can't tell
it apart from you.

**OAuth 2.0** (RFC 6749) is a **delegated authorization** framework: it lets a user grant a third-party
application **limited** (scoped), **revocable** access to their resources **without sharing their
credentials**.

### 4.2 The four roles

| Role | In the example | What it is |
|---|---|---|
| **Resource owner** | You | The user who can grant access |
| **Client** | The printing site | The app wanting access |
| **Authorization server (AS)** | The photo service's login/consent service | Authenticates the user, asks for consent, issues tokens |
| **Resource server (RS)** | The photo service's API | Holds the data; accepts access tokens |

The AS and RS are often run by the same company but are logically separate: the RS never sees the user's
password, only tokens.

### 4.3 Authorization code flow with PKCE

This is the flow for user-facing apps (web, mobile, single-page). PKCE ("pixy", Proof Key for Code
Exchange, RFC 7636) was originally for mobile apps but is now recommended for all clients.

```
 User/Browser            Client (printing site)          Authorization Server         Resource Server
     │                          │                                │                          │
     │  1. "Connect photos"     │                                │                          │
     │─────────────────────────►│ create code_verifier (random)  │                          │
     │                          │ code_challenge = SHA256(verifier)                         │
     │  2. redirect to AS: /authorize?response_type=code&client_id=print&                   │
     │     redirect_uri=https://print.example/cb&scope=photos.read&state=xyz&               │
     │     code_challenge=...&code_challenge_method=S256                                     │
     │◄─────────────────────────│                                │                          │
     │  3. browser goes to AS ──────────────────────────────────►│                          │
     │  4. user logs in AT THE AS (password never touches client) │                         │
     │     and approves: "Print site wants to read your photos"  │                          │
     │  5. redirect back: https://print.example/cb?code=AUTHCODE&state=xyz                  │
     │◄──────────────────────────────────────────────────────────│                          │
     │─────────────────────────►│ check state == xyz             │                          │
     │                          │ 6. POST /token (back channel, server-to-server):          │
     │                          │    code=AUTHCODE, code_verifier, client auth ────────────►│ (AS)
     │                          │                                │ verify SHA256(verifier)  │
     │                          │                                │   == code_challenge       │
     │                          │◄── access_token (+ refresh_token) ──│                     │
     │                          │ 7. GET /photos  Authorization: Bearer access_token ──────────────►│
     │                          │◄──────────────────────────── photos ──────────────────────────────│
```

Why each piece exists:
- **Login happens at the AS**, in the AS's own page. The client never sees the password.
- **The code is a short-lived, single-use intermediary**, delivered through the browser (the "front
  channel", where URLs can leak via history, logs, referrers). The actual **tokens are delivered over a
  direct server-to-server "back channel"** response to step 6, never via the browser URL.
- **`state`** ties the callback to the request the client started, preventing CSRF/login-injection where
  an attacker makes your browser complete *their* authorization.
- **PKCE**: the client proves in step 6 that it's the same party that started step 2 (only it knows the
  `code_verifier` whose hash was sent earlier). A stolen authorization code (e.g. intercepted by a
  malicious app registered for the same redirect URI scheme on a phone) is useless without the verifier.
  For public clients that can't keep a client secret (mobile apps, SPAs), PKCE is the protection.
- **`redirect_uri` must exactly match a pre-registered value**, so codes can't be sent to an attacker's
  site.
- **Scopes** (`photos.read`) limit what the token allows; the user saw them on the consent screen.

### 4.4 What each party knows

This is the core of OAuth's security model:

| Party | Knows | Does not know |
|---|---|---|
| User | Their password; what they consented to | The tokens (in the server-side flow) |
| Browser | The authorization code briefly (in the URL), `state` | The access token, refresh token, code verifier (if the client is server-side), client secret |
| Client | Access/refresh tokens, its own client secret/verifier, the granted scopes | The user's password |
| Authorization server | The user's identity and credentials, the client's identity, the consent | What the client does with the data |
| Resource server | That a valid token with certain scopes (and subject) was presented | The user's password, and (with opaque tokens) anything beyond what introspection reveals |

Each party gets the minimum needed. The client gets a **scoped, expiring, revocable** credential instead of
the password; the user can revoke the client's access at the AS without changing their password.

### 4.5 Other flows

- **Client credentials**: no user involved. A service authenticates as itself (client ID + secret, or a
  signed assertion, or mTLS) and receives an access token. Used for **service-to-service** calls and batch
  jobs.
- **Device authorization flow**: for TVs and CLIs without a convenient browser: the device shows a code,
  the user enters it at a URL on their phone, the device polls the AS until approved.
- **Deprecated**: the **implicit flow** (token returned directly in the URL fragment: exposed to the
  browser and history) and the **resource owner password grant** (the client collects the password,
  defeating the purpose). OAuth 2.1 removes both.

### 4.6 Access tokens: opaque or JWT

OAuth doesn't mandate a token format.
- **Opaque tokens**: random strings; the RS calls the AS's introspection endpoint (and caches results) to
  validate. Revocable instantly, costs a lookup: server-side state.
- **JWT access tokens** (RFC 9068): the RS validates locally with the AS's JWKS. Fast, but with the
  revocation issue of section 3.4, so they're kept short-lived.

### 4.7 OpenID Connect: authentication on top of OAuth

OAuth 2.0 is about **authorization**: "this client may read these photos". It deliberately says nothing
reliable about **who the user is**. Using an OAuth access token as proof of login ("I got a token, so the
user must be X") is a known mistake: an access token is meant for the resource server, its format is
opaque to the client, and a token issued to a different client could be substituted.

**OpenID Connect (OIDC)** is a thin identity layer on OAuth 2.0 that adds:
- **The `openid` scope**: requesting it makes this an authentication request.
- **The ID token**: a JWT issued **to the client** (its `aud` is the client ID) that says "user `sub` was
  authenticated by issuer `iss` at time `auth_time`", plus optional profile claims (email, name). The client
  validates it (signature via the issuer's JWKS, `iss`, `aud`, `exp`, and a `nonce` it sent, to stop replay).
- **The UserInfo endpoint**: an API (called with the access token) returning profile claims.
- **Discovery** (`/.well-known/openid-configuration`) and dynamic metadata: endpoints, JWKS URL, supported
  algorithms, so clients can configure themselves from the issuer URL.

The distinction to keep straight:

| | ID token | Access token |
|---|---|---|
| Purpose | Tells the **client** who the user is (authentication) | Lets the client call the **resource server** (authorization) |
| Audience | The client | The resource server (API) |
| Format | Always a JWT | Opaque or JWT |
| Should be sent to APIs? | No | Yes |

OIDC is what powers "Sign in with Google/Apple/Microsoft" and enterprise **single sign-on (SSO)**: many
applications trust one identity provider; you log in once at the provider and each app gets its own ID
token. (SAML is the older XML-based protocol for the same enterprise SSO job.)

### 4.8 Identity across microservices

Inside a system, a common pattern:
1. The edge (API gateway, topic 10) authenticates the external credential (session cookie, OAuth access
   token).
2. It forwards a **verified identity** to internal services, often as a short-lived internal JWT signed by
   the gateway or an internal token service, carrying the user ID, tenant and roles.
3. Services authenticate **each other** with mTLS (service mesh, topic 10) or client-credentials tokens, so
   a service knows both *which service* is calling and *on behalf of which user*.

Blindly trusting a header like `X-User-Id: 42` from anywhere inside the network is dangerous: any
compromised or misconfigured service can impersonate any user. Internal identity should be signed and
verifiable, and services should still enforce authorization on the resources they own (**zero trust**:
being "inside the network" grants nothing by itself).

Watch for the **confused deputy** problem: service B, called by service A on behalf of user U, uses its own
broad privileges to fetch data U isn't allowed to see. The fix is to propagate the user's identity and
authorize the action as U, not as B.

---

## 5. Authorization models

Authentication tells you the principal. Authorization decides, for a request `(principal, action,
resource)`, allow or deny. The models differ in **what the decision is based on**.

### 5.1 Access control lists (ACLs)

Each resource lists who can do what: `doc 7: {alice: edit, bob: view}`. Simple and direct (Unix file
permissions, S3 bucket ACLs). Weaknesses: managing permissions per user per resource doesn't scale with
organizations ("give the whole finance team access to these 10,000 files"), and answering "what can Alice
access?" requires scanning every ACL.

### 5.2 RBAC: role-based access control

Permissions are attached to **roles**, and users are assigned roles:

```
users ──(assigned)──► roles ──(grant)──► permissions
alice                 admin              orders:read, orders:refund, users:manage
bob                   support_agent      orders:read, orders:refund
carol                 customer           orders:read_own
```

Check: does any of the user's roles include the required permission? Fast (role sets are small and fit in
a token or cache), easy to audit ("who are the admins?"), and maps to how organizations think.

Limits:
- **Role explosion**: real rules depend on context. "Support agents in the EU can refund orders under €100
  for EU customers during their shift" leads to roles like `support_eu_refund_small`, multiplying
  combinatorially.
- **No notion of the specific resource**: RBAC says Carol may `read_own` orders, but deciding which orders
  are "her own" needs data about the resource. RBAC alone can't express ownership or per-object sharing.
- **Stale roles in tokens**: if roles are embedded in a JWT, removing a role takes effect only when the
  token expires (section 3.4).

### 5.3 ABAC: attribute-based access control

Decisions are **policies** evaluated over **attributes** of the subject, the resource, the action and the
environment:

```
allow if
    subject.department == "support"
    and subject.region == resource.customer_region
    and action == "refund"
    and resource.amount < 100
    and env.time within subject.shift_hours
```

Expressive: one policy replaces dozens of roles, and it handles context (time, location, device posture,
data sensitivity labels). Policy engines such as **Open Policy Agent (OPA, Rego language)** and **AWS
Cedar** (and AWS IAM policies, which are attribute- and condition-based) separate policy from code: the
service is a **policy enforcement point (PEP)** that asks a **policy decision point (PDP)** with the
relevant attributes.

Costs:
- **Attributes must be fetched** for each decision: the resource's owner, region, amount... which may live
  in other services. Policy evaluation is cheap; gathering attributes is the expensive and
  consistency-sensitive part.
- **Hard to audit in reverse**: "who can access this record?" requires evaluating the policy against every
  possible subject.
- **"List everything I can see"** (filtering a search or a list page) can't be done by checking each item
  individually; it needs the policy translated into a query filter (OPA's partial evaluation does this).

### 5.4 ReBAC: relationship-based access control (Zanzibar)

Many products' permissions are naturally a **graph of relationships**: Alice is an *editor* of a document;
the document is in a *folder*; Bob is a *viewer* of the folder; viewers of a folder can view its documents;
the *engineering* group is a viewer of the folder; Carol is a *member* of a group that's a *member* of
engineering. The question "can Carol view the document?" is "is there a path in this graph".

Google's **Zanzibar** (USENIX ATC 2019) is the system behind permissions for Google Drive, Calendar,
YouTube, Cloud and many others. Its design has become the template for open-source systems (SpiceDB,
OpenFGA, Ory Keto).

**Relation tuples**: all permission data is stored as tuples of the form `object#relation@subject`:

```
doc:readme#owner@user:alice
doc:readme#parent@folder:eng            (the doc is in folder eng)
folder:eng#viewer@group:engineering#member   (members of the engineering group view folder eng)
group:engineering#member@user:carol
group:engineering#member@group:sre#member    (nested group: SRE members are engineering members)
```

**Namespace configuration** defines how relations compose, via **userset rewrite rules**:

```
definition doc {
    relation owner:  user
    relation editor: user
    relation viewer: user
    relation parent: folder

    permission edit = owner + editor
    permission view = viewer + edit + parent->view     # "->": inherit view from the parent folder
}
```

So "viewer" includes editors, editors include owners, and viewers of the parent folder can view the
document. Nobody writes a tuple saying Carol can view `doc:readme`; it's **derived** by walking the graph:

```
check(doc:readme, view, user:carol)?
  doc:readme view <- parent->view: folder:eng
     folder:eng view <- viewer: group:engineering#member
        group:engineering#member includes user:carol   ✓  ALLOW
```

APIs: **Check** (is this subject allowed?), **Read** / **Write** tuples, **Expand** (show the tree of who has
access), and listing ("which documents can Carol view?", which needs reverse indexes).

**Scale and performance**: Zanzibar stores tuples in Spanner (topic 55), replicated globally, and serves
trillions of tuples at millions of checks per second with low-millisecond latency, using aggressive
caching, request coalescing for hot objects (topic 12), and a specialized index (**Leopard**) that
precomputes flattened group memberships so deeply nested groups don't require deep graph walks.

**Consistency: the "new enemy" problem.** Authorization decisions must respect the *order* of permission
changes and content changes. Two scenarios from the paper:
1. Alice removes Bob from a folder's ACL, then adds new secret documents to the folder. If a check uses a
   stale replica of the ACL (from before the removal) but sees the new content, Bob sees documents he should
   never see.
2. Alice removes Bob from a document's ACL, then asks Charlie to add new content. Same issue.

Zanzibar's answer is **zookies**: opaque consistency tokens encoding a timestamp (from Spanner's TrueTime,
topic 27). When content is modified, the application asks Zanzibar for a zookie and stores it with the
content version. Later checks on that content pass the zookie, meaning "evaluate with a snapshot at least
as fresh as this". Zanzibar may serve from any replica/cache that's at least that fresh, which keeps most
checks fast and local while guaranteeing that ACL changes made before a content change are respected
(a causal / bounded-staleness guarantee, topic 28). This is the same idea as read-your-writes via version
tokens (topic 23).

### 5.5 Comparing the models

| | RBAC | ABAC | ReBAC (Zanzibar-style) |
|---|---|---|---|
| Decision based on | User's roles | Attributes + policies | Graph of relationships |
| Per-object permissions | Poor | Possible (via resource attributes) | Natural (sharing, ownership) |
| Inheritance (folders, nested groups) | Role hierarchies only | Awkward | Natural (rewrite rules) |
| Context (time, location) | No | Yes | Limited (some systems add conditions) |
| "Who can access X?" | Easy | Hard | Expand API |
| "What can U access?" | Easy for coarse perms | Hard | Needs reverse index |
| Typical use | Admin consoles, internal tools, coarse API permissions | Enterprise/cloud policies, compliance rules | Collaborative products: docs, files, projects, orgs |

They combine in practice: RBAC for coarse capabilities ("is this user an admin of the org?"), ReBAC for
object sharing, ABAC conditions for context ("only from managed devices").

### 5.6 Where checks happen

- **At the edge (API gateway)**: coarse checks: authenticated? has scope `orders:read`? rate limits
  (topic 45). Cheap, centralized, but the gateway doesn't know resource ownership.
- **In the service that owns the resource**: fine-grained "may *this* user act on *this* order". This
  must happen here, because only this service has the resource data. Most authorization bugs are missing
  checks at this layer.
- **Centralized authorization service** (Zanzibar-like or an OPA PDP): one model and one source of truth
  for permissions, consistent across services, auditable. The cost: a network dependency on every
  request (so it must be fast, cached and highly available; consider what happens when it's down: fail
  closed is the safe default for authorization, topic 44's degradation thinking applies).

---

## 6. How it connects

- **Topic 05 (TLS)**: tokens and cookies are bearer credentials and must only travel over TLS; mTLS for
  service identity; certificate chains of trust parallel JWKS key trust.
- **Topic 07 (API styles)**: 401 vs 403 semantics; bearer tokens in headers.
- **Topic 09 (load balancing)**: sticky sessions as the alternative to a shared session store.
- **Topic 10 (gateways and service mesh)**: edge authentication, mTLS between services.
- **Topic 11 and 12 (caching)**: session stores in Redis; caching introspection and permission checks;
  staleness vs revocation.
- **Topic 27 (clocks)**: `exp`/`nbf` with clock skew; Zanzibar zookies built on TrueTime.
- **Topic 28 and 23 (consistency)**: revocation latency as a staleness bound; the new-enemy problem and
  snapshot tokens like read-your-writes.
- **Topic 44 and 45**: fail-open vs fail-closed for auth dependencies; rate limiting per identity.
- **Topic 49**: guessable IDs make broken object-level authorization exploitable at scale.
- **Topic 55**: Zanzibar runs on Spanner.

---

## Common misconceptions

- "A valid token means the request is allowed." → Authentication is not authorization; every resource
  access still needs an object-level check.
- "JWTs are encrypted." → They're signed and base64url-encoded; anyone can read the payload.
- "JWTs are stateless, so they're strictly better than sessions." → They trade revocation and freshness
  for no lookups; immediate revocation needs server-side state again.
- "Logging out deletes the JWT." → Deleting it from the client doesn't invalidate copies; it stays valid
  until `exp` unless you denylist it.
- "OAuth is a login protocol." → OAuth 2.0 is delegated authorization; OpenID Connect adds authentication
  (the ID token).
- "The access token tells my app who the user is." → Use the ID token (validated, with `aud` = your client)
  for identity; access tokens are for the resource server.
- "Internal services can trust each other's headers." → Any compromised service could then impersonate any
  user; propagate signed identity and authorize at each service.
- "RBAC is enough for any app." → It can't express per-object sharing or inheritance without role explosion;
  collaborative products need relationship-based models.
- "Authorization caches can be eventually consistent like any other cache." → Stale permission data can
  expose new content to revoked users (new enemy problem); freshness must be bounded relative to content
  changes.

---

## Further reading (supplements)

- RFC 6749 (OAuth 2.0), RFC 7636 (PKCE), RFC 7519 (JWT), RFC 7662 (Token Introspection), RFC 9068 (JWT access tokens); the OAuth 2.0 Security Best Current Practice (RFC 9700) and the OAuth 2.1 draft.
- OpenID Connect Core 1.0 specification.
- Aaron Parecki, *OAuth 2.0 Simplified* (book and oauth.com).
- Pang et al., "Zanzibar: Google's Consistent, Global Authorization System" (USENIX ATC 2019).
- OWASP API Security Top 10 (Broken Object Level Authorization) and the OWASP Cheat Sheets on session management and JWT.
- Open Policy Agent documentation; AWS Cedar documentation.

---

## Questions

1. A user is logged in with a valid token and changes `/orders/1001` to `/orders/1002` in the URL, seeing another customer's order. Is this an authentication or authorization failure, and where should the fix live?
2. Compare server-side sessions and JWTs in terms of where state lives, per-request cost, and what happens when an admin bans a user.
3. Your app runs on 20 servers behind a load balancer and stores sessions in process memory. What breaks, and what are the two standard fixes?
4. Why must a JWT verifier choose the algorithm itself rather than trusting the `alg` in the header? Describe the attack.
5. A token issued for `billing-api` is accepted by `orders-api`. Which check was missing, and why does it matter?
6. Design revocation for a system using JWT access tokens: the user clicks "sign out of all devices". What happens to existing access tokens, and how do short expiries and refresh tokens bound the damage?
7. What is refresh token rotation, and how does it detect theft?
8. In the authorization code flow, why is the token returned over the back channel rather than in the redirect URL?
9. What attack does PKCE prevent, and why is it essential for mobile apps and SPAs?
10. What does the `state` parameter protect against?
11. A developer uses "we received an OAuth access token from the provider" as proof the user is logged in. What's wrong, and what should they use?
12. Explain what each party (browser, client, authorization server, resource server) knows in the code flow, and why the user's password stays with one party.
13. A support tool's RBAC has grown to 300 roles like `support_eu_refund_under_100`. What's happening, and which model would fit better?
14. Model "members of a team can view all documents in the team's folders, including nested folders" in Zanzibar-style tuples and rewrites, and explain how a check is evaluated.
15. Describe the "new enemy" problem and how zookies solve it.

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** Authorization: the user is correctly authenticated, but the service didn't check that order 1002 belongs to them (broken object-level authorization / IDOR). The fix belongs in the orders service, which owns the data: every read/write checks ownership or permission for that specific object (e.g. `WHERE id = :id AND customer_id = :user`). Unpredictable IDs (topic 49) only reduce exploitability; they don't fix it.

**2.** Sessions: state in a server-side store; the client holds an opaque ID; each request looks it up; a ban deletes the session (or flags the user) and takes effect on the next request. JWTs: state inside the signed token; each request just verifies the signature (CPU, no lookup); a ban doesn't affect already-issued tokens, which stay valid until `exp` unless you add a denylist, a per-user validity timestamp, or introspection.

**3.** A user whose request lands on a different server has no session there, so they appear logged out, randomly. Fixes: sticky sessions at the load balancer (keeps users on one server; restarts and rebalancing still lose sessions), or a shared session store like Redis (stateless app servers; the store becomes a critical, replicated dependency).

**4.** If the library uses the header's `alg`, an attacker controls how the token is verified. Attacks: `alg: none` (no signature, accepted as valid), or algorithm confusion: the server expects RS256 and has an RSA public key; the attacker sends `alg: HS256` signed with HMAC using the *public key* as the secret; a naive library verifies HMAC with the same public key and accepts it. The verifier must pin the allowed algorithm(s) per key.

**5.** The `aud` (audience) check. Without it, any token from the same issuer is accepted by every service, so a token intended for a less-privileged or compromised service (or leaked from one) can be replayed against another. Audience restriction scopes each token to its intended recipient.

**6.** The auth server revokes all of the user's refresh tokens (server-side records) and optionally sets `tokens_valid_after = now` for the user. Existing access tokens remain cryptographically valid; with no extra check they work until they expire (e.g. ≤ 10 minutes), after which the clients can't refresh. If that window is unacceptable, services check a per-user "valid after" timestamp or a `jti` denylist (cached, pushed via pub/sub), accepting a lookup and some propagation delay.

**7.** Each use of a refresh token returns a new refresh token and invalidates the previous one; the server tracks the token family. If an already-used refresh token is presented again, one of two parties (the legitimate client or an attacker) is holding a stale copy, so the token was leaked; the server revokes the whole family, forcing re-login and cutting off the attacker.

**8.** The redirect URL passes through the browser: it can end up in history, logs, Referer headers, and is visible to browser extensions or malicious apps intercepting redirects. The authorization code delivered there is short-lived, single-use and useless without client authentication/PKCE. The real tokens are returned only in the direct HTTPS response to the client's server-to-server token request, never exposed to the browser.

**9.** Authorization code interception: a malicious app (e.g. one registering the same custom URL scheme on a phone) captures the redirect containing the code and exchanges it for tokens. With PKCE the token request must include the `code_verifier` whose hash was sent in the original request; the interceptor doesn't have it. Mobile apps and SPAs are public clients that can't keep a client secret, so PKCE is the only thing binding the code to the client that requested it.

**10.** CSRF on the callback / login injection: an attacker starts a flow with their own account and tricks the victim's browser into hitting the client's callback with the attacker's code, so the victim's session is linked to the attacker's account (e.g. the victim then uploads data into the attacker's account). `state` is a random value the client generated and stored; a callback without the matching state is rejected.

**11.** An access token is meant for the resource server and says nothing reliable to the client about who authenticated; it may be opaque, and a token obtained by a different client for the same user could be substituted (token substitution). Use OpenID Connect: request the `openid` scope and validate the ID token (signature via JWKS, `iss`, `aud` = your client ID, `exp`, `nonce`), then use its `sub` as the user's identity.

**12.** Browser: sees the authorization code and state in the redirect, nothing else. Client: gets access/refresh tokens and granted scopes, has its own secret/verifier, never sees the password. Authorization server: knows the user's credentials and identity, the client, and the consent given. Resource server: sees only the access token (subject, scopes, client) and validates it. The password is entered only on the AS's own page, so only the AS ever handles it; everyone else gets scoped, revocable, expiring tokens.

**13.** Role explosion: RBAC can't express conditions on attributes (region, amount, time), so each combination becomes a role. ABAC fits better: one policy over attributes (`subject.region == resource.region and resource.amount < 100 and action == refund`), with coarse RBAC roles kept for broad capability.

**14.** Tuples: `folder:design#parent@folder:team-root`, `doc:spec#parent@folder:design`, `folder:team-root#viewer@team:payments#member`, `team:payments#member@user:dana`. Rewrites: `folder.view = viewer + parent->view`, `doc.view = viewer + editor + parent->view`. Check(doc:spec, view, user:dana): doc:spec has no direct viewer tuple for dana → follow `parent` to folder:design → its view includes parent->view → folder:team-root → its `viewer` includes `team:payments#member` → dana is a member → allow. Nested folders are handled by the recursive `parent->view`.

**15.** If permission checks read stale ACL data while content is fresh, a user whose access was revoked can see content added after the revocation (e.g. Bob removed from a folder, then secret docs added, then a check using a pre-removal ACL snapshot shows Bob the new docs). Zookies: when content is saved, the app gets a zookie (a consistency token encoding a TrueTime/Spanner timestamp of the ACL state) and stores it with the content. Checks for that content pass the zookie, and Zanzibar evaluates against a snapshot at least that fresh, so any ACL change that happened before the content change is guaranteed to be applied, while still letting most checks be served from caches/replicas.

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** Authentication vs authorization?
**A:** Authn: who are you? Authz: what may you do?

**Q:** 401 vs 403?
**A:** 401: not authenticated; 403: authenticated but not allowed.

**Q:** What is IDOR / broken object-level authorization?
**A:** An authenticated user accesses another user's object by changing its ID, because no per-object check exists.

**Q:** Where does session state live?
**A:** On the server; the cookie holds only an opaque session ID.

**Q:** Where does token state live?
**A:** In the signed token held by the client.

**Q:** Main advantage of server-side sessions?
**A:** Immediate revocation and always-fresh data.

**Q:** Main advantage of self-contained tokens?
**A:** No per-request lookup; any service with the key can verify.

**Q:** Cookie flags and what they do?
**A:** HttpOnly (no JS access), Secure (HTTPS only), SameSite (limits cross-site sending, anti-CSRF).

**Q:** Three parts of a JWT?
**A:** Header, payload, signature, base64url-encoded and dot-separated.

**Q:** Is a JWT payload confidential?
**A:** No; it's encoded, not encrypted.

**Q:** HS256 vs RS256?
**A:** HS256: shared secret signs and verifies; RS256: private key signs, public key verifies.

**Q:** What are JWKS and kid?
**A:** A published set of public keys; kid names which key signed the token, enabling rotation.

**Q:** Five checks when verifying a JWT?
**A:** Signature (pinned alg), exp/nbf, iss, aud, then authorization on claims.

**Q:** Why are JWTs hard to revoke?
**A:** Verifiers don't consult the issuer; tokens stay valid until exp.

**Q:** Standard JWT revocation strategy?
**A:** Short-lived access tokens + revocable server-side refresh tokens; denylist or per-user valid-after if needed.

**Q:** What does refresh token rotation detect?
**A:** Reuse of an old refresh token, signalling theft; revoke the family.

**Q:** OAuth 2.0's purpose?
**A:** Delegated, scoped, revocable access to resources without sharing the user's credentials.

**Q:** Four OAuth roles?
**A:** Resource owner, client, authorization server, resource server.

**Q:** Why the authorization code step?
**A:** Front channel carries only a short-lived code; tokens travel on the back channel.

**Q:** What does PKCE do?
**A:** Binds the code to the client that started the flow via a secret verifier and its hash.

**Q:** What does state protect against?
**A:** CSRF/login injection on the OAuth callback.

**Q:** Flow for service-to-service?
**A:** Client credentials.

**Q:** Which OAuth flows are deprecated?
**A:** Implicit and resource owner password grants.

**Q:** What does OIDC add to OAuth?
**A:** Authentication: the openid scope, an ID token (JWT for the client), UserInfo, discovery.

**Q:** ID token vs access token?
**A:** ID token: tells the client who the user is (aud = client). Access token: for calling the API.

**Q:** What is the confused deputy problem?
**A:** A service uses its own broad privileges on behalf of a user who lacks them.

**Q:** RBAC's main weakness?
**A:** Role explosion and no per-object or contextual rules.

**Q:** ABAC in one line?
**A:** Policies evaluated over subject, resource, action and environment attributes.

**Q:** Zanzibar relation tuple format?
**A:** object#relation@subject, where subject can be a user or a userset like group#member.

**Q:** What do userset rewrites do?
**A:** Derive permissions from other relations (editors are viewers; inherit from parent folder).

**Q:** The new enemy problem?
**A:** Stale ACL data plus fresh content lets a revoked user see new content.

**Q:** What is a zookie?
**A:** A consistency token (Spanner timestamp) requiring checks to use an ACL snapshot at least that fresh.

**Q:** Where must fine-grained authz live?
**A:** In the service owning the resource (gateway handles coarse checks).
