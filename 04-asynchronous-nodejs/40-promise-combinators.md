# Lesson 40 — Promise.all, allSettled, race and any

## Why Promise combinators matter

Promise combinators let you coordinate multiple asynchronous operations.

The four most important are:

```text
Promise.all
Promise.allSettled
Promise.race
Promise.any
```

You should know not only their syntax, but also their failure behavior.

That is what interviews usually test.

---

# 1. Promise.all

Use when:

> All operations must succeed.

```js
const results = await Promise.all([
  taskA(),
  taskB(),
  taskC(),
]);
```

Result:

```js
[
  resultA,
  resultB,
  resultC
]
```

Order matches input order, not completion order.

---

## Promise.all success behavior

Suppose:

```text
Task A -> 800 ms
Task B -> 200 ms
Task C -> 500 ms
```

Result order is still:

```text
[A, B, C]
```

not:

```text
[B, C, A]
```

---

## Promise.all rejection behavior

If one promise rejects:

```js
await Promise.all([
  fetchUser(),
  fetchOrders(),
  fetchPayment(),
]);
```

then the combined promise rejects.

Important:

```text
Promise.all rejects early
BUT
already-started tasks are not automatically cancelled
```

---

## Best use cases for Promise.all

- dashboard data
- independent DB queries
- multiple API calls
- loading independent configuration
- parallel validation calls

---

# 2. Promise.allSettled

Use when:

> I need the outcome of every operation, even if some fail.

```js
const results =
  await Promise.allSettled([
    sendEmail(),
    sendSms(),
    sendPush(),
  ]);
```

Possible result:

```js
[
  {
    status: "fulfilled",
    value: "email sent"
  },
  {
    status: "rejected",
    reason: Error("SMS failed")
  },
  {
    status: "fulfilled",
    value: "push sent"
  }
]
```

---

## Best use cases for allSettled

- multiple notifications
- batch jobs
- independent cleanup operations
- partial success workflows
- reporting all failures

Example:

```text
Send:
  email
  SMS
  push

If SMS fails,
still keep email/push result.
```

---

# 3. Promise.race

Use when:

> I care about whichever promise settles first.

```js
const result = await Promise.race([
  taskA(),
  taskB(),
]);
```

Important:

The first settled promise wins.

That can be:
- fulfillment
- rejection

---

## Timeout pattern

```js
function timeout(ms) {
  return new Promise((_, reject) => {
    setTimeout(() => {
      reject(
        new Error("Operation timed out")
      );
    }, ms);
  });
}

await Promise.race([
  fetchData(),
  timeout(3000),
]);
```

This rejects if timeout settles first.

---

## Important race limitation

The losing operation is not automatically cancelled.

Example:

```text
fetchData still running
timeout wins
Promise.race rejects

fetchData may continue
```

For real cancellation, use:
- AbortController
- AbortSignal
- API-specific cancellation support

---

# 4. Promise.any

Use when:

> I want the first successful result.

```js
const result = await Promise.any([
  serverA(),
  serverB(),
  serverC(),
]);
```

Rejected promises are ignored until:
- one fulfills
- or all reject

---

## Promise.any failure

If every promise rejects:

```js
await Promise.any([
  Promise.reject("A"),
  Promise.reject("B"),
]);
```

it rejects with an:

```text
AggregateError
```

This error contains multiple failure reasons.

---

## Best use cases for Promise.any

- fallback servers
- redundant endpoints
- fastest successful cache/provider
- multi-region reads

Example:

```text
Region A fails
Region B slow
Region C succeeds

Promise.any
   -> Region C result
```

---

# 5. Quick comparison

| Method | Resolves when | Rejects when | Best use |
|---|---|---|---|
| Promise.all | all fulfill | first rejection | all required |
| Promise.allSettled | all settle | does not reject due to input rejection | inspect every result |
| Promise.race | first settles | if first settlement is rejection | timeout / fastest settlement |
| Promise.any | first fulfills | all reject | first successful result |

---

# 6. Interview trap: all vs allSettled

Question:

> You send email, SMS and push notification. Even if one fails, you still want results from the others. Which method?

Correct:

```text
Promise.allSettled
```

---

# 7. Interview trap: race vs any

```text
race
  -> first settled
  -> success OR failure

any
  -> first fulfilled
  -> ignores rejection until all fail
```

This is a very common interview comparison.

---

# 8. Promise.all with non-Promise values

```js
const result = await Promise.all([
  Promise.resolve(1),
  2,
  Promise.resolve(3),
]);
```

Result:

```js
[1, 2, 3]
```

Non-Promise values are treated as already fulfilled.

---

# 9. Empty arrays

```js
await Promise.all([]);
```

resolves immediately with:

```js
[]
```

Likewise:

```js
await Promise.allSettled([]);
```

returns:

```js
[]
```

---

# 10. Error design matters

Suppose:

```js
await Promise.all([
  createInvoice(),
  sendEmail(),
  updateAnalytics(),
]);
```

This may be a bad design.

Why?

If email fails:
- invoice may already exist
- analytics may still complete

Promise combinators do not give transactional guarantees.

Use them only when the operation semantics make sense.

---

# 11. Timeout with AbortController

Better timeout systems often combine timeout logic with cancellation.

Conceptual pattern:

```js
const controller = new AbortController();

const timer = setTimeout(() => {
  controller.abort();
}, 3000);

try {
  const response = await fetch(url, {
    signal: controller.signal,
  });
} finally {
  clearTimeout(timer);
}
```

This actually attempts to stop the underlying operation.

---

# 12. Common mistakes

### Mistake 1 — Using all when partial success is acceptable

Use allSettled.

### Mistake 2 — Using race expecting first success

Use any.

### Mistake 3 — Assuming combinators cancel other promises

They do not automatically.

### Mistake 4 — Ignoring side effects

Already-started operations may still mutate state.

### Mistake 5 — Using huge Promise.all arrays

Can overload resources.

---

# 13. Interview questions

### Difference between Promise.all and allSettled?

`all` rejects when one input rejects. `allSettled` waits for every input and returns each outcome.

### Difference between Promise.race and Promise.any?

`race` resolves or rejects based on the first settled promise. `any` returns the first fulfilled promise and rejects only if all inputs reject.

### Does Promise.all cancel remaining promises after rejection?

No.

### What error does Promise.any produce if all inputs fail?

`AggregateError`.

---

# 14. Strong interview answer

> Promise.all is used when every operation is required and rejects when one input rejects. Promise.allSettled waits for every operation and gives each outcome, making it useful for partial-success workflows. Promise.race settles with the first settled promise, whether success or failure, which makes it useful for timeout-style patterns. Promise.any resolves with the first successful promise and rejects with AggregateError only when all inputs reject. None of these combinators automatically cancel the remaining work.

---

## Interview-Ready Summary

```text
Promise.all
   -> all required

Promise.allSettled
   -> inspect every result

Promise.race
   -> first settled

Promise.any
   -> first successful

Important:
no automatic cancellation
no transaction guarantee
```

## Practice Task

Implement four examples:

1. dashboard with `Promise.all`
2. notification system with `Promise.allSettled`
3. timeout with `Promise.race`
4. fallback providers with `Promise.any`
