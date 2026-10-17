# Lesson 201 — WebSockets in Node.js

## 1. Why WebSockets?
HTTP usually follows **request → response**. In a real-time chat application, the server must deliver a new message to connected users without waiting for them to ask again. A **WebSocket** provides a long-lived, two-way communication channel.

```text
Browser -- HTTP Upgrade handshake --> Node.js server
Browser <======== open WebSocket ========> Server
      messages can move in both directions
```

## 2. HTTP polling vs SSE vs WebSocket

| Technique | Direction | Use case |
| --- | --- | --- |
| Polling | Client repeatedly requests | Simple occasional status checks |
| SSE | Server → client | Live news/notifications |
| WebSocket | Both directions | Chat, multiplayer games, collaboration |

WebSocket begins with an HTTP-based handshake (usually an Upgrade request for HTTP/1.1) and then uses framed messages over the open connection. The connection isn't a normal repeated HTTP request.

## 3. Runnable example using ws
Install: `npm install ws`. Use `"type":"module"`.

**server.js**
```js
import { WebSocketServer, WebSocket } from "ws";
const wss = new WebSocketServer({ port: 8080 });

wss.on("connection", (socket, request) => {
  console.log("Client connected");
  socket.send(JSON.stringify({ type: "welcome" }));

  socket.on("message", (data) => {
    let message;
    try { message = JSON.parse(data.toString()); }
    catch { socket.close(1003, "Invalid JSON"); return; }

    if (message.type !== "chat" ||
        typeof message.text !== "string" ||
        message.text.length > 1000) {
      socket.send(JSON.stringify({ type: "error", message: "Invalid message" }));
      return;
    }

    for (const client of wss.clients) {
      if (client.readyState === WebSocket.OPEN) {
        client.send(JSON.stringify({ type: "chat", text: message.text }));
      }
    }
  });
  socket.on("error", (error) => console.error("Socket error:", error.message));
});
```

**Client (browser console on a suitable test page):**
```js
const ws = new WebSocket("ws://localhost:8080");
ws.onmessage = (event) => console.log(event.data);
ws.onopen = () => ws.send(JSON.stringify({ type: "chat", text: "Hello!" }));
```

This is a **local learning broadcast demo**, not a secure public chat server.

## 4. How messages travel
Browser establishes connection → server sends welcome → browser sends chat JSON → server parses/validates → broadcasts to currently connected clients. Offline users won't receive these broadcasts unless messages are saved to a database and fetched on reconnect.

## 5. WebSocket lifecycle
Browser states: CONNECTING, OPEN, CLOSING, CLOSED. Handle `open`, `message`, `error`, and `close`. Network failure can leave dead connections; heartbeat ping/pong and idle timeouts help detect them.

## 6. Authentication and authorization
Authenticate the handshake using your session/auth architecture and validate authorization **for each resource/action**. Don't accept arbitrary `userId` or room names as proof of identity. Browser WebSocket API can't set arbitrary Authorization headers; cookies or carefully designed short-lived connection credentials are common approaches. Check allowed `Origin` during handshake. **CORS middleware alone does not secure raw WebSocket upgrades.**

## 7. Security and production
- Require `wss://` on public HTTPS sites.
- Limit inbound payload size, clients, message rate and expensive handlers.
- Avoid unbounded broadcast and backpressure; check `bufferedAmount` / stream pressure.
- Handle close/error/heartbeat and graceful shutdown.
- Persist durable messages separately, with access checks.
- Consider a reverse proxy with correct Upgrade configuration.

## 8. WebSocket vs Socket.IO
`ws` implements the WebSocket protocol with a small API. Socket.IO offers a higher-level event model, rooms, reconnection and additional transports; **Socket.IO is not wire-compatible with a plain WebSocket client**.

## Practice
Create two browser clients and send messages. Add server-side maximum length checks. Then explain why reconnecting alone cannot recover messages sent while a user was offline.

**Interview answer:** WebSockets provide persistent bidirectional communication, ideal for chat and collaboration. Build authentication, per-message authorization, heartbeat, bounded payloads, persistence, reconnection strategy, and horizontal scaling around the socket transport.
