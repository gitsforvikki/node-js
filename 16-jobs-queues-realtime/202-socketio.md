# Lesson 202 — Socket.IO

## 1. What is Socket.IO?
**Socket.IO** is a Node/browser real-time communication library with named events, acknowledgments, automatic reconnection, broadcasting and rooms. It can use WebSocket or HTTP long-polling transport depending on configuration.

**Important:** Socket.IO is not the same protocol as raw WebSocket: a browser `new WebSocket()` client cannot connect directly to a Socket.IO server.

## 2. Real-world example — chat
When two developers exchange messages, the server should:
1. authenticate them;
2. verify they belong to the conversation;
3. persist messages in MongoDB;
4. notify online members;
5. let offline members fetch missed messages.

```text
React client -> emit chat:send
       -> Socket.IO server -> verify membership
                            -> DB save
                            -> room emit chat:new
```

## 3. Installation
```bash
npm install express socket.io
npm install socket.io-client
```

## 4. Express + Socket.IO backend
```js
import express from "express";
import { createServer } from "node:http";
import { Server } from "socket.io";

const app = express();
const server = createServer(app);

const io = new Server(server, {
  cors: {
    origin: "http://localhost:3000",
    credentials: true,
  },
});

io.use(async (socket, next) => {
  try {
    // Implement verified session/token lookup here.
    const user = await authenticateSocket(socket);
    if (!user) return next(new Error("Unauthorized"));
    socket.data.userId = user.id;
    next();
  } catch {
    next(new Error("Unauthorized"));
  }
});

io.on("connection", (socket) => {
  console.log("Connected:", socket.id);

  socket.on("chat:send", async (payload, ack) => {
    try {
      if (typeof payload?.conversationId !== "string" ||
          typeof payload?.text !== "string" ||
          payload.text.length < 1 || payload.text.length > 2000) {
        return ack?.({ ok: false, error: "Invalid message" });
      }

      const allowed = await isConversationMember(
        socket.data.userId, payload.conversationId
      );
      if (!allowed) return ack?.({ ok: false, error: "Forbidden" });

      const saved = await Message.create({
        conversationId: payload.conversationId,
        senderId: socket.data.userId,
        text: payload.text,
      });

      io.to(`conversation:${payload.conversationId}`).emit("chat:new", {
        id: String(saved.id), text: saved.text,
        conversationId: payload.conversationId,
      });
      ack?.({ ok: true, messageId: String(saved.id) });
    } catch (error) {
      console.error("chat send failed", error);
      ack?.({ ok: false, error: "Server error" });
    }
  });

  socket.on("disconnect", (reason) => console.log("Disconnected:", reason));
});

server.listen(4000);
```

This illustrates the **message handler**. `authenticateSocket`, `isConversationMember`, `Message` and database startup are application dependencies that must be implemented. Join logic is covered in Lesson 203.

## 5. Client example
```js
import { io } from "socket.io-client";

const socket = io("http://localhost:4000", {
  withCredentials: true,
});

socket.on("connect", () => console.log("Connected", socket.id));
socket.on("chat:new", (msg) => console.log("New message", msg));
socket.on("connect_error", (err) => console.error(err.message));

socket.emit("chat:send", {
  conversationId: "conv-1",
  text: "Hello!",
}, (result) => console.log("ACK:", result));
```

A Socket.IO acknowledgment means **your handler called the callback**, not that every other recipient read the message.

## 6. Socket IDs vs user IDs
`socket.id` identifies a **connection**, not a durable user. It can change after reconnection. Store the verified user ID in `socket.data` and your DB. A user can have several sockets across devices.

## 7. Reconnection and delivery
Socket.IO automatically attempts reconnection, but **default server events aren't a durable offline inbox**. Persist important events, use application sequence IDs/offsets, and fetch missed events after reconnect. Acknowledgments and retries require designed idempotency for message creation.

## 8. Security essentials
Check allowed browser origins and authenticated sessions. **Socket.IO CORS controls HTTP long-polling; WebSocket isn't governed by browser CORS**, so apply `allowRequest`/handshake checks as appropriate, not CORS alone. Authorize events and rooms separately. Limit message sizes/rates; do not trust `senderId` from payload.

## 9. When to use Socket.IO vs WebSocket
Use Socket.IO when its rooms, event naming, fallback transport, acknowledgments and reconnection are valuable. Use raw WebSocket for protocol interoperability and greater control with less abstraction.

## Practice
Add a `typing:start` event limited to authenticated conversation members. Explain why typing events need not be persisted, while chat messages should be.

**Interview answer:** Socket.IO layers events, rooms, reconnection and acknowledgments over real-time transports. Authenticate connections, authorize actions, persist important messages, and design replay/reconnection rather than expecting automatic durable delivery.
