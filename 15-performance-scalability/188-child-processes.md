# Lesson 188 — Child Processes in Node.js

## 1. What is a child process?
A **child process** is a separate OS process started by Node.js. It has its own memory and can run another Node program or a system executable.

```text
Node API process
     |
     +--> child process: Node.js script
     |
     +--> child process: ffmpeg / other executable
```

Use this to run a separate program, isolate failures or offload work to an external executable.

## 2. Four important APIs
| API | Behavior | Typical use |
| --- | --- | --- |
| `spawn()` | Starts process; stdout/stderr are streams | Long-running command / large output |
| `exec()` | Runs a shell command; buffers output | Short trusted shell commands |
| `execFile()` | Runs a specific executable without a shell by default | Safer execution of fixed commands |
| `fork()` | Starts a Node.js module with IPC channel | Separate Node worker process |

**Security:** Never concatenate request input into an `exec()` command. Shell metacharacters can cause command injection. Prefer fixed executables with argument arrays, strict allowlists, and OS permissions.

## 3. Practical spawn example
```js
import { spawn } from "node:child_process";

const child = spawn(process.execPath, ["-v"], {
  shell: false,
});

child.stdout.on("data", (chunk) => process.stdout.write(chunk));
child.stderr.on("data", (chunk) => process.stderr.write(chunk));
child.on("error", (err) => console.error("Could not start:", err));
child.on("close", (code) => console.log("Exited with:", code));
```
`process.execPath` points to the Node binary used to run the parent. `-v` prints its version. The main JS event loop can continue while the child runs.

## 4. fork() and IPC example
**child.js**
```js
process.on("message", ({ number }) => {
  process.send?.({ squared: number * number });
});
```
**parent.js**
```js
import { fork } from "node:child_process";

const child = fork(new URL("./child.js", import.meta.url));
child.once("message", (message) => {
  console.log(message); // { squared: 49 }
  child.disconnect();
});
child.once("error", console.error);
child.send({ number: 7 });
```
`fork()` here creates **another Node process**, not a Worker Thread. Messages cross a process boundary; they do not share a JS heap.

## 5. Why not exec for large output?
`exec()` buffers command output in memory and has a `maxBuffer` limit. For streaming output or long-running processes, `spawn()` is often a better fit.

## 6. Secure command execution
Unsafe:
```js
exec(`convert ${req.body.filename} output.png`); // DON'T DO THIS
```
Safer pattern:
```js
spawn("/usr/bin/some-tool", ["--input", trustedFilePath], { shell: false });
```
Even `spawn` arguments can be risky if the external tool interprets user values as flags, paths or scripts. Validate/allowlist inputs; use safe `--` boundaries where supported, restrict directories, permissions, resource usage and timeouts.

## 7. Worker Thread vs child process
| Worker Thread | Child process |
| --- | --- |
| Same OS process | Separate process |
| Separate V8 isolate | Separate runtime/address space |
| Can transfer/shared memory explicitly | Usually IPC / OS streams |
| Good for JS CPU work | Good for external tools/isolation |
| Worker failure may impact shared process | Stronger process isolation, not a security sandbox |

Neither automatically replaces a durable job queue or distributed worker architecture.

## 8. Handling lifecycle
Handle `error` and `close`, kill timed-out work, cap concurrent subprocesses, drain streams and clean temporary files. Track child PIDs and shut them down gracefully with the parent.

## Practice
Use `spawn(process.execPath, ["-v"])`, then build a `fork()` IPC example. Explain why `exec(userInput)` is unsafe.

**Interview answer:** Child-process APIs run separate OS processes. Use spawn for streaming, execFile for a known executable, fork for Node-to-Node IPC, and avoid shell injection and uncontrolled process creation.
