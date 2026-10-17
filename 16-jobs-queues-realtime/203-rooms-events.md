# Lesson 203 — Socket.IO Rooms and Events

## 1. What are rooms?
A **room** is a server-side label grouping sockets. You can send messages only to clients in that group.

Example: CodeBuddy conversation `conv-5` has User A and User B. Their sockets join `conversation:conv-5`. A message to the room reaches those connected participants, not every user.

```text
Socket A --\
           -> room: conversation:conv-5 <- server emit
Socket B --/
Socket C -------- room: conversation:conv-9
```

Rooms are managed server-side; browser clients can't list all rooms by default. **Rooms are not a security boundary:** you must authorize joining and every privileged event.

## 2. Join and leave safely
```js
io.on("connection", (socket) => {
  socket.on("conversation:join", async (conversationId, ack) => {
    if (typeof conversationId !== "string") {
      return ack?.({ ok: false, error: "Invalid ID" });
    }

    const allowed = await isConversationMember(
      socket.data.userId, conversationId
    );
    if (!allowed) return ack?.({ ok: false, error: "Forbidden" });

    await socket.join(`conversation:${conversationId}`);
    ack?.({ ok: true });
  });

  socket.on("conversation:leave", async (conversationId) => {
    if (typeof conversationId !== "string") return;
    await socket.leave(`conversation:${conversationId}`);
  });
});
```

Assumes authentication middleware has set trusted `socket.data.userId`. Leaving doesn't require membership privileges, but event inputs still need validation and limits.

## 3. How to emit events
```js
socket.emit("welcome", { message: "Hello" }); // one socket

io.emit("announcement", { text: "Maintenance soon" }); // all sockets

io.to("conversation:conv-5").emit("chat:new", message); // one room

socket.to("conversation:conv-5").emit("typing", { userId }); // room except sender
```

Do not send private message content to `io.emit()` across all users.

## 4. Different event types
- **Persistent business events:** chat messages, order status changes → save in DB and replay after reconnect.
- **Ephemeral UI events:** typing, online presence, cursor location → often okay to drop when offline.
- **Acknowledgments:** callbacks indicate receiver/handler processed the event at an application-defined level.

## 5. Message event with safe flow
```text
Client emits chat:send
  -> server validates text + conversation
  -> verifies authenticated sender is a member
  -> stores in MongoDB
  -> emits chat:new to authorized room
  -> acknowledges saved message ID
```

**Never** trust a client-provided `senderId`. Use authenticated `socket.data.userId`. A user might join the correct room but later lose permission; consider rechecking membership on sensitive writes and revoking active room membership when access changes.

## 6. Multiple devices for one user
A user may have phone + laptop sockets. Put all authenticated sockets into `user:${userId}` rooms after successful authentication:
```js
socket.join(`user:${socket.data.userId}`);
```
Then `io.to(`user:${userId}`).emit("notification", data)` reaches current user sockets. Validate the user ID on server, not from the browser.

## 7. Rooms across several servers
By default, Socket.IO's in-memory adapter only knows **local process rooms**. In a multi-instance deployment, add a compatible distributed adapter (Lesson 204) so broadcasts reach connections on other servers.

## 8. Disconnection and cleanup
Socket.IO automatically removes a disconnected socket from its rooms. Update persistent presence using a robust TTL/heartbeat strategy; a single `disconnect` may not mean the user is offline if they have other sockets.

## 9. Common mistakes
Using room names as authentication; letting clients join any user room; broadcasting private data to everyone; treating socket ID as a permanent user ID; forgetting multi-server adapter; assuming events persist after disconnection.

## Practice
Build rooms for `conversation:A` and `conversation:B`, join three test clients, emit to A, and confirm B's client does not receive it. Add membership denial tests.

**Interview answer:** Rooms group connected Socket.IO sockets for targeted emits. Authenticate and authorize room membership, separate durable from ephemeral events, and use a shared adapter for multi-server broadcasts.
