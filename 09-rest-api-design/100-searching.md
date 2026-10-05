# Lesson 100 — Searching

## Why search is different from filtering

Filtering usually means exact or structured constraints:

```text
status=paid
category=laptop
price between 1000 and 5000
```

Searching usually means matching text relevance:

```text
?q=wireless headphones
```

Search design can range from simple SQL LIKE queries to dedicated search engines.

---

## 1. Basic search endpoint

Example:

```text
GET /products?q=iphone
```

---

## 2. Naive substring search

SQL example:

```sql
WHERE name ILIKE '%iphone%'
```

Simple and useful for small datasets.

But:

```text
leading wildcard
%iphone%
```

can make normal B-tree indexes ineffective.

---

## 3. Prefix search

```sql
WHERE name ILIKE 'iph%'
```

This can be more index-friendly depending on DB and collation/index configuration.

---

## 4. Case-insensitive search

In PostgreSQL:

```text
ILIKE
```

can perform case-insensitive pattern matching.

Other databases use different approaches.

---

## 5. Search multiple fields

Example:

```text
q=node
```

Search:

- title
- description
- tags

Concept:

```sql
WHERE title ILIKE ...
OR description ILIKE ...
```

This becomes expensive as data grows.

---

## 6. Full-text search

Databases such as PostgreSQL provide full-text search capabilities.

Conceptually:

```text
document
   |
   v
tokenization
   |
   v
search vector/index
   |
   v
query
   |
   v
ranked matches
```

This is more powerful than simple substring matching.

---

## 7. Relevance ranking

Search results should often be ranked by relevance.

Example:

```text
query: "node streams"

Result A:
title contains both words

Result B:
description contains one word
```

A search system may rank A higher.

---

## 8. Dedicated search engines

At larger scale or with advanced requirements, use systems such as:

- Elasticsearch/OpenSearch
- Meilisearch
- Typesense
- hosted search services

Useful for:
- typo tolerance
- faceting
- ranking
- autocomplete
- synonyms
- large-scale full-text indexing

---

## 9. Database search vs search engine

Use DB search when:

- dataset moderate
- search simple
- consistency important
- operational simplicity matters

Use dedicated search when:

- relevance quality matters
- typo tolerance
- autocomplete
- complex faceting
- high search scale

---

## 10. Search index synchronization

If data lives in DB but search happens in Elasticsearch:

```text
Database
   |
   v
change event / queue / CDC
   |
   v
Search Index
```

Now you have consistency considerations.

Search index may be eventually consistent.

---

## 11. Search + filters

Example:

```text
GET /products?q=iphone&category=mobile&minPrice=50000
```

Good flow:

```text
text search
   +
structured filters
   |
   v
rank/sort
   |
   v
paginate
```

---

## 12. Search + pagination

Cursor pagination can be harder when sorting by relevance score.

You may need cursor fields such as:

```text
score
id
```

Search engines often provide their own pagination primitives.

---

## 13. Search query limits

Do not accept unlimited query length.

Example policy:

```text
minimum 2 characters
maximum 100 characters
```

This protects:
- DB
- logs
- search backend

---

## 14. Debouncing belongs to the client

For live search UI:

```text
user types:
n
no
nod
node
```

Frontend should often debounce requests.

Backend still needs:
- rate limiting
- caching
- efficient queries

---

## 15. SQL injection

Never build:

```js
const sql =
  `SELECT * FROM products
   WHERE name LIKE '%${q}%'`;
```

Use parameterized values.

---

## 16. Wildcard escaping

If you allow LIKE search, understand that:

```text
%
_
```

have wildcard meaning.

Depending on requirements, escape them or intentionally support them.

---

## 17. Normalization

Search may normalize:
- case
- whitespace
- punctuation
- accents
- stemming

Example:

```text
"Node.js"
"node js"
"NODEJS"
```

Whether these match depends on search strategy.

---

## 18. Typo tolerance

Query:

```text
javasript
```

Should it match:

```text
javascript
```

A simple SQL LIKE usually will not.

Dedicated search systems may.

---

## 19. Autocomplete

Autocomplete search often uses a separate optimized index or prefix strategy.

Example:

```text
"rea"
  -> react
  -> react native
  -> react router
```

Do not run expensive full-table substring scans on every keystroke.

---

## 20. Search result shape

Example:

```json
{
  "data": [],
  "meta": {
    "query": "node",
    "tookMs": 12
  },
  "pagination": {
    "nextCursor": "..."
  }
}
```

Latency metadata is optional.

---

## 21. Search caching

Popular searches may be cached.

Example:

```text
q=iphone
category=mobile
page/cursor
```

Cache key must include all relevant search/filter/sort inputs.

---

## 22. Common mistakes

### Mistake 1
Using `LIKE '%q%'` forever as data grows.

### Mistake 2
Building raw SQL strings.

### Mistake 3
No query length limits.

### Mistake 4
No indexes/search vectors.

### Mistake 5
Ignoring search index consistency.

### Mistake 6
Confusing relevance ranking with normal sorting.

---

## 23. Interview questions

### Filtering vs searching?

Filtering uses structured field constraints; searching usually matches text by relevance.

### When is LIKE enough?

For small/simple datasets and low search requirements.

### When use Elasticsearch/OpenSearch?

When you need large-scale full-text search, relevance ranking, typo tolerance, autocomplete, or faceting.

### What trade-off appears with external search index?

Eventual consistency and index synchronization complexity.

---

## 24. Strong interview answer

> I separate structured filtering from text search. For small datasets, parameterized database search such as prefix or full-text search may be enough. As requirements grow toward ranking, typo tolerance, autocomplete, or faceting, I move to a dedicated search engine. I validate query length, combine search with structured filters safely, index the searchable fields, and account for eventual consistency if the search index is separate from the source database.

---

## Interview-Ready Summary

```text
Filtering
  -> exact/structured

Searching
  -> text/relevance

Options:
LIKE
DB full-text search
Search engine

Consider:
indexes
ranking
typos
autocomplete
consistency
pagination
```

## Practice Task

Design search for an e-commerce product catalog with:

- q
- category
- brand
- price range
- relevance sorting
- pagination

Explain when you would move from PostgreSQL search to a dedicated search engine.
