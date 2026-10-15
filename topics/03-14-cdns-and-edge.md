# 14 — CDNs & the Edge

> Part 3 — Caching & the Edge · Prerequisites: topics 04, 05, 06, 11, 12

A user in Sydney requesting an image from a server in Virginia pays roughly 200 ms of round-trip time per
exchange before a single byte of content arrives, and several round trips are needed to set up TCP and TLS
(topics 03, 05). No amount of server optimization fixes the speed of light. A **content delivery network
(CDN)** solves this by placing caching servers in many locations close to users, so most requests are
answered nearby and only misses travel back to your servers. CDNs also absorb traffic spikes and attacks,
terminate TLS close to users, and increasingly run code at the edge. This topic explains how a CDN is
structured, how HTTP caching headers control what it does, how content gets in and out of it, how far you can
push caching toward dynamic content, and how to serve private content from a shared cache safely.

By the end of this topic you should be able to:
1. Describe edge PoPs, the origin, origin shields and the cache hierarchy, and how a request flows through them.
2. Explain cache keys and the HTTP headers that control caching: `Cache-Control`, `ETag`, `Last-Modified`, `Vary`, and revalidation.
3. Compare push and pull CDNs, and purging vs versioned URLs for updating content.
4. Explain the techniques for caching dynamic content and what edge compute adds.
5. Explain signed URLs and signed cookies for private content.
6. Predict how misconfigured caching leaks data or serves stale content, and how to avoid it.

---

## 1. Anatomy of a CDN

### 1.1 Edge, PoP, origin

- **Origin**: your own infrastructure that holds the authoritative content: application servers behind a load
  balancer, or an object store such as S3 for static files.
- **Point of presence (PoP)**: a CDN site in a city or metro area, typically inside or next to internet
  exchange points and ISPs. A large CDN has hundreds of PoPs. Each PoP contains many cache servers, load
  balancers and often DDoS filtering.
- **Edge server**: a cache server in a PoP that terminates the user's connection and serves content.

### 1.2 Getting the user to a nearby PoP

Two mechanisms, both from topic 04:

- **DNS-based routing (GeoDNS)**: `cdn.example.com` is a CNAME to the CDN's domain; the CDN's authoritative DNS
  returns the IP of a PoP close to the resolver (or to the client subnet, via EDNS Client Subnet), taking PoP
  health and load into account.
- **Anycast**: every PoP announces the same IP prefix via BGP; internet routing delivers packets to the
  topologically nearest PoP. Simple and resilient (a failing PoP withdraws its route), and absorbs DDoS traffic
  by spreading it across all PoPs.

### 1.3 The request flow

```
   User ──(short RTT)──> Edge PoP ──miss──> [Origin shield / regional tier] ──miss──> Origin
                            │                         │
                            hit: respond              hit: respond to edge, edge caches too
```

1. The user's browser resolves the CDN hostname and connects to a nearby edge. TCP and TLS handshakes cost a
   few ms instead of hundreds, a large win in itself (topics 03, 05, 06).
2. The edge computes the **cache key** for the request and looks it up. **Hit**: respond immediately.
3. **Miss**: the edge fetches from the next tier (a shield or the origin), stores the response if it is
   cacheable, and returns it.

Even for responses that cannot be cached at all, routing through the CDN often helps: the edge holds warm,
long-lived, tuned connections to the origin over the CDN's private backbone, so the user avoids a long-distance
handshake.

### 1.4 Origin shield and the cache hierarchy

With hundreds of PoPs each caching independently, a newly published popular object could produce hundreds of
simultaneous misses to the origin (one per PoP), and a long-tail object might be requested once per PoP, each
time missing. An **origin shield** (a designated mid-tier cache, also called a regional or parent tier) sits
between the edges and the origin:

```
                    ┌── Edge Sydney ──┐
                    ├── Edge Tokyo  ──┤
 users ───────────> ├── Edge Paris  ──┼──> Shield (e.g. near origin) ──> Origin
                    ├── Edge NYC    ──┤        one fetch per object
                    └── Edge Lagos  ──┘
```

- Edge misses go to the shield instead of the origin. The shield aggregates them: the origin sees roughly one
  request per object per TTL, regardless of the number of PoPs.
- Higher hit ratio overall: an object evicted from a small edge is still in the larger shield.
- Cost: one more hop on a shield miss, and the shield is a concentration point (CDNs make it redundant).

Inside a PoP, the same object should not be cached separately on every server in it, or the PoP's effective
capacity is divided by the number of servers. PoPs route requests for a given cache key to a specific server
using consistent or rendezvous hashing (topic 13), so each object is stored once per PoP. This multi-level
structure (memory → SSD within a server, server → PoP → shield → origin) is a cache hierarchy, exactly like
CPU caches.

### 1.5 Request collapsing

When many users in one PoP request the same uncached object simultaneously, the edge sends **one** request
upstream and makes the others wait for it: request coalescing (topic 12), usually called **request collapsing**
or collapsed forwarding at CDNs. Combined with a shield, this is what lets a viral object go from zero to
millions of requests without the origin noticing.

### 1.6 What else lives at the edge

- **TLS termination** close to users, with session resumption (topic 05).
- **HTTP/2 and HTTP/3** to clients even if the origin speaks only HTTP/1.1 (topic 06).
- **Compression** (gzip/Brotli) and image optimization (resizing, format conversion).
- **DDoS absorption and a web application firewall (WAF)**: the CDN's total capacity dwarfs any single origin,
  and anycast spreads attack traffic out. Hide the origin (allow only CDN IPs or authenticate CDN-to-origin
  requests) so attackers can't bypass the CDN.
- **Rate limiting and bot management** (topic 45).

---

## 2. Cache keys and HTTP caching headers

A CDN is an HTTP cache shared by many users. Its behavior is governed by HTTP caching semantics (RFC 9111),
mostly controlled by response headers from the origin, plus CDN configuration that can override them.

### 2.1 The cache key

The **cache key** identifies "the same response". By default it is roughly the method plus the full URL:
scheme, host, path and query string. Two requests with the same key get the same cached response.

Getting the key right is the most important CDN design decision:

- **Too specific** → poor hit ratio. If the key includes a query parameter like `?utm_source=newsletter` or a
  random cache-buster, each variation is cached separately. CDNs let you ignore or sort query parameters, or
  include only an allow-list.
- **Too broad** → wrong content served to the wrong user. If the response depends on a cookie, a header
  (language, device), or the user's identity, but the key doesn't include that input, the first user's response
  is served to everyone. This is how account pages end up shown to strangers.

The rule: **the cache key must include every input that changes the response, and nothing else.**

### 2.2 `Cache-Control`

The main header controlling caching. Common response directives:

| Directive | Meaning |
|-----------|---------|
| `max-age=N` | Fresh for N seconds (for all caches) |
| `s-maxage=N` | Fresh for N seconds in **shared** caches (CDN, proxies); overrides `max-age` there |
| `public` | Any cache may store it, even if it would normally not be (e.g. responses to requests with `Authorization`) |
| `private` | Only the user's own browser may store it; shared caches must not |
| `no-cache` | May be stored, but **must be revalidated** with the origin before every use |
| `no-store` | Must not be stored anywhere at all |
| `must-revalidate` | Once stale, must not be served without successful revalidation |
| `immutable` | Will never change during its freshness lifetime; don't even revalidate on reload |
| `stale-while-revalidate=N` | After expiry, may serve stale for up to N seconds while revalidating in the background |
| `stale-if-error=N` | May serve stale for up to N seconds if the origin errors or is unreachable |

Classic confusion: **`no-cache` does not mean "don't cache"**; it means "always check before using". The
directive that means "never store" is `no-store`.

Typical policies:

```
Fingerprinted static assets (app.3f9a2c.js):
    Cache-Control: public, max-age=31536000, immutable

HTML pages that change on deploy:
    Cache-Control: public, max-age=0, s-maxage=60, stale-while-revalidate=300

Personalized pages / API responses with user data:
    Cache-Control: private, no-cache        (or no-store for sensitive data)
```

The older `Expires` header (an absolute date) is superseded by `max-age` when both exist. Without any
freshness information, caches may apply **heuristic freshness** (e.g. 10% of the time since `Last-Modified`),
which is a common source of surprise caching; set headers explicitly.

### 2.3 Freshness and staleness

A cached response is **fresh** while its age is below its freshness lifetime (`s-maxage`/`max-age`), and is
served without contacting the origin. After that it is **stale**. Stale doesn't mean "deleted": the cache can
keep it and **revalidate** it cheaply, or serve it stale if allowed (`stale-while-revalidate`, `stale-if-error`).
The `Age` response header tells you how long a response has been in a cache.

### 2.4 Validators and revalidation: `ETag` and `Last-Modified`

Revalidation asks the origin "has this changed since the version I have?" without transferring the body if it
hasn't. It requires a **validator** on the cached response:

- **`ETag`**: an opaque identifier for a specific version of the representation, e.g. a content hash or a
  version number: `ETag: "a3f9c2"`. Weak ETags (`W/"a3f9c2"`) signal semantic equivalence rather than
  byte-for-byte identity.
- **`Last-Modified`**: a timestamp with one-second resolution; less precise than an ETag.

The cache sends a **conditional request**:

```
Cache (stale copy with ETag "a3f9c2")                Origin
   │  GET /products/42                                 │
   │  If-None-Match: "a3f9c2"                          │
   │ ─────────────────────────────────────────────────>│
   │                                                   │  unchanged?
   │  304 Not Modified   (headers only, no body)       │
   │ <─────────────────────────────────────────────────│
   │  → refresh freshness, keep serving cached body    │
   │                                                   │
   │  (if changed: 200 OK with the new body and new ETag)
```

With `Last-Modified`, the request header is `If-Modified-Since`. A 304 costs a round trip but saves bandwidth
and often origin work (if the origin can compute the ETag cheaply, e.g. from a version column, without rendering
the whole page). Browsers revalidate the same way against the CDN. ETags also enable optimistic concurrency
for writes via `If-Match` (topic 22), a separate use of the same header.

Pitfall: ETags generated from per-server data (inode numbers, timestamps that differ across servers) differ
between origin servers for identical content, so revalidation fails and every check returns a full 200.

### 2.5 `Vary`: when the response depends on request headers

`Vary` tells caches which **request** headers the response depends on, so they add those headers' values to the
cache key:

```
Response: Vary: Accept-Encoding
  → cache stores separate variants for "gzip", "br", and uncompressed clients
```

Uses: `Accept-Encoding` (compression), `Accept-Language` (localized content), `Accept` (image format
negotiation). Dangers:

- **`Vary` on high-cardinality headers destroys hit ratio.** `Vary: User-Agent` creates a variant for every
  distinct browser version string (thousands). `Vary: Cookie` effectively makes every user's response separate,
  which is often equivalent to not caching. CDNs typically normalize (e.g. reduce `User-Agent` to
  mobile/desktop, `Accept-Encoding` to a few values) or let you build the cache key explicitly.
- **Forgetting `Vary`** when the response does depend on a header (e.g. returning different HTML for mobile
  based on `User-Agent` without `Vary`) leads to the wrong variant being served to everyone.

### 2.6 What not to cache, and what goes wrong

- Responses with `Set-Cookie` should generally not be cached by shared caches (one user's session cookie served
  to everyone); many CDNs refuse to cache them by default.
- **Web cache deception**: an attacker tricks a victim into visiting `/account/settings/nonexistent.css`; the app
  ignores the suffix and returns the victim's account page; the CDN sees `.css` and caches it publicly; the
  attacker then fetches the same URL. Defense: cache only based on the origin's `Cache-Control`, not file
  extensions, and make the app reject unexpected paths.
- **Cache poisoning**: the attacker sends a request with an unkeyed header (e.g. `X-Forwarded-Host`) that the
  origin reflects into the response; the poisoned response is cached under the normal key and served to
  everyone. Defense: don't let unkeyed inputs affect responses.

---

## 3. Getting content in and out

### 3.1 Pull vs push CDNs

**Pull (origin pull)**: the CDN fetches content from the origin on the first miss and caches it according to
headers. You only change DNS and headers; content arrives lazily.

**Push**: you upload content to the CDN's storage ahead of time (via API, rsync, etc.); the CDN serves it as if it
were the origin, and there is no origin fetch.

| | Pull | Push |
|--|------|------|
| Setup | Point CDN at origin | Upload pipeline to the CDN |
| First request | Miss, fetched from origin | Already there |
| Storage | Only what's requested, evicted when cold | Everything you upload, until you delete it |
| Updates | Expire via TTL, purge, or new URL | Re-upload |
| Fits | Websites, APIs, frequently changing content, big catalogs with long tails | Large, rarely changing files: software downloads, video libraries, game patches, launches where the first request must be fast |

Most CDNs today are primarily pull, with object storage acting as the origin (which looks like push from your
side: you upload to S3, the CDN pulls from S3).

### 3.2 Updating content: purging vs versioned URLs

When content changes, cached copies everywhere are stale. Two strategies:

**Purging (invalidation)**: tell the CDN to drop a URL (or a set of URLs) from all PoPs.
- Purge by URL, by prefix/wildcard, or by **surrogate key / cache tag** (the origin labels responses with tags like
  `product-42`, and one purge call invalidates every response carrying that tag, e.g. the product page, category
  listings and search results containing it).
- Propagation takes from about a second to minutes depending on the CDN. Some providers rate-limit or charge for
  invalidations.
- It can't reach browsers: users' private caches keep their copy until it expires. So purge only works well with
  short browser TTLs (`max-age` small, `s-maxage` long).
- A "purge everything" turns the whole CDN cold: a cache avalanche on your origin (topic 12). Prefer targeted purges
  and **soft purges** (mark stale rather than delete, so `stale-while-revalidate` can still serve while one request
  refetches).

**Versioned URLs (cache busting / fingerprinting)**: put a version or content hash in the URL
(`/static/app.3f9a2c.js`, `/img/logo.png?v=17`). A change produces a new URL; old and new never collide.
- Cache the files forever (`max-age=31536000, immutable`) in both CDN and browsers.
- No invalidation needed; the old version simply stops being referenced and ages out.
- Atomic deploys: the HTML references a consistent set of asset versions; users mid-session with old HTML still
  get old assets that exist.
- Requires the referencing document (HTML, manifest) to update, and that document itself must have a short TTL or
  be purged. Build tools generate fingerprinted names automatically.

The standard combination: **fingerprinted, immutable assets + short-lived or purgeable HTML**. It mirrors the
versioned-key strategy for application caches (topic 11).

### 3.3 Cache warming at the edge

For a planned launch (a product drop, a big video release), pre-request the objects from multiple PoPs or push
them to the shield so the first real users don't all miss together: the CDN version of cold-start warming
(topic 12).

---

## 4. Caching dynamic content and edge compute

"Static" vs "dynamic" is a spectrum. Many responses that are generated per request are identical for many
users for some time, and caching them even briefly has a huge effect.

### 4.1 Micro-caching

Cache dynamic responses for a very short time: 1–10 seconds. If a page gets 2,000 requests per second, a 1 s
TTL with request collapsing turns that into ~1 origin request per second per PoP (fewer with a shield). Users
see data at most a second old. Excellent for news front pages, product listings, public API endpoints,
leaderboards and live scores. Combine with `stale-while-revalidate` so expiry never costs latency.

### 4.2 Separating shared from personal

A product page for a logged-in user is mostly the same for everyone (description, images, price) with a small
personalized part (name in header, cart count, recommendations). Options:

- **Client-side composition**: cache the shared page at the CDN; the browser fetches personalized fragments via
  an uncached API call (`Cache-Control: private`). The most common approach.
- **Edge Side Includes (ESI)**: the cached page contains placeholders that the edge fills by fetching
  fragments, each with its own caching policy. Supported by Varnish and some CDNs.
- **Edge compute assembly**: code at the edge stitches cached and uncached pieces together.

### 4.3 Caching API responses

GET responses for public, non-personalized resources (catalog, search results for common queries, configuration)
can be cached like pages with `s-maxage`. Pitfalls: the API must emit correct `Cache-Control`; authenticated
requests must not be cached in a shared cache unless the key includes the identity (rarely worth it) or the
response is genuinely user-independent; GraphQL over POST is not cached by default (topic 07), so persisted
queries over GET are used to make it cacheable.

### 4.4 Dynamic site acceleration (when nothing is cacheable)

Even uncacheable requests benefit: TLS terminated near the user, a warm persistent connection pool from edge to
origin, routing over the CDN's optimized backbone, and protocol upgrades (HTTP/3 to the client). This cuts
connection setup round trips and avoids slow public internet paths.

### 4.5 Edge compute

CDNs now let you run code inside PoPs: Cloudflare Workers, Fastly Compute, Lambda@Edge and CloudFront
Functions, among others. Usually lightweight runtimes (V8 isolates or WebAssembly) with millisecond startup and
tight CPU/memory limits.

Good uses:
- Request routing, redirects, rewriting URLs, A/B test assignment, feature flags.
- Building custom cache keys (e.g. normalize headers, include only relevant cookies).
- Authentication checks: verifying a JWT or signed token at the edge and rejecting bad requests before they
  reach the origin (topic 54).
- Personalization and assembly of cached fragments; localization.
- Image resizing and format conversion.
- Bot detection, geofencing, header manipulation.

Limits and trade-offs:
- **Data is the problem, not code.** Edge code runs near the user, but the database usually lives in one region.
  An edge function that makes three queries to a database 150 ms away is slower than running the logic next to
  the database. Edge compute wins when it needs no data, cached data, or edge-replicated data (edge key-value
  stores, which are typically eventually consistent: writes propagate to PoPs over seconds; topic 28).
- Restricted runtimes, limited debugging and observability, vendor lock-in.
- Consistency of state across hundreds of locations is hard; edge-native stores that give strong consistency do
  so by routing to a single home location for each object.

---

## 5. Private content: signed URLs and signed cookies

### 5.1 The problem

You want to serve paid videos, private documents or user uploads through the CDN (for speed and to protect the
origin), but only to authorized users. The CDN is a shared cache and doesn't know about your users or sessions,
and you don't want every request to call back to your auth service.

### 5.2 Signed URLs

Your application, after checking authorization, generates a URL containing an expiry and a cryptographic
signature over the path and parameters. The CDN verifies the signature with a key you configured and serves the
object only if it's valid and unexpired.

```
1. User → App:  "I want video 123"
2. App checks the user is entitled, then generates:
     https://cdn.example.com/videos/123.mp4?Expires=1760000000&KeyId=k1&Signature=Hk3...
     where Signature = sign(secret_or_private_key, "/videos/123.mp4" + Expires [+ optional IP restriction])
3. User → CDN with that URL
4. CDN recomputes/verifies the signature, checks Expires > now
     valid   → serve from cache (or fetch from origin, which only accepts CDN requests)
     invalid → 403
```

Key points:
- **Signing uses either a shared secret (HMAC) or a public/private key pair.** With asymmetric signing (e.g.
  CloudFront), the CDN only holds the public key; only your app can mint URLs. Rotate keys via key IDs.
- **The cache key should exclude the signature parameters**, or every user's distinct signature creates a separate
  cache entry and nothing is shared. The CDN validates the signature, then looks up the object by its path.
- **Short expiry** limits sharing: a leaked URL works only until it expires. Optionally bind to an IP range.
- A signed URL is a **bearer token**: anyone holding it has access until expiry. Don't log it carelessly.
- **Object stores** have the same concept: S3 presigned URLs (for both downloads and direct uploads that bypass
  your servers), GCS signed URLs.

### 5.3 Signed cookies and tokens

A signed URL covers one object. For many objects (all segments of an HLS/DASH video stream, a whole folder), use a
**signed cookie** carrying a policy (a path pattern like `/videos/123/*` plus expiry) and signature, or a signed
token (JWT) that edge compute verifies. The URLs stay clean and cacheable.

### 5.4 Protecting the origin

Private content is only private if the origin can't be reached directly: restrict the origin to accept only
requests from the CDN (origin access control for S3 buckets, IP allow-lists, or a secret header / mTLS between
CDN and origin, topic 10).

---

## 6. Worked example: a product page through a CDN

A shopping site serves `/product/42` (HTML), `/static/app.9c1e.js`, product images, and `/api/cart` (personal).

```
/static/app.9c1e.js     Cache-Control: public, max-age=31536000, immutable
                        → cached forever everywhere; deploy changes the filename

/img/products/42/main.jpg?w=800
                        Cache-Control: public, max-age=86400
                        cache key: path + w param; Vary: Accept (WebP/AVIF) normalized by CDN
                        → updated by uploading under a new path or purging tag "product-42"

/product/42             Cache-Control: public, max-age=0, s-maxage=60, stale-while-revalidate=600
                        Surrogate-Key: product-42 category-7
                        → CDN caches 60 s; browsers always revalidate (ETag → 304)
                        → price change: origin purges tag "product-42"; users see it within seconds
                        → origin down: stale-if-error keeps the page up

/api/cart               Cache-Control: private, no-store
                        → never cached by the CDN; passes through over warm edge→origin connections
```

Timeline for a viral moment: the product goes viral; 50,000 requests per second across 200 PoPs. Each PoP collapses
concurrent misses into one request to the shield; the shield collapses those into one request to the origin per
60 s. The origin sees about one request per minute for that page, plus personalized `/api/cart` calls, which
are what you actually need to scale.

---

## 7. How it connects

- **Topic 01**: CDN as one of the basic building blocks; latency vs throughput.
- **Topic 04**: GeoDNS and anycast steer users to a PoP; DNS TTLs affect failover between CDNs.
- **Topic 05, 06**: TLS termination and HTTP/2/3 at the edge; the handshake savings are a major part of a CDN's benefit.
- **Topic 07**: REST GET semantics make responses cacheable; GraphQL over POST isn't, by default.
- **Topic 09, 10**: a CDN is a geographically distributed reverse proxy and load balancer.
- **Topic 11**: the same patterns (TTL, invalidation, versioned keys, hierarchy) applied over HTTP.
- **Topic 12**: request collapsing, `stale-while-revalidate`, origin shields and warming are the CDN's answers to
  stampedes, avalanches and cold starts.
- **Topic 13**: consistent/rendezvous hashing distributes objects across servers in a PoP.
- **Topic 28**: edge key-value stores are eventually consistent replicas.
- **Topic 44, 45, 48**: the edge sheds load, rate-limits, and serves stale during origin outages; multi-CDN for resilience.
- **Topic 53, 54**: object storage as origin; signed URLs and edge token verification as authorization.

## Common misconceptions

- "`no-cache` means don't cache." → It means store but revalidate before every use; `no-store` means don't store.
- "CDNs are only for static files." → Micro-caching, fragment separation, stale serving and dynamic site
  acceleration help dynamic sites a lot; edge compute runs logic there too.
- "Purging updates what users see immediately." → Purges take time to propagate and can't touch browser caches;
  versioned URLs avoid the problem for assets.
- "Purge everything after each deploy is safe." → It cold-starts the entire CDN onto your origin.
- "Adding `Vary: User-Agent` is harmless." → It fragments the cache into thousands of variants; normalize instead.
- "Signed URLs mean nothing gets cached." → Exclude the signature from the cache key; the object is cached once
  and served to any validly signed request.
- "Edge compute makes everything faster." → Not if the code needs a database far away; data locality decides.
- "If the CDN is in front, the origin is protected." → Only if the origin rejects traffic that doesn't come
  through the CDN.

## Further reading (supplements)

- RFC 9111, *HTTP Caching* (and RFC 9110 for conditional requests); RFC 5861 for `stale-while-revalidate` / `stale-if-error`.
- MDN Web Docs: "HTTP caching", "Cache-Control", "ETag", "Vary".
- Ilya Grigorik, *High Performance Browser Networking* (free online), chapters on latency and HTTP caching.
- Mark Nottingham's caching tutorial and his blog posts on `Vary` and cache keys.
- Cloudflare and Fastly engineering blogs: articles on anycast, tiered caching/origin shield, request collapsing, surrogate keys.
- Omer Gil, *Web Cache Deception Attack* (Black Hat 2017); James Kettle, *Practical Web Cache Poisoning* (PortSwigger, 2018).
- AWS CloudFront documentation: signed URLs vs signed cookies, origin access control.

---

## Questions

1. Explain where the latency savings of a CDN come from for (a) a cached image and (b) an uncacheable,
   personalized API call.
2. Your site uses a CDN with 300 PoPs. A new popular article is published with a 5-minute TTL. Without an origin
   shield, how many origin requests might you see per TTL period, and how does a shield change that?
3. A product listing URL is requested with dozens of different `utm_*` tracking parameters. What happens to the hit
   ratio and how do you fix it?
4. A user's account page shows up for other users. Name two ways a CDN configuration can cause this.
5. What is the difference between `no-cache`, `no-store` and `private`? Give a use case for each.
6. Walk through revalidation with `ETag`. What does the origin save on a 304, and what does it not save?
7. Why is `Vary: User-Agent` usually a bad idea, and what do CDNs do instead?
8. Compare purging with fingerprinted URLs for CSS and JS. Why do you still need purging (or short TTLs) for HTML?
9. After a deploy, the team purges the entire CDN cache. The origin falls over. Explain why and how to deploy safely.
10. A news homepage gets 5,000 requests/s and changes every few seconds. How can you cache it, and what is the
    worst-case staleness?
11. When is a push CDN a better fit than a pull CDN?
12. Edge functions run in 300 locations. Your function reads a user's profile from a Postgres primary in one region.
    Is this faster than running in the origin region? What would make edge compute worthwhile here?
13. Explain how signed URLs work for private videos. Why must the signature be excluded from the cache key, and why
    must the origin reject direct requests?
14. A video is split into 2,000 segment files. Why are signed cookies a better fit than signed URLs?
15. The origin is down for 10 minutes. Which headers let the CDN keep serving the site, and what do users see?

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** (a) The image is served from a nearby PoP: short RTT for TCP/TLS handshakes and data transfer, no trip to
the origin at all. (b) Uncacheable calls still terminate TCP/TLS at the nearby edge (saving several long round
trips), then travel over pre-established, warm, tuned connections on the CDN's backbone to the origin, often a
faster path than the public internet. Only the unavoidable request/response travels the long distance.

**2.** Without a shield, each PoP misses independently, so up to ~300 origin requests per object per 5 minutes
(more without request collapsing inside each PoP). With a shield, all edge misses go to the shield, which fetches
once: about 1 origin request per object per TTL.

**3.** Each distinct query string is a different cache key, so the same content is cached many times and most
requests miss. Configure the CDN to strip or ignore tracking parameters (or use an allow-list of parameters that
affect the response) when building the cache key.

**4.** (a) The response was cacheable in shared caches (`public`/`s-maxage`, or heuristic caching with no
`Cache-Control`) while depending on the session cookie, which isn't in the cache key. (b) Web cache deception: the
CDN caches by file extension (e.g. `.css`) and the app returns the account page for a path with that suffix. Also
caching a response containing `Set-Cookie`, or cache poisoning via unkeyed headers.

**5.** `no-cache`: store, but revalidate every time (HTML that must always be current but benefits from 304s).
`no-store`: never store anywhere (banking pages, one-time tokens). `private`: only the browser may store; shared
caches must not (a personalized dashboard that the browser can reuse briefly).

**6.** The cache holds a stale copy with `ETag: "v1"` and sends `If-None-Match: "v1"`. If unchanged, the origin
replies `304 Not Modified` with no body; the cache resets freshness and serves its copy. Saved: bandwidth, body
transfer time, and possibly rendering work if the origin can compute the ETag cheaply. Not saved: the round trip
to the origin and the request processing needed to determine the current version.

**7.** User-Agent strings have enormous cardinality, so the cache stores a separate variant per browser version
and the hit ratio collapses. CDNs normalize it (e.g. into a device class like mobile/desktop) and include that
normalized value in the cache key, or the origin uses responsive design so no variation is needed.

**8.** Fingerprinted URLs: each version has a unique name, so files can be cached forever in CDN and browsers with
no invalidation, deploys are atomic, and there are no propagation delays. Purging must reach every PoP, takes time,
and can't clear browser caches. HTML is the entry point whose URL can't change (`/product/42`), and it's what
references the new asset filenames, so it needs a short TTL, revalidation or purging.

**9.** Purging everything makes every PoP and the shield cold at once: hit ratio drops to zero, and the origin
receives the full traffic (an avalanche/cold start, topic 12). Safer: fingerprinted assets need no purge; purge only
affected HTML via tags; use soft purge with stale-while-revalidate; and rely on the shield and request collapsing.

**10.** Micro-cache with `s-maxage=1` to `5` plus request collapsing and `stale-while-revalidate`. The origin sees
about one request per TTL per shield (or per PoP), instead of 5,000/s. Worst-case staleness is roughly the TTL plus
the stale-while-revalidate window actually used (usually one refresh time) plus purge propagation if you purge.

**11.** Large, rarely changing content where the first request must be fast or the origin shouldn't serve
it at all: software downloads, game patches, video libraries, launch assets. Also when you want no origin
infrastructure for that content.

**12.** Probably not: each query travels from the edge to the database region (~100+ ms), possibly several times,
whereas origin-region code talks to the DB in ~1 ms. Edge compute pays off if the data is cached or replicated at
the edge (an edge KV store, cached profile fragments), if the work needs no data (routing, auth verification,
personalization from a signed token), or if the function can return early (rejecting bad requests).

**13.** After authorizing the user, the app creates a URL with an expiry and a signature over the path and expiry
(HMAC or private key). The CDN verifies the signature and expiry, then serves the object. If the signature were
part of the cache key, every user's URL would be a separate cache entry and nothing would be shared. The origin
must reject direct requests or anyone could bypass the CDN's signature check by fetching from the origin.

**14.** A signed URL authorizes one object, so you'd have to sign and deliver 2,000 URLs (and rewrite the playlist
for each user). A signed cookie carries a policy for a path pattern (e.g. `/videos/123/*`) and expiry, so one
signature covers all segments and the segment URLs stay clean and shared across users.

**15.** `stale-if-error=N` (and CDN "serve stale on origin error" settings), plus `stale-while-revalidate`.
Users see the last cached version of each page, possibly minutes old; uncached or personalized endpoints still
fail, so a degraded experience rather than a total outage.

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** What is a PoP?
**A:** A CDN point of presence: a site in a metro area with many edge cache servers.

**Q:** Two ways users are routed to a nearby PoP?
**A:** GeoDNS (resolver/client-subnet based answers) and anycast (same IP announced from every PoP).

**Q:** What is the origin?
**A:** Your authoritative servers or object storage that the CDN fetches from on a miss.

**Q:** What does an origin shield do?
**A:** A mid-tier cache that collects edge misses so the origin sees about one fetch per object per TTL.

**Q:** Request collapsing?
**A:** The edge sends one upstream request for many concurrent misses of the same object.

**Q:** How is an object stored once per PoP rather than per server?
**A:** Consistent or rendezvous hashing routes each cache key to one server in the PoP.

**Q:** Rule for the cache key?
**A:** Include every input that changes the response and nothing else.

**Q:** `s-maxage`?
**A:** Freshness lifetime for shared caches (CDNs, proxies), overriding `max-age` there.

**Q:** `no-cache` vs `no-store`?
**A:** `no-cache`: store but revalidate before each use. `no-store`: never store.

**Q:** `private`?
**A:** Only the user's browser may cache it; shared caches must not.

**Q:** `immutable`?
**A:** Content won't change during its lifetime; don't revalidate even on reload.

**Q:** `stale-while-revalidate`?
**A:** Serve stale for N seconds after expiry while refreshing in the background.

**Q:** `stale-if-error`?
**A:** Serve stale for N seconds if the origin errors or is unreachable.

**Q:** What is an ETag?
**A:** An opaque version identifier for a representation, used as a validator.

**Q:** Conditional request headers?
**A:** `If-None-Match` (ETag) and `If-Modified-Since` (Last-Modified).

**Q:** Response for unchanged content on revalidation?
**A:** `304 Not Modified` with no body.

**Q:** What does `Vary` do?
**A:** Adds the named request headers to the cache key so different variants are cached separately.

**Q:** Risk of `Vary: User-Agent` or `Vary: Cookie`?
**A:** Huge numbers of variants, so the hit ratio collapses.

**Q:** Pull vs push CDN?
**A:** Pull fetches lazily from the origin on a miss; push has you upload content ahead of time.

**Q:** Purging vs versioned URLs?
**A:** Purging removes cached copies (slow propagation, can't reach browsers); versioned URLs make every change a new URL that can be cached forever.

**Q:** What are surrogate keys / cache tags?
**A:** Labels on responses that let one purge call invalidate every response with that tag.

**Q:** Why not purge everything on deploy?
**A:** It cold-starts every PoP and floods the origin.

**Q:** Micro-caching?
**A:** Caching dynamic responses for a few seconds, which with collapsing cuts origin load enormously.

**Q:** Main limit of edge compute?
**A:** Data locality: if code needs a far-away database, running at the edge doesn't help.

**Q:** Web cache deception?
**A:** Tricking a CDN into publicly caching a private page served under a static-looking path.

**Q:** What does a signed URL contain?
**A:** The path, an expiry (optionally IP/policy), a key ID and a signature the CDN verifies.

**Q:** Why exclude signature params from the cache key?
**A:** So the object is cached once and served to every validly signed request.

**Q:** Signed cookie vs signed URL?
**A:** A cookie can authorize a path pattern (many objects); a URL authorizes one object.

**Q:** Why lock down the origin?
**A:** So attackers can't bypass the CDN's protections and access controls by hitting the origin directly.
