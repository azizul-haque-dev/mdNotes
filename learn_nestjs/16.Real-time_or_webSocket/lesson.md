# 18. WebSockets / Real-time — NestJS

এবার এমন একটা বিষয় শিখি যেখানে **client আর server-এর মধ্যে live connection** থাকে।

সাধারণ REST API-তে flow হয়:

```text
Client
   ↓
Request
   ↓
Server
   ↓
Response
```

যেমন:

```http
GET /notifications
```

Server response দিল:

```json
{
  "notifications": 5
}
```

এরপর নতুন notification এলো।

REST API নিজে থেকে client-কে কিছু পাঠাবে না। Client-কে আবার request করতে হবে:

```text
GET /notifications
```

কিন্তু WebSocket-এ:

```text
Client ←──────────────→ Server
        Live connection
```

Connection একবার establish হওয়ার পর **server নিজে থেকেই client-কে data পাঠাতে পারে।**

---

# 1. WebSocket কী?

সহজভাবে:

> **WebSocket হলো client এবং server-এর মধ্যে একটা persistent, two-way connection।**

মানে দুই দিক থেকেই যেকোনো সময় message যেতে পারে।

```text
             WebSocket
Client  ←────────────────→  Server
         দুই দিকেই data
```

REST:

```text
Client ──request──> Server
Client <──response── Server
```

WebSocket:

```text
Client ──────────────── Server
       <──────────────>
       <──────────────>
       <──────────────>
       connection alive
```

---

# 2. Real-time কেন দরকার?

ধরুন একটা chat application।

User A message পাঠালো:

```text
A → Hello
```

Server যদি WebSocket ব্যবহার করে:

```text
User A
  ↓
NestJS
  ↓
User B
```

User B সঙ্গে সঙ্গে message পেয়ে যাবে।

আবার:

```text
Admin dashboard
       ↑
       │
  live updates
       │
       ↑
     Server
```

এখানে বারবার API call করার দরকার নেই।

---

# 3. WebSocket কোথায় ব্যবহার হয়?

আপনার roadmap-এর examples:

### Chat

```text
User A → "Hello"
           ↓
         Server
           ↓
User B → "Hello"
```

### Notifications

```text
Order shipped
     ↓
NestJS
     ↓
User's browser
     ↓
🔔 New notification
```

### Live Order Status

```text
Order

PLACED
  ↓
CONFIRMED
  ↓
PREPARING
  ↓
SHIPPED
  ↓
DELIVERED
```

Status change হলেই client update পাবে।

### Admin Dashboard

```text
New Order
   ↓
Backend
   ↓
WebSocket
   ↓
Admin Dashboard
   ↓
Orders: 101 → 102
```

---

# 4. WebSocket বনাম REST

| REST API                    | WebSocket                        |
| --------------------------- | -------------------------------- |
| Request → Response          | Two-way communication            |
| সাধারণত short-lived request | Persistent connection            |
| Client request করে          | Server নিজেও message পাঠাতে পারে |
| CRUD-এর জন্য excellent      | Real-time-এর জন্য useful         |
| `/products`                 | `order.updated`                  |
| `/users/me`                 | `notification.new`               |

সহজভাবে:

```text
REST
"আমাকে data দাও"

WebSocket
"Connection রাখো, data change হলে আমাকে জানাও"
```

---

# 5. Socket.IO কী?

এখন আসি **Socket.IO**-তে।

Socket.IO হলো WebSocket-এর উপর তৈরি একটি real-time communication library।

এটা আপনাকে অনেক convenient feature দেয়:

```text
Socket.IO
 ├── Events
 ├── Rooms
 ├── Broadcasting
 ├── Reconnection
 └── Namespaces
```

NestJS-এর সাথে Socket.IO খুব সুন্দরভাবে কাজ করে।

---

# 6. WebSocket বনাম Socket.IO

এগুলোকে একই জিনিস ভাববেন না।

```text
WebSocket
   ↓
Communication protocol

Socket.IO
   ↓
Real-time communication library
```

সহজ analogy:

```text
WebSocket
= রাস্তা

Socket.IO
= সেই রাস্তার উপর বানানো feature-rich transport system
```

Socket.IO নিজের event-based abstraction দেয়।

যেমন:

```text
message
notification
order.updated
typing
```

---

# 7. NestJS Gateway

NestJS-এ WebSocket handle করার জন্য সবচেয়ে important concept হলো:

# Gateway

REST API-তে আমরা ব্যবহার করি:

```ts
@Controller()
```

WebSocket-এর জন্য:

```ts
@WebSocketGateway()
```

Example:

```ts
import {
  WebSocketGateway,
} from '@nestjs/websockets';

@WebSocketGateway()
export class ChatGateway {}
```

এটাই আপনার WebSocket gateway।

---

# 8. Gateway কী করে?

Gateway হলো এমন একটা NestJS class যেখানে আপনি WebSocket connection এবং events handle করবেন।

Conceptually:

```text
Client
   ↓
WebSocket
   ↓
ChatGateway
   ↓
Chat Service
   ↓
Database
```

---

# 9. Basic Gateway

```ts
import {
  WebSocketGateway,
  SubscribeMessage,
  MessageBody,
} from '@nestjs/websockets';

@WebSocketGateway()
export class ChatGateway {

  @SubscribeMessage('message')
  handleMessage(
    @MessageBody() message: string,
  ) {
    console.log(message);

    return {
      event: 'message',
      data: message,
    };
  }
}
```

এখানে:

```ts
@SubscribeMessage('message')
```

মানে:

> Client যদি `message` event পাঠায়, এই method execute করো।

---

# 10. Event কী?

WebSocket-এ data সাধারণত **event** দিয়ে communicate করা হয়।

যেমন:

```text
message
notification
typing
order.updated
user.online
```

Client:

```text
emit("message")
```

Server:

```text
listen("message")
```

---

# 11. Client → Server

ধরুন client:

```ts
socket.emit(
  'message',
  'Hello bro!',
);
```

Server:

```ts
@SubscribeMessage('message')
handleMessage(
  @MessageBody() message: string,
) {
  console.log(message);
}
```

Flow:

```text
Browser
   │
   │ message
   │ "Hello bro!"
   ▼
NestJS Gateway
   │
   ▼
handleMessage()
```

---

# 12. Server → Client

এখন server client-কে message পাঠাবে।

NestJS Gateway-তে সাধারণত Socket instance ব্যবহার করা হয়।

```ts
import {
  WebSocketGateway,
  WebSocketServer,
} from '@nestjs/websockets';

import { Server } from 'socket.io';

@WebSocketGateway()
export class ChatGateway {

  @WebSocketServer()
  server: Server;

  sendMessage() {
    this.server.emit(
      'message',
      {
        text: 'Hello everyone!',
      },
    );
  }
}
```

এখানে:

```ts
this.server.emit(...)
```

মানে connected clients-দের event পাঠানো।

---

# 13. Broadcasting

ধরুন:

```text
User A
User B
User C
User D
```

Server থেকে:

```ts
this.server.emit(
  'notification',
  {
    message: 'New order created',
  },
);
```

তাহলে সবাই পাবে:

```text
        Server
          │
    ┌─────┼─────┐
    ▼     ▼     ▼
   A      B      C
```

এটাকে broadcasting বলা যায়।

---

# 14. Rooms

এখন আসি **Rooms**-এ।

ধরুন আপনার chat application আছে।

```text
Room: "room-123"

User A
User B
User C
```

আপনি চান message শুধু এই room-এর user-রা পাবে।

তখন room ব্যবহার করবেন।

```text
Server
 │
 ├── room-123
 │     ├── A
 │     ├── B
 │     └── C
 │
 └── room-456
       ├── D
       └── E
```

---

# 15. User Room Join করবে

NestJS:

```ts
@SubscribeMessage('joinRoom')
handleJoinRoom(
  client: Socket,
  roomId: string,
) {
  client.join(roomId);
}
```

যদি:

```text
roomId = "room-123"
```

তাহলে:

```text
User A
   ↓
join("room-123")
```

এখন A room-এর member।

---

# 16. Room-এ Message পাঠানো

ধরুন:

```ts
this.server
  .to('room-123')
  .emit(
    'message',
    {
      text: 'Hello room!',
    },
  );
```

তাহলে:

```text
room-123

A ← message
B ← message
C ← message

D ← ❌
E ← ❌
```

কারণ D/E অন্য room-এ।

---

# 17. Chat Example

একটা basic chat gateway:

```ts
import {
  WebSocketGateway,
  WebSocketServer,
  SubscribeMessage,
  MessageBody,
} from '@nestjs/websockets';

import { Server, Socket } from 'socket.io';

@WebSocketGateway()
export class ChatGateway {

  @WebSocketServer()
  server: Server;

  @SubscribeMessage('joinRoom')
  joinRoom(
    client: Socket,
    @MessageBody() roomId: string,
  ) {
    client.join(roomId);
  }

  @SubscribeMessage('sendMessage')
  sendMessage(
    @MessageBody()
    data: {
      roomId: string;
      message: string;
    },
  ) {
    this.server
      .to(data.roomId)
      .emit('newMessage', {
        message: data.message,
      });
  }
}
```

Flow:

```text
User A
   ↓
joinRoom("room-123")
   ↓
Server
   ↓
Room membership

User A
   ↓
sendMessage
   ↓
room-123
   ↓
A + B + C
```

---

# 18. Authentication

এটা production WebSocket application-এর খুব গুরুত্বপূর্ণ অংশ।

REST API-তে আপনি করেন:

```http
Authorization: Bearer access_token
```

WebSocket-এ connection establish করার সময় authentication করতে হয়।

Concept:

```text
Client
   ↓
Connect
   ↓
Send token
   ↓
NestJS
   ↓
Verify JWT
   ↓
Authenticated socket
```

---

# 19. Socket.IO Authentication

Client:

```ts
const socket = io(
  'http://localhost:3000',
  {
    auth: {
      token: accessToken,
    },
  },
);
```

Server side middleware:

```ts
@WebSocketGateway()
export class ChatGateway {

  afterInit(server: Server) {
    server.use((socket, next) => {

      const token =
        socket.handshake.auth.token;

      if (!token) {
        return next(
          new Error('Unauthorized'),
        );
      }

      // Verify JWT here

      next();
    });
  }
}
```

Flow:

```text
Client
  │
  │ JWT
  ▼
Socket.IO
  │
  ▼
JWT verification
  │
  ├── Valid → connection allowed
  │
  └── Invalid → connection rejected
```

---

# 20. Authentication-এর পরে User কে রাখবেন?

JWT verify করার পরে user information socket-এর সাথে attach করা যায়।

Conceptually:

```ts
socket.data.user = user;
```

তারপর:

```ts
@SubscribeMessage('sendMessage')
handleMessage(client: Socket) {

  const user = client.data.user;

  console.log(user.id);
}
```

এখন server জানে:

```text
এই socket = কোন authenticated user
```

---

# 21. WebSocket Events

একটা real application-এ event naming important।

যেমন:

```text
message.sent
message.received

notification.created

order.created
order.updated
order.status_changed

user.online
user.offline
```

উদাহরণ:

```ts
this.server.emit(
  'order.status_changed',
  {
    orderId: '123',
    status: 'SHIPPED',
  },
);
```

Client শুনবে:

```ts
socket.on(
  'order.status_changed',
  (data) => {
    console.log(data);
  },
);
```

---

# 22. Live Order Status

ধরুন user-এর order:

```text
Order #1001
```

Initially:

```text
PENDING
```

Admin order confirm করল:

```text
CONFIRMED
```

Backend:

```ts
await this.orderService.updateStatus(
  orderId,
  'CONFIRMED',
);
```

তারপর:

```ts
this.server
  .to(`order:${orderId}`)
  .emit(
    'order.status_changed',
    {
      orderId,
      status: 'CONFIRMED',
    },
  );
```

User-এর browser সঙ্গে সঙ্গে পাবে:

```text
Order #1001
Status: CONFIRMED
```

Page refresh দরকার নেই।

---

# 23. Order-specific Room

এখানে room খুব useful।

User যখন order page open করবে:

```text
order:1001
```

room join করবে:

```ts
client.join(`order:${orderId}`);
```

তারপর:

```ts
this.server
  .to(`order:${orderId}`)
  .emit(
    'order.status_changed',
    data,
  );
```

শুধু ওই order-এর interested clients event পাবে।

---

# 24. Notifications

ধরুন User ID:

```text
user-123
```

তার personal room:

```text
user:user-123
```

User connection করলে:

```ts
client.join(
  `user:${userId}`,
);
```

তারপর server:

```ts
this.server
  .to(`user:${userId}`)
  .emit(
    'notification.new',
    {
      title: 'New Order',
      message: 'Your order has been confirmed.',
    },
  );
```

শুধু ওই user notification পাবে।

---

# 25. Chat Architecture

একটা production-style conceptual architecture:

```text
                 Client
                   │
             WebSocket
                   │
                   ▼
             Chat Gateway
                   │
          ┌────────┴────────┐
          ▼                 ▼
    Authentication      Chat Service
                              │
                              ▼
                           Database
```

Message flow:

```text
User A
  │
  │ sendMessage
  ▼
Gateway
  │
  ▼
Authenticate
  │
  ▼
Chat Service
  │
  ├── Save message
  │
  └── Broadcast
          │
          ▼
       Room
       /   \
      B     C
```

---

# 26. WebSocket + Database

একটা common ভুল হলো শুধু WebSocket-এ message পাঠিয়ে database-এ save না করা।

ধরুন:

```text
A → Hello
```

Server:

```ts
this.server.emit(
  'message',
  message,
);
```

User B message পেল।

কিন্তু database-এ save করলেন না।

তারপর B refresh করলে:

```text
Message disappeared
```

তাই chat-এর ক্ষেত্রে সাধারণ flow:

```text
Message
   ↓
Validate
   ↓
Save DB
   ↓
Emit WebSocket event
```

---

# 27. REST + WebSocket একসাথে

বাস্তব application-এ WebSocket REST-এর replacement না।

দুটো একসাথে থাকে।

যেমন:

### REST

```http
POST /messages
GET /messages
GET /orders/123
```

### WebSocket

```text
message.new
order.status_changed
notification.new
```

একটা useful architecture:

```text
REST
 ↓
CRUD / initial data

WebSocket
 ↓
Live updates
```

যেমন user chat page খুলল:

```text
GET /messages?roomId=123
```

পুরনো messages load করল।

তারপর:

```text
WebSocket
```

দিয়ে নতুন messages receive করবে।

---

# 28. Connection Lifecycle

WebSocket connection-এর lifecycle:

```text
Client
  │
  ▼
CONNECT
  │
  ▼
Authenticate
  │
  ▼
Connected
  │
  ├── Events
  ├── Join rooms
  ├── Receive messages
  └── Send messages
  │
  ▼
DISCONNECT
```

NestJS:

```ts
handleConnection(client: Socket) {
  console.log('Client connected');
}

handleDisconnect(client: Socket) {
  console.log('Client disconnected');
}
```

Gateway-এ এই lifecycle handlers ব্যবহার করা যায়।

---

# 29. Gateway-এ Service ব্যবহার

Gateway-এ সব business logic লিখবেন না।

খারাপ:

```ts
@WebSocketGateway()
export class ChatGateway {

  @SubscribeMessage('sendMessage')
  async sendMessage() {

    // validation
    // database
    // authorization
    // business logic
    // notification
    // everything here
  }
}
```

বরং:

```text
Gateway
   ↓
Service
   ↓
Repository / Database
```

Example:

```ts
@WebSocketGateway()
export class ChatGateway {

  constructor(
    private readonly chatService: ChatService,
  ) {}

  @SubscribeMessage('sendMessage')
  async sendMessage(
    @MessageBody() dto: SendMessageDto,
  ) {
    const message =
      await this.chatService.sendMessage(dto);

    // emit event

    return message;
  }
}
```

Gateway-এর কাজ মূলত:

```text
WebSocket communication
        ↓
Business service
```

---

# 30. WebSocket Error

REST-এ আমরা HTTP status code ব্যবহার করি:

```text
400
401
403
404
500
```

WebSocket-এ communication model আলাদা।

তাই event/error handling carefully design করতে হয়।

উদাহরণ:

```ts
throw new WsException(
  'You are not allowed to send messages',
);
```

NestJS WebSocket-এর জন্য:

```ts
WsException
```

ব্যবহার করা যায়।

---

# 31. সবচেয়ে গুরুত্বপূর্ণ Concepts

আপনার roadmap-এর প্রতিটা item:

```text
WebSocket
→ Persistent two-way connection

Socket.IO
→ WebSocket-based real-time library

Gateway
→ NestJS-এ WebSocket endpoint/event handler

Authentication
→ কে socket-এর সাথে connected তা verify করা

Rooms
→ নির্দিষ্ট group/client-দের কাছে event পাঠানো

Events
→ Real-time messages-এর named communication
```

একসাথে:

```text
                         NestJS
                           │
                     WebSocket Gateway
                           │
                 ┌─────────┼─────────┐
                 │         │         │
            Authentication Events   Rooms
                 │         │         │
                 ▼         ▼         ▼
                JWT    message    room:123
                         │
                         ▼
                      Service
                         │
                         ▼
                      Database
```

আর একটা real-time order system:

```text
                 Admin
                   │
              Update Order
                   │
                   ▼
              Order Service
                   │
                   ├── Update DB
                   │
                   ▼
             WebSocket Gateway
                   │
                   ▼
             order:123 room
                   │
             ┌─────┴─────┐
             ▼           ▼
           User A       User B
             │           │
             ▼           ▼
        Status Updated  Status Updated
```

**মূল idea:** REST-এ client সাধারণত request করে data নেয়; WebSocket-এ connection establish হওয়ার পরে server এবং client দুজনেই real-time event পাঠাতে পারে। NestJS-এ এই communication-এর কেন্দ্র হলো **Gateway + Events + Authentication + Rooms**।
