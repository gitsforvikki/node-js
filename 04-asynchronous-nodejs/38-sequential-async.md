# Lesson 38 — Sequential Async Operations

## Why this lesson matters

In backend systems, not every asynchronous operation should run at the same time.

Some operations **depend on previous results**.

That means they must execute sequentially.

Understanding when to run tasks sequentially vs concurrently is one of the most important async design skills in Node.js.

---

## 1. What are sequential async operations?

Sequential asynchronous operations are tasks that execute one after another.

Example:

```js
const user = await getUser();

const orders = await getOrders(user.id);

const payment = await getPayment(
  orders[0].paymentId
);
```

Execution flow:

```text
getUser
   |
   v
user result
   |
   v
getOrders
   |
   v
orders result
   |
   v
getPayment
```

Each step depends on the previous one.

---

## 2. Sequential does not mean synchronous

This distinction matters.

```js
const user = await getUser();
```

The current async function pauses.

But Node.js can continue handling other work.

So:

```text
sequential async
   !=
blocking entire Node.js process
```

---

## 3. When sequential execution is required

Use sequential async operations when:

- task B needs task A's result
- order matters
- operations mutate shared state in sequence
- rate limits must be respected
- transactions require ordered steps
- the workflow itself is dependent

Example:

```text
Create User
   |
   v
Create Profile using user.id
   |
   v
Generate Welcome Settings
```

Parallel execution would be incorrect because later steps need earlier results.

---

## 4. Real backend example

```js
async function checkout(userId) {
  const cart = await getCart(userId);

  const order = await createOrder(cart);

  const payment = await createPayment(
    order.id,
    order.total
  );

  return {
    order,
    payment,
  };
}
```

This flow is sequential because:

```text
payment needs order.id
order needs cart data
```

---

## 5. Sequential array processing

Example:

```js
for (const userId of userIds) {
  await sendEmail(userId);
}
```

This processes one user at a time.

Timeline:

```text
User 1 ---- done
             |
User 2 ------ done
                    |
User 3 ------------- done
```

This is slower than parallel execution, but may be correct.

---

## 6. When sequential loops are useful

Sequential loops are useful when:

- external API has rate limits
- order must be preserved
- each result affects the next task
- database writes should not race
- resource consumption must remain low

Example:

```js
for (const job of jobs) {
  await processJob(job);
}
```

This limits concurrency naturally to 1.

---

## 7. Sequential reduce pattern

Promises can also be chained using reduce:

```js
await items.reduce(
  async (previous, item) => {
    await previous;

    await processItem(item);
  },
  Promise.resolve()
);
```

This works but is often less readable than:

```js
for (const item of items) {
  await processItem(item);
}
```

Prefer clarity.

---

## 8. Performance cost of unnecessary sequential awaits

Suppose three independent operations each take 500 ms.

```js
const profile = await getProfile();

const notifications =
  await getNotifications();

const settings = await getSettings();
```

Approximate total:

```text
500 + 500 + 500
= 1500 ms
```

If they are independent, sequential execution creates unnecessary latency.

So the key question is:

> Does the next task actually depend on the previous result?

---

## 9. Sequential by dependency

Correct:

```js
const user = await getUser();

const organization =
  await getOrganization(user.organizationId);
```

Incorrect parallel attempt:

```js
await Promise.all([
  getUser(),
  getOrganization(user.organizationId),
]);
```

`user` does not exist yet.

---

## 10. Sequential writes and consistency

Suppose:

```text
1. reserve inventory
2. create payment
3. mark order paid
```

These operations may need strict ordering.

Running them concurrently can create inconsistent states.

Example bad flow:

```js
await Promise.all([
  reserveInventory(),
  chargePayment(),
  markOrderPaid(),
]);
```

This is logically dangerous.

Async design is not just about speed.

Correctness comes first.

---

## 11. Sequential does not automatically mean safe transaction

Even if code is sequential:

```js
await updateInventory();

await createPayment();

await updateOrder();
```

a failure in the middle may leave partial state.

You may need:

- database transactions
- compensating actions
- idempotency
- saga workflows

Sequential execution controls order, not atomicity.

---

## 12. Common mistakes

### Mistake 1 — Running everything sequentially

This creates avoidable latency.

### Mistake 2 — Running dependent work in Promise.all

This breaks logical dependency.

### Mistake 3 — Assuming sequential means transaction-safe

It does not.

### Mistake 4 — Using forEach for sequential work

Bad:

```js
items.forEach(async (item) => {
  await processItem(item);
});
```

Use:

```js
for (const item of items) {
  await processItem(item);
}
```

---

## 13. Interview questions

### What is sequential async execution?

Executing async operations one after another where each next step waits for the previous step to finish.

### When should you use sequential async operations?

When tasks depend on earlier results, ordering matters, concurrency must be limited, or workflows require strict sequencing.

### Is sequential async code blocking?

It blocks only the current async flow at each await, not the whole Node.js process.

### Is sequential execution always slower?

Usually yes compared with safe concurrent execution, but correctness may require it.

---

## 14. Strong interview answer

> Sequential async operations are used when one task depends on the previous result or when order matters. In Node.js this is often written with multiple awaits or a for...of loop. Await pauses only the current async function, so the process can still handle other work. Sequential execution is correct for dependent workflows, but independent I/O should usually not be awaited one by one because that adds unnecessary latency.

---

## Interview-Ready Summary

```text
Sequential Async
   |
   +--> task B waits for task A
   +--> order matters
   +--> dependency matters
   +--> useful for rate limiting
   +--> use for...of for sequential loops

Important:
sequential != blocking Node.js
sequential != transaction
```

## Practice Task

Create:

```text
createAccount()
  -> createUser()
  -> createProfile(user.id)
  -> sendWelcomeEmail(user.email)
```

Implement the flow sequentially and explain why each step must wait.
