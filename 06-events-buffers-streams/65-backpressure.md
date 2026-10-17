# Lesson 65 — Backpressure

## Start here — understand the situation before the solution

### 1. What problem are we trying to solve?

Imagine you are filling a water tank:

- Water comes **in** at 10 liters per second.
- Water goes **out** at only 3 liters per second.
- Extra water collects in the tank.
- If you never slow the incoming water, the tank may overflow.

A Node.js stream has a similar problem:

```text
Readable Stream (fast)       Writable Stream (slow)
   produces chunks  ------>   writes chunks
          \                     /
           Extra data waits in a buffer
```

This is called a **speed mismatch**: the producer makes data faster than the consumer can use it. The waiting data is called **buffered data**. If we keep adding more, the program may use too much memory.

### 2. Is backpressure a situation or a mechanism?

Both words are used when people talk about streams, but they mean different things. To avoid confusion, learn them in this order:

| Term | Simple meaning |
| --- | --- |
| Speed mismatch | Producer is fast, consumer is slow |
| Buffer buildup | Extra data waits in memory |
| Backpressure signal | Consumer tells the producer: slow down |
| Backpressure handling | Producer pauses, waits, and then continues |

**Backpressure is the pressure or flow-control signal sent from a slower consumer toward a faster producer.** **Backpressure handling** is the mechanism that reacts to that signal. People also use “backpressure” to describe the overall flow-control mechanism.

So **backpressure does not mean an actual overflow has already happened**. Its purpose is to control flow *before* memory keeps growing.

**Remember:**

```text
PROBLEM:     Reader is fast, writer is slow
RISK:        Extra chunks fill the buffer
SIGNAL:      writable.write(chunk) returns false
ACTION:      Pause producing more chunks
READY AGAIN: "drain" event
ACTION:      Resume producing chunks
```

### 3. First see the problem in code (ignores backpressure)

Suppose `large.txt` contains lots of data:

```js
import fs from "node:fs";

const readable = fs.createReadStream("large.txt");
const writable = fs.createWriteStream("output.txt");

readable.on("data", (chunk) => {
  writable.write(chunk); // ignores the return value
});

readable.on("end", () => writable.end());
```

**What can go wrong?** Reading may be faster than writing. Calling `write()` again and again, without checking its result, can fill the writable buffer and increase memory use. This example is only to show the problem, not recommended production code.

### 4. Handle backpressure yourself — pause and resume

```js
import fs from "node:fs";

const readable = fs.createReadStream("large.txt");
const writable = fs.createWriteStream("output.txt");

readable.on("data", (chunk) => {
  const canContinue = writable.write(chunk);

  if (!canContinue) {
    console.log("Writer is busy. Pause reading.");
    readable.pause();
  }
});

writable.on("drain", () => {
  console.log("Writer is ready. Resume reading.");
  readable.resume();
});

readable.on("end", () => writable.end());
readable.on("error", (error) => writable.destroy(error));
writable.on("error", (error) => readable.destroy(error));
```

Here is the meaning of each important part:

1. `writable.write(chunk)` tries to write a chunk and returns a Boolean.
2. `true` means you may continue sending data.
3. `false` means **stop sending more chunks for now**. It does not mean the chunk was rejected: Node.js accepted it into its buffer.
4. `readable.pause()` stops the normal flow of `data` events.
5. `drain` means the writable has made enough room for more writes.
6. `readable.resume()` starts the data flow again.

In real applications, prefer `pipeline()` for complete stream jobs because it also helps manage errors and cleanup.

### 5. Easier solution: pipe() and pipeline() do this automatically

With `pipe()`:

```js
import fs from "node:fs";

fs.createReadStream("large.txt")
  .pipe(fs.createWriteStream("output.txt"));
```

With `pipeline()`:

```js
import fs from "node:fs";
import { pipeline } from "node:stream/promises";

try {
  await pipeline(
    fs.createReadStream("large.txt"),
    fs.createWriteStream("output.txt")
  );
  console.log("Copy complete");
} catch (error) {
  console.error("Copy failed:", error.message);
}
```

Both move data in chunks and handle normal backpressure for you. `pipeline()` also gives coordinated error handling and stream cleanup.

### 6. One-line answer for your notes

> Backpressure is a flow-control signal or concept: when the consumer cannot keep up with the producer, it signals the producer to slow down or pause. Backpressure handling uses that signal to control data flow and prevent too much buffering.

---

## Why backpressure is one of the most important stream concepts

Backpressure prevents a fast producer from overwhelming a slow consumer.

Without it, memory usage can grow uncontrollably.

This is a major production and interview topic.

---

## 1. The problem

Imagine:

```text
Producer
  100 MB/s
     |
     v
Consumer
  10 MB/s
```

The producer generates data 10x faster than the consumer can handle.

Where does the extra data go?

Into memory buffers.

If this continues:

```text
buffer
buffer
buffer
buffer
buffer
...
```

Memory grows.

---

## 2. What is backpressure?

Backpressure is the mechanism that tells the producer:

> Slow down. The consumer cannot accept more data yet.

Mental model:

```text
Producer
   |
   v
Writable buffer full
   |
   v
STOP
   |
   v
consumer drains
   |
   v
RESUME
```

---

## 3. Writable write() return value

```js
const ok =
  writable.write(chunk);
```

If:

```text
true
```

continue writing.

If:

```text
false
```

stop and wait for:

```text
drain
```

---

## 4. Manual backpressure handling

```js
import {
  once,
} from "node:events";

for (
  const chunk of chunks
) {
  const canWrite =
    writable.write(chunk);

  if (!canWrite) {
    await once(
      writable,
      "drain"
    );
  }
}

writable.end();
```

---

## 5. pipe handles backpressure

```js
readable.pipe(writable);
```

Internally, pipe coordinates pause/resume behavior based on writable pressure.

Conceptually:

```text
Readable
   |
   v
Writable write() false
   |
   v
Readable.pause()
   |
   v
drain
   |
   v
Readable.resume()
```

---

## 6. pipeline handles it too

`pipeline()` composes streams while preserving normal backpressure flow.

That is one reason stream composition is safer than manually listening to `data` and calling `write()` without checking return values.

---

## 7. highWaterMark

Streams use an internal buffering threshold called:

```text
highWaterMark
```

When buffered data reaches the threshold, writable `write()` may return false.

Important:

> highWaterMark is a threshold, not a strict maximum memory cap.

---

## 8. Why highWaterMark is not a hard cap

Even if threshold is:

```text
64 KB
```

the application may still temporarily use more memory due to:
- chunk sizes
- multiple streams
- object allocations
- queued callbacks
- native buffers

Do not interpret it as a total process memory limit.

---

## 9. Object mode

In object mode, `highWaterMark` is generally based on object count rather than byte count.

Conceptually:

```text
16 objects buffered
```

instead of:

```text
16 KB
```

This is important for ETL/data pipelines.

---

## 10. Real example — file to slow destination

```text
SSD file read
   |
   | fast
   v
stream
   |
   v
slow network upload
```

Without backpressure:
- memory fills

With backpressure:
- file reading slows automatically

---

## 11. HTTP backpressure

HTTP responses are Writable streams.

If the client has a slow connection:

```text
Server
  |
  | produces fast
  v
socket buffer
  |
  v
slow client
```

Backpressure helps prevent the server from buffering unlimited response data.

---

## 12. Database export example

```text
DB cursor
   |
   v
CSV Transform
   |
   v
HTTP Response
```

The HTTP client's speed should influence how quickly upstream records are consumed.

A proper stream pipeline naturally propagates pressure backward.

---

## 13. What happens if you ignore backpressure?

Possible consequences:

- memory spikes
- garbage collection overhead
- process crashes
- unstable latency
- poor throughput
- OOM termination

---

## 14. Backpressure is not only a Node.js concept

It appears in many systems:

- message queues
- reactive streams
- TCP
- data pipelines
- distributed systems

General principle:

> A producer must respect downstream capacity.

---

## 15. Common mistakes

### Mistake 1
Ignoring `write()` return value.

### Mistake 2
Assuming `data` event consumption automatically handles all pressure.

### Mistake 3
Setting huge `highWaterMark` values to "make it faster."

This may just increase memory.

### Mistake 4
Using Promise.all on huge streaming workloads instead of controlling flow.

---

## 16. Interview questions

### What is backpressure?

A flow-control mechanism that prevents a fast producer from overwhelming a slower consumer.

### How does a Writable signal backpressure?

`write()` returns false.

### When should the producer resume?

After the `drain` event.

### What is highWaterMark?

An internal buffering threshold.

### Does pipe handle backpressure?

Yes.

---

## 17. Strong interview answer

> Backpressure is flow control between a producer and consumer. If a Writable stream's internal buffer reaches its highWaterMark, write() returns false, signaling the producer to pause. Once enough buffered data is flushed, the writable emits drain and the producer can resume. pipe() and pipeline() manage this automatically, which prevents unbounded memory growth when consumers are slower than producers.

---

## Interview-Ready Summary

```text
Fast producer
     |
     v
buffer fills
     |
write() -> false
     |
     v
pause
     |
     v
drain
     |
     v
resume

Core idea:
respect downstream capacity
```

## Practice Task

Create a fast Readable and intentionally slow Writable.

Observe:
- `write()` returning false
- `drain` events
- memory behavior with and without backpressure handling
