# Lesson 27 — Events and EventEmitter in Node.js

## 1. Event kya hota hai?

**Event** ka matlab hai koi action ya activity hona, jiske response mein hum function chala sakte hain.

Examples:
- User button click kare → `click` event
- Server ko HTTP request mile → `request` event
- Order place ho → custom `orderPlaced` event
- Koi operation fail ho → `error` event

**Real-life example:** Restaurant mein customer order deta hai. Order milte hi kitchen cooking aur waiter notification jaisi activities start kar sakte hain. Order place hona *event* hai; in activities ko handle karne wale functions *listeners* hain.

Node.js ka bahut saara I/O event-driven hai.

## 2. EventEmitter kya hai?

Node.js ka built-in `node:events` module `EventEmitter` class provide karta hai. Isse hum apne events aur unke listeners bana sakte hain.

| Term | Simple meaning |
| --- | --- |
| Event | Ek named action, jaise `orderPlaced` |
| Emitter | Object jo event trigger karta hai |
| Listener | Function jo event hone par chalaya jata hai |
| `on()` | Listener register karta hai |
| `emit()` | Event trigger karta hai |

**Flow:**

```text
emitter.on("orderPlaced", callback)  →  listener register
emitter.emit("orderPlaced")         →  event trigger
                                    →  callback execute
```

## 3. First Practical Example

```js
import { EventEmitter } from "node:events";

const emitter = new EventEmitter();

// 1. Listener register: abhi function execute nahi hoga
emitter.on("greet", () => {
  console.log("Hello Vikash!");
});

// 2. Event trigger: registered listener chalega
emitter.emit("greet");
```

**Output:**

```text
Hello Vikash!
```

**Step-by-step:**
1. `new EventEmitter()` ek emitter object banata hai.
2. `on("greet", callback)` Node ko batata hai: *jab greet event aaye, ye function chalao.*
3. `emit("greet")` event trigger karta hai aur listener execute hota hai.

> Important: `on()` listener ko register karta hai; **`emit()` use trigger karta hai**.

## 4. Ek Event, Multiple Listeners

Ek event ko multiple listeners handle kar sakte hain. Default mein listeners registration order mein run hote hain.

```js
import { EventEmitter } from "node:events";

const emitter = new EventEmitter();

emitter.on("orderPlaced", () => {
  console.log("1. Order received");
});

emitter.on("orderPlaced", () => {
  console.log("2. Send confirmation email");
});

emitter.on("orderPlaced", () => {
  console.log("3. Notify restaurant");
});

emitter.emit("orderPlaced");
```

**Output:**

```text
1. Order received
2. Send confirmation email
3. Notify restaurant
```

Ek hi `emit()` call ne teeno listeners chalaye. Ye example sirf console log karta hai; actual email bhejne ke liye email service integrate karni hogi.

## 5. Event ke Saath Data Pass Karna

`emit(eventName, ...args)` ke extra arguments listeners ko milte hain.

```js
import { EventEmitter } from "node:events";

const emitter = new EventEmitter();

emitter.on("orderPlaced", (order) => {
  console.log("Order ID:", order.id);
  console.log("Customer:", order.customer);
  console.log("Amount:", order.amount);
});

emitter.emit("orderPlaced", {
  id: 101,
  customer: "Vikash",
  amount: 501,
});
```

**Output:**

```text
Order ID: 101
Customer: Vikash
Amount: 501
```

Multiple values bhi pass kar sakte hain:

```js
emitter.on("message", (roomId, message) => {
  console.log(roomId, message);
});

emitter.emit("message", "room-1", "Hello");
// Output: room-1 Hello
```

## 6. `on()`, `once()` aur `off()`

| Method | Kya karta hai? |
| --- | --- |
| `on(name, fn)` | Har baar event aane par listener chalata hai |
| `once(name, fn)` | Sirf pehli baar chalata hai; phir listener remove |
| `off(name, fn)` | Usi function reference wale listener ko remove karta hai |
| `removeListener(name, fn)` | `off()` jaisa kaam karta hai |

```js
import { EventEmitter } from "node:events";

const emitter = new EventEmitter();

function onLogin() {
  console.log("User logged in");
}

emitter.on("login", onLogin);

emitter.once("login", () => {
  console.log("Welcome for the first time");
});

emitter.emit("login");
emitter.emit("login");

emitter.off("login", onLogin);
emitter.emit("login"); // Ab koi output nahi
```

**Output:**

```text
User logged in
Welcome for the first time
User logged in
```

**Dhyan do:** `off()` ke liye same function reference chahiye; alag arrow function pass karne se pehla listener remove nahi hota.

## 7. Kya EventEmitter Asynchronous Hai?

**Nahi. `emit()` listeners ko by default synchronously invoke karta hai.**

```js
import { EventEmitter } from "node:events";

const emitter = new EventEmitter();

console.log("Start");

emitter.on("test", () => {
  console.log("Event executed");
});

emitter.emit("test");
console.log("End");
```

**Output:**

```text
Start
Event executed
End
```

### Listener ke Andar Async Work

```js
emitter.on("asyncTask", async () => {
  await new Promise((resolve) => setTimeout(resolve, 1000));
  console.log("Async work completed");
});

emitter.emit("asyncTask");
console.log("End");
```

**Output (approximately 1 second baad second line):**

```text
End
Async work completed
```

Reason: `emit()` async listener se returned Promise ko automatically **await nahi karta**. Async listener mein rejected Promise ko explicitly handle karna zaroori hai; `captureRejections` bhi ek option hai, lekin error handling phir bhi sahi honi chahiye.

## 8. Real Node.js Example — HTTP Request Event

`http.createServer()` se bana server EventEmitter-based hota hai. HTTP request aane par `request` event trigger hota hai.

```js
import http from "node:http";

const server = http.createServer();

server.on("request", (req, res) => {
  console.log("New HTTP request received");
  res.end("Hello from Node.js");
});

server.listen(3000, () => {
  console.log("Server running on port 3000");
});
```

Browser mein `http://localhost:3000` kholo.

**Terminal (ek request ke liye):**

```text
Server running on port 3000
New HTTP request received
```

**Browser response:**

```text
Hello from Node.js
```

`http.createServer((req, res) => {...})` wali callback bhi internally `request` event listener ke roop mein register hoti hai.

Streams aur sockets bhi event-based APIs use karte hain.

## 9. Special `error` Event

`EventEmitter` mein `error` special event hai. Agar `error` emit ho aur listener na ho, Node.js error throw karta hai; unhandled raha toh process terminate ho sakta hai.

```js
import { EventEmitter } from "node:events";

const emitter = new EventEmitter();

emitter.on("error", (error) => {
  console.log("Error handled:", error.message);
});

emitter.emit("error", new Error("Database failed"));
```

**Output:**

```text
Error handled: Database failed
```

Production mein error ko log karna, recover karna ya relevant layer tak propagate karna situation par depend karta hai.

## 10. Listener Leaks

Agar tum repeatedly listeners register karte rahoge aur cleanup nahi karoge, unnecessary references memory mein reh sakte hain.

Node.js kabhi `MaxListenersExceededWarning` warn karta hai (default warning threshold usually 10 listeners per event). Ye automatically proven memory leak nahi hai, lekin investigate karna chahiye.

**Avoid repeated registration:**

```js
// Bad: every request par same emitter mein naya listener add ho raha hai
function handleRequest() {
  emitter.on("orderPlaced", sendEmail);
}

// Better: app initialization mein ek baar register karo
emitter.on("orderPlaced", sendEmail);
```

Sirf limit badhakar warning hide na karo; pehle lifecycle samjho.

## 11. Industry Example — Event-Driven Order Service

**Directly coupled design:**

```text
createOrder()
  → save order
  → send email
  → update analytics
  → create audit log
```

**Event-based design:**

```text
createOrder()
  → save order
  → emit("orderCreated", order)
         ├── email listener
         ├── analytics listener
         └── audit listener
```

Is approach se responsibilities separate hoti hain. Ek simple illustration:

```js
import { EventEmitter } from "node:events";

const orderEvents = new EventEmitter();

orderEvents.on("orderCreated", (order) => {
  console.log("Email for order:", order.id);
});

orderEvents.on("orderCreated", (order) => {
  console.log("Analytics updated:", order.id);
});

function createOrder(order) {
  // Real application mein pehle database save successfully complete karo
  console.log("Order saved:", order.id);
  orderEvents.emit("orderCreated", order);
}

createOrder({ id: 101, amount: 501 });
```

**Output:**

```text
Order saved: 101
Email for order: 101
Analytics updated: 101
```

### EventEmitter vs Message Queue

| EventEmitter | Message queue (e.g. RabbitMQ) |
| --- | --- |
| Same Node.js process ke andar | Multiple processes/services ke beech |
| In-memory | Durable delivery support ho sakti hai |
| Built-in aur lightweight | Separate infrastructure ho sakta hai |
| Process crash par pending event lost ho sakta hai | Persistence/retries configure kar sakte hain |

**Important:** EventEmitter durable message queue nahi hai. Critical business operations ke liye reliable queue/outbox patterns consider karo.

## 12. EventEmitter vs Event Loop

Dono naam similar hain, par kaam alag hai:

- **EventEmitter:** Named events register aur trigger karne ka API (`on`, `emit`).
- **Event Loop:** Node.js runtime ka mechanism jo async I/O ke ready callbacks aur scheduling ko manage karta hai.

`emit()` directly synchronous listeners chalata hai; `emit()` khud event loop mein task queue nahi karta.

## 13. Interview Questions

**Q1. Node.js EventEmitter kya hai?**  
Built-in class jisse named events emit karke listeners execute kar sakte hain.

**Q2. `on()` aur `emit()` mein difference?**  
`on()` callback register karta hai, `emit()` event trigger karta hai.

**Q3. Ek event ke multiple listeners ho sakte hain?**  
Haan. By default registration order mein execute hote hain.

**Q4. Kya EventEmitter async hai?**  
Listeners ka invocation synchronous hota hai; listeners apna async work schedule kar sakte hain.

**Q5. `once()` kyun use karte hain?**  
Jab listener ko sirf first event par chalana ho.

**Q6. Unhandled `error` event ka kya hoga?**  
Error throw ho sakta hai, jo unhandled hone par process terminate kar sakta hai.

**Q7. Kya EventEmitter Kafka/RabbitMQ/BullMQ replace kar sakta hai?**  
Nahi. EventEmitter same-process, non-durable event dispatch ke liye hai.

## 14. Practice Tasks

### Task A — User Registration

`userRegistered` event emit karo jisme email address pass ho. Teen listeners banao:
1. `User registered successfully`
2. `Welcome email sent to vikash@gmail.com`
3. `User profile created`

**Starter code:**

```js
import { EventEmitter } from "node:events";

const emitter = new EventEmitter();

// TODO: register 3 listeners

emitter.emit("userRegistered", "vikash@gmail.com");
```

### Task B — Order Lifecycle

Do events banao:
- `orderCreated`
- `orderCancelled`

Har event ke liye email, analytics aur audit-log listeners register karo. `emit()` ke saath order object pass karke output inspect karo.

---

**Quick revision:** `EventEmitter` → `on()` register → `emit()` trigger → listener runs; `once()` runs one time, `off()` removes listener. Yaad rakho ki listeners default mein **synchronous** hain.
