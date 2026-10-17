# Lesson 186 — HTTP Compression

## 1. Why compress responses?
API responses and static files travel over a network. **Compression** reduces bytes transferred, potentially speeding up responses on slower networks.

Example: a large repetitive JSON payload might shrink greatly under gzip; a JPEG or already compressed ZIP often shrinks little.

```text
Server JSON (large)
  -> gzip or Brotli compression
  -> fewer network bytes
  -> browser decompresses
  -> original JSON
```

Compression trades **CPU time** for smaller response size.

## 2. Content negotiation
The client advertises support:
```http
Accept-Encoding: gzip, br
```
The server may choose:
```http
Content-Encoding: gzip
Vary: Accept-Encoding
```
`Vary` matters for caches: they must not serve a compressed representation to a client that cannot decode it.

## 3. Express example
```bash
npm install compression
```
```js
import express from "express";
import compression from "compression";

const app = express();
app.use(compression({ threshold: "1kb" }));

app.get("/api/products", (req, res) => {
  res.json({
    products: Array.from({ length: 2000 }, (_, i) => ({
      id: i + 1, name: `Product ${i % 20}`,
    })),
  });
});
app.listen(3000);
```
The `compression` middleware can compress suitable responses based on client support and response headers. This example focuses on middleware behavior; your reverse proxy/CDN may already compress responses.

## 4. Test in curl
```bash
curl -sS -D - -o /dev/null -H "Accept-Encoding: gzip" \
  http://127.0.0.1:3000/api/products
```
Inspect `Content-Encoding`; whether compression activates depends on response content, headers, middleware options and size. To download/decode automatically, use `curl --compressed`.

## 5. Gzip vs Brotli
- **gzip:** widely supported, solid general-purpose compression.
- **Brotli:** often better compression for text/static assets, with CPU cost depending on settings.
- **No compression:** suitable for very small or already compressed payloads.

CDNs/reverse proxies often offer production-grade compression and caching. Avoid accidental double compression; measure end-to-end latency and CPU.

## 6. Security consideration: compression side channels
When responses include **secret values near attacker-controlled input**, compression-size differences can leak information under certain conditions (e.g., BREACH-style attacks).

For sensitive responses, assess whether dynamic compression is appropriate; avoid reflecting untrusted input next to secrets and consider disabling compression for particular responses. HTTPS alone does not eliminate compression side-channel risk.

## 7. What else speeds responses?
- Avoid fetching/serializing unnecessary fields.
- Paginate large lists.
- Cache repeated data.
- Use appropriate HTTP caching.
- Serve static files through CDN.
- Fix slow DB queries.

Compression does **not** make a slow MongoDB query fast; it reduces transport payload size after generating the response.

## 8. Common mistakes
Compressing already compressed media indiscriminately; turning compression level to maximum without measuring CPU; ignoring reverse proxy settings; confusing transfer size with query latency; exposing secrets to compression side channels.

## Practice
Compare a large JSON endpoint with and without compression, recording response size and CPU/latency. Then compare a JPEG or ZIP file.

**Interview answer:** HTTP compression reduces network bytes, usually using gzip/Brotli negotiation. It is most beneficial for text/JSON, costs CPU, interacts with caches and CDNs, and should be measured rather than enabled blindly.
