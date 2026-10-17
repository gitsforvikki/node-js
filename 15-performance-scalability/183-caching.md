# Lesson 183 — Caching

## 1. What is a cache?
A **cache** stores the result of an expensive operation so later requests can reuse it.

Example: 1,000 people request a public product list. Instead of querying MongoDB 1,000 times, a short-lived cache can serve repeated requests faster.

```text
Request -> cache lookup
           | HIT  -> cached response
           | MISS -> query DB -> store with expiry -> response
```
Caching can reduce latency and database load, but **stale or incorrect cached data creates bugs**.

## 2. Terms
- **Hit / miss:** key found / not found.
- **TTL:** time to live (expiry duration).
- **Eviction:** removing entries (on expiry, size limit, or policy).
- **Cache-aside:** application reads cache first and populates it on a miss.
- **Write-through:** application writes through cache to backing store.
- **Invalidation:** removing/changing entries when source data changes.

## 3. Small in-memory cache example
```js
const cache = new Map();
const TTL_MS = 30_000;

async function listActiveProducts() {
  const entry = cache.get("active-products");
  if (entry && entry.expiresAt > Date.now()) return entry.value;

  const products = await Product.find({ active: true })
    .select("name price").lean();

  cache.set("active-products", {
    value: products, expiresAt: Date.now() + TTL_MS,
  });
  return products;
}
```
This shows **cache-aside**. But `Map` has no automatic expiry cleanup or maximum size: production caches need a bound/eviction policy. Also, each Node process gets its own Map; instances can disagree.

## 4. Cache invalidation
If a product price changes, the old cached list is wrong. Invalidate it after successful update:
```js
await Product.updateOne({ _id: id }, { $set: { price } });
cache.delete("active-products");
```
With multiple servers or database writers, local invalidation is incomplete. Use a shared cache, event-driven invalidation or suitably short TTLs. **Ordering matters:** cache consistency needs careful design when writes and concurrent misses race.

## 5. Cache keys must reflect inputs and permissions
Bad:
```js
cache.set("profile", userProfile); // one key for everyone's private profile!
```
Safer idea: separate keys per verified user and relevant filters, but consider whether private data should be cached at all:
```js
const key = `profile:${authenticatedUserId}`;
```
Never use request-provided IDs as proof of ownership; authorization must still happen. Avoid caching sensitive session information in uncontrolled public caches.

## 6. Cache stampede (thundering herd)
A popular key expires, and 100 requests miss simultaneously → 100 database queries. Mitigations include request coalescing (single-flight), probabilistic expiry jitter, stale-while-revalidate, and carefully designed locks.

## 7. Other caching layers
| Layer | Example |
| --- | --- |
| Browser/HTTP | `Cache-Control`, `ETag`, `304 Not Modified` |
| CDN | Cache public static assets close to users |
| Application local memory | Bounded LRU cache |
| Shared distributed cache | Redis |
| Database | DB-managed page/buffer cache |

For personal responses, use appropriate HTTP controls such as `Cache-Control: private` or `no-store`. Avoid `public` caches for authenticated private data.

## 8. What to measure
Cache hit ratio, p95 latency, DB query rate, memory used, stale-response incidents, and eviction rate. A cache is useful when its benefit exceeds its complexity and consistency cost.

## 9. Mistakes
Caching everything; forgetting invalidation; unbounded Maps; leaking private data between user keys; assuming cache is the source of truth; retrying all misses during outages.

## Practice
Cache a public product list with 30-second TTL. Update a product and demonstrate stale data before adding invalidation. Explain why two Node replicas need a shared strategy.

**Interview answer:** Caching trades freshness and complexity for lower latency and reduced load. Design keys, TTL, invalidation, bounds, failure behavior and permissions; verify hit rate and user-visible speed.
