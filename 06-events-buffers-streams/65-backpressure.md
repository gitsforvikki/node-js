# Lesson 65 — Backpressure

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
