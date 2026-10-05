# Lesson 66 — Processing Large Files Efficiently

## Why this lesson matters

Large-file processing is where Node.js streams become practically valuable.

Examples:

- CSV imports
- log processing
- database exports
- video delivery
- backups
- compression
- ETL pipelines

The main rule is:

> Do not load huge files entirely into memory unless you truly need to.

---

## 1. Bad approach

```js
import {
  readFile,
} from "node:fs/promises";

const data =
  await readFile(
    "huge.csv",
    "utf8"
  );
```

For a multi-GB file, this can create huge memory pressure.

---

## 2. Better approach

```js
import fs from "node:fs";

const stream =
  fs.createReadStream(
    "huge.csv"
  );
```

Now data arrives incrementally.

---

## 3. Memory comparison

Without streaming:

```text
5 GB file
   |
   v
attempt to hold ~5 GB
```

With streaming:

```text
5 GB file
   |
   v
small chunk
   |
   v
process
   |
   v
release / continue
```

---

## 4. Copying a large file

```js
import fs from "node:fs";
import {
  pipeline,
} from "node:stream/promises";

await pipeline(
  fs.createReadStream(
    "large.iso"
  ),
  fs.createWriteStream(
    "copy.iso"
  )
);
```

This is memory-efficient and backpressure-aware.

---

## 5. Compression

```js
import fs from "node:fs";
import {
  createGzip,
} from "node:zlib";
import {
  pipeline,
} from "node:stream/promises";

await pipeline(
  fs.createReadStream(
    "large.log"
  ),
  createGzip(),
  fs.createWriteStream(
    "large.log.gz"
  )
);
```

The entire file never needs to exist in memory at once.

---

## 6. Line-by-line processing

A common requirement:

```text
huge.log
  -> process one line at a time
```

Node.js provides tools such as readline for line-oriented streaming.

Example:

```js
import fs from "node:fs";
import readline from "node:readline";

const input =
  fs.createReadStream(
    "huge.log"
  );

const rl =
  readline.createInterface({
    input,
    crlfDelay: Infinity,
  });

for await (
  const line of rl
) {
  processLine(line);
}
```

This is ideal for log analysis.

---

## 7. CSV processing

Conceptual pipeline:

```text
CSV file
   |
   v
Read Stream
   |
   v
CSV Parser
   |
   v
Validation Transform
   |
   v
Batch Writer
   |
   v
Database
```

The key is never storing the full CSV in memory.

---

## 8. Batch database writes

Writing one DB row per line may be too slow.

Better:

```text
read rows
   |
   v
collect 500 records
   |
   v
bulk insert
   |
   v
next 500
```

This balances:
- memory
- DB throughput
- transaction overhead

---

## 9. Bounded concurrency

Suppose each file record requires an API call.

Bad:

```js
await Promise.all(
  allRows.map(callApi)
);
```

That defeats streaming and can overwhelm the API.

Better:

```text
stream records
   |
   v
process N concurrently
   |
   v
wait
   |
   v
continue
```

Use:
- worker pools
- queues
- bounded concurrency

---

## 10. Chunk boundaries

Never assume stream chunks equal:
- lines
- CSV rows
- JSON objects
- protocol messages

Chunk boundaries are transport-level implementation details.

Use a parser that preserves incomplete data between chunks.

---

## 11. NDJSON

Newline-delimited JSON is especially stream-friendly.

Example file:

```text
{"id":1}
{"id":2}
{"id":3}
```

Each line can be parsed independently.

This is often easier to stream than one giant JSON array.

---

## 12. Giant JSON problem

File:

```json
[
  {...},
  {...},
  {...}
]
```

A normal `JSON.parse()` generally requires the full string.

For very large JSON:
- use streaming JSON parsers
- use NDJSON
- redesign export format

---

## 13. HTTP file download

Bad:

```js
const file =
  await readFile(
    "movie.mp4"
  );

res.end(file);
```

Better:

```js
const file =
  fs.createReadStream(
    "movie.mp4"
  );

file.pipe(res);
```

Production systems should also support:
- range requests
- content type
- content length where appropriate
- cancellation
- errors

---

## 14. Client disconnects

If a client cancels a download, continuing to read/process the entire source wastes resources.

Production systems should observe:
- response close
- abort signal
- pipeline cancellation

Mental model:

```text
client disconnect
     |
     v
stop downstream
     |
     v
destroy upstream
```

---

## 15. Error handling

For multi-stream workflows, prefer:

```js
try {
  await pipeline(
    source,
    transform,
    destination
  );
} catch (error) {
  // log / cleanup
}
```

This is safer than loosely connected event listeners.

---

## 16. Progress tracking

Large jobs often need progress metrics.

Example:

```text
bytes processed
---------------- x 100
total file size
```

You can inspect:
- bytes read
- records processed
- batches completed

Do not add expensive logging per chunk in high-throughput pipelines.

---

## 17. Production architecture example

Large CSV import:

```text
Upload
  |
  v
Object Storage
  |
  v
Queue Job
  |
  v
Worker
  |
  v
Read Stream
  |
  v
CSV Parser
  |
  v
Validation
  |
  v
Batch DB Writes
  |
  v
Progress Update
```

This architecture is much safer than processing a multi-GB upload entirely inside one HTTP request.

---

## 18. Common mistakes

### Mistake 1
Using readFile for huge files.

### Mistake 2
Collecting all chunks into an array.

That recreates the same memory problem.

### Mistake 3
Unbounded concurrency per record.

### Mistake 4
Ignoring client cancellation.

### Mistake 5
Parsing huge JSON synchronously.

### Mistake 6
Writing one DB operation per record when batching would help.

---

## 19. Interview questions

### Why are streams useful for large files?

They process data incrementally instead of holding the full file in memory.

### How would you process a 10 GB CSV?

Use a Readable stream, streaming parser, validation transform, bounded/batched DB writes, and proper backpressure/error handling.

### Why not use Promise.all for every row?

Because it can create unbounded concurrency and memory/resource pressure.

### What format is stream-friendly for JSON records?

NDJSON is often a good choice.

---

## 20. Strong interview answer

> For large files, I avoid readFile because it loads the entire file into memory. I use a Readable stream, parse data incrementally, transform or validate records, and write downstream using pipeline so backpressure and errors are handled correctly. For database imports, I batch writes and bound concurrency. For very large jobs, I usually move processing to a background worker and store progress separately.

---

## Interview-Ready Summary

```text
Large File
   |
   v
Read Stream
   |
   v
Parser / Transform
   |
   v
Bounded Processing
   |
   v
Batch Write
   |
   v
Destination

Goals:
low memory
backpressure
bounded concurrency
safe errors
cancellation
```

## Section 6 Final Mental Model

```text
Events
   |
   +--> EventEmitter
   +--> custom events

Buffers
   |
   +--> raw bytes

Streams
   |
   +--> Readable
   +--> Writable
   +--> Duplex
   +--> Transform
   |
   +--> pipe
   +--> pipeline
   +--> backpressure
   +--> large-file processing
```

If you understand this section deeply, you can explain how Node.js efficiently moves large amounts of data without loading everything into memory.

## Practice Task

Build a production-style CSV import flow:

1. read a large CSV as a stream
2. parse records incrementally
3. validate each row
4. batch 500 records
5. simulate DB writes
6. track progress
7. stop cleanly if an error occurs
8. keep memory usage stable
