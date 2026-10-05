# Lesson 35 — Promises

## Why promises matter

Promises are one of the most important asynchronous concepts in modern JavaScript and Node.js.

They solve several callback-related problems by giving asynchronous operations a standardized object that represents:

```text
future completion
    |
    +--> success
    |
    +--> failure
```

---

## 1. What is a Promise?

A Promise is an object representing the eventual completion or failure of an asynchronous operation.

A promise has three states:

```text
pending
   |
   +--> fulfilled
   |
   +--> rejected
```

Once fulfilled or rejected, it is settled.

A settled promise cannot change state again.

---

## 2. Creating a Promise

```js
const promise = new Promise((resolve, reject) => {
  const success = true;

  if (success) {
    resolve("Operation completed");
  } else {
    reject(new Error("Operation failed"));
  }
});
```

---

## 3. Consuming a Promise

```js
promise
  .then((result) => {
    console.log(result);
  })
  .catch((error) => {
    console.error(error);
  });
```

---

## 4. finally()

```js
promise
  .then(handleSuccess)
  .catch(handleError)
  .finally(() => {
    console.log("Cleanup");
  });
```

`finally` executes when the promise settles, regardless of success or failure.

Typical uses:
- cleanup
- loading-state reset
- releasing resources
- instrumentation

---

## 5. Promise state is immutable after settlement

```js
const promise = new Promise((resolve, reject) => {
  resolve("first");

  resolve("second");

  reject(new Error("third"));
});
```

Only the first settlement matters.

The promise remains fulfilled with:

```text
first
```

---

## 6. Promise chaining

```js
getUser()
  .then((user) => {
    return getOrders(user.id);
  })
  .then((orders) => {
    return getPayment(orders[0].id);
  })
  .then((payment) => {
    console.log(payment);
  })
  .catch((error) => {
    console.error(error);
  });
```

Each `.then()` returns a new promise.

That is why chaining works.

---

## 7. Returning values from then

```js
Promise.resolve(10)
  .then((value) => {
    return value * 2;
  })
  .then((value) => {
    console.log(value);
  });
```

Output:

```text
20
```

A normal returned value becomes the fulfillment value of the next promise in the chain.

---

## 8. Returning a promise from then

```js
getUser()
  .then((user) => {
    return getOrders(user.id);
  })
  .then((orders) => {
    console.log(orders);
  });
```

The chain waits for the returned promise.

This is fundamental.

---

## 9. Promise flattening

Bad:

```js
getUser()
  .then((user) => {
    getOrders(user.id)
      .then((orders) => {
        console.log(orders);
      });
  });
```

This reintroduces nesting.

Better:

```js
getUser()
  .then((user) => {
    return getOrders(user.id);
  })
  .then((orders) => {
    console.log(orders);
  });
```

---

## 10. Error propagation

One major advantage of promises is centralized error propagation.

```js
getUser()
  .then((user) => getOrders(user.id))
  .then((orders) => processOrders(orders))
  .catch((error) => {
    console.error("Something failed:", error);
  });
```

If any promise rejects, the chain skips success handlers until an appropriate rejection handler is found.

---

## 11. Throwing inside then

```js
Promise.resolve("data")
  .then(() => {
    throw new Error("Validation failed");
  })
  .catch((error) => {
    console.log(error.message);
  });
```

Thrown errors become promise rejections.

This gives promises a unified error model.

---

## 12. Promise.resolve()

```js
const promise = Promise.resolve(42);
```

Useful when normalizing values into promise form.

---

## 13. Promise.reject()

```js
const promise = Promise.reject(
  new Error("Failed")
);
```

---

## 14. Promise executor runs immediately

This is an important interview detail.

```js
console.log("A");

new Promise((resolve) => {
  console.log("B");
  resolve();
});

console.log("C");
```

Output:

```text
A
B
C
```

The executor function passed to `new Promise()` runs synchronously.

But `.then()` handlers run asynchronously through the microtask queue.

Example:

```js
console.log("A");

Promise.resolve().then(() => {
  console.log("B");
});

console.log("C");
```

Output:

```text
A
C
B
```

This is extremely important for event loop interview questions.

---

## 15. Promise callbacks and microtasks

Promise handlers:

```js
.then()
.catch()
.finally()
```

are scheduled as microtasks after the current synchronous JavaScript stack finishes.

Simplified model:

```text
Current JS Stack
      |
      v
finishes
      |
      v
Microtask Queue
      |
      +--> promise .then
      +--> promise .catch
      +--> promise .finally
```

More detailed event loop behavior comes later.

---

## 16. Promises do not make work parallel

This is another common misunderstanding.

```js
const promise = new Promise(() => {
  while (true) {}
});
```

Still blocks.

A Promise is an abstraction for completion, not a thread.

---

## 17. Wrapping callback APIs

```js
function readFilePromise(path) {
  return new Promise((resolve, reject) => {
    fs.readFile(path, "utf8", (err, data) => {
      if (err) {
        reject(err);
        return;
      }

      resolve(data);
    });
  });
}
```

Modern Node.js often already provides promise APIs, so do not manually wrap them unnecessarily.

---

## 18. Avoid the Promise constructor antipattern

Bad:

```js
function getUser() {
  return new Promise((resolve, reject) => {
    existingPromiseFunction()
      .then(resolve)
      .catch(reject);
  });
}
```

Better:

```js
function getUser() {
  return existingPromiseFunction();
}
```

Do not create a new Promise when you already have one.

---

## 19. Real backend example

```js
function getDashboard(userId) {
  return getUser(userId)
    .then((user) => {
      return getOrders(user.id);
    })
    .then((orders) => {
      return {
        orderCount: orders.length,
      };
    });
}
```

Async/await is often clearer, but understanding the underlying promise chain is essential.

---

## 20. Common mistakes

### Mistake 1 — Forgetting return

Bad:

```js
getUser()
  .then((user) => {
    getOrders(user.id);
  })
  .then((orders) => {
    console.log(orders);
  });
```

The next `.then` receives `undefined`.

Correct:

```js
return getOrders(user.id);
```

### Mistake 2 — Nesting then unnecessarily

Flatten the chain.

### Mistake 3 — Creating promises for already promise-based APIs

Avoid redundant wrapping.

### Mistake 4 — Assuming promises create threads

They do not.

### Mistake 5 — Forgetting rejection handling

Unhandled promise rejections can cause production problems.

---

## 21. Interview questions

### What is a Promise?

An object representing the eventual fulfillment or rejection of an asynchronous operation.

### What are Promise states?

- pending
- fulfilled
- rejected

Fulfilled and rejected promises are settled.

### Can a settled Promise change state?

No.

### What does then return?

A new Promise.

### What happens if you throw inside then?

The returned promise becomes rejected with that error.

### When does a Promise executor run?

Synchronously when the Promise is constructed.

### When do then callbacks run?

As microtasks after the current synchronous stack completes.

---

## 22. Strong interview answer

> A Promise represents the future result of an asynchronous operation. It starts in the pending state and can settle exactly once as fulfilled or rejected. Promise chaining works because every `.then()` returns a new promise. Returning another promise from a handler causes the chain to wait for it, while throwing an error converts that step into a rejection. Promise handlers execute through the microtask queue, which is important for understanding Node.js execution order.

---

## Interview-Ready Summary

```text
Promise
   |
   +--> pending
   |
   +--> fulfilled
   |
   +--> rejected

then()
   -> returns new promise

throw
   -> rejection

return promise
   -> chain waits

then/catch/finally
   -> microtasks
```

## Practice Task

Implement:

```text
wait(ms)
```

using a Promise.

Then create a chain that:
1. waits 1 second
2. returns a value
3. transforms the value
4. throws an error
5. catches the error
