# 17. Performance — NestJS

Performance মানে শুধু **"API কত দ্রুত response দেয়"** না।

একটা production backend-এর performance বলতে বোঝায়:

```text
Request
   ↓
NestJS
   ↓
Database / Cache / External Service
   ↓
Processing
   ↓
Response
```

এই পুরো flow-টা যেন:

* দ্রুত হয়
* কম resource ব্যবহার করে
* অনেক concurrent request handle করতে পারে
* unnecessary কাজ না করে

এখন আপনার দেওয়া প্রতিটা বিষয় একে একে দেখি।

---

# 1. Database Indexes

ধরুন database-এ 10 million users আছে।

আপনি query করছেন:

```ts
const user = await prisma.user.findUnique({
  where: {
    email: 'aziz@example.com',
  },
});
```

যদি `email`-এ index না থাকে, database-কে অনেক বেশি row search করতে হতে পারে।

```text
Users
│
├── user 1
├── user 2
├── user 3
├── ...
├── user 10,000,000
```

Database-কে potentially অনেক data scan করতে হতে পারে।

---

## Index কী করে?

Index হলো database-এর একটা বিশেষ data structure যা search দ্রুত করতে সাহায্য করে।

Conceptually:

```text
Database

10 million users
       │
       ▼
   email index
       │
       ▼
aziz@example.com
       │
       ▼
     User
```

যেমন বইয়ের শেষে:

```text
Index
A → Page 10
B → Page 50
C → Page 100
```

তাই পুরো বই পড়তে হয় না।

Database index-ও একই ধরনের ধারণা দেয়।

---

## Example

Prisma:

```prisma
model User {
  id    String @id
  email String @unique
}
```

`@unique` সাধারণত database-এ uniqueness enforce করার পাশাপাশি একটি unique index তৈরি করে।

আর query:

```ts
await prisma.user.findUnique({
  where: {
    email,
  },
});
```

দ্রুত lookup করতে পারে।

---

## Composite Index

ধরুন আপনার query:

```ts
await prisma.product.findMany({
  where: {
    farmerId,
    status: 'ACTIVE',
  },
});
```

তখন এই ধরনের index useful হতে পারে:

```prisma
model Product {
  id       String @id
  farmerId String
  status   String

  @@index([farmerId, status])
}
```

মানে:

```text
farmerId + status
        ↓
      index
```

---

## কিন্তু সব field-এ index দেবেন?

না।

Index-এরও cost আছে:

```text
Index
 ↓
Faster reads
```

কিন্তু:

```text
More indexes
 ↓
More storage
 ↓
Insert/update-এর overhead
```

তাই যেসব field দিয়ে frequently search/filter/sort করা হয়, সেগুলোর জন্য appropriate index design করতে হয়।

---

# 2. Query Optimization

একটা query কাজ করছে মানেই query ভালো—এটা ধরে নেওয়া যাবে না।

ধরুন:

```ts
const products = await prisma.product.findMany();
```

Database-এ 500,000 product আছে।

আপনার UI-তে দরকার মাত্র 20টা।

কিন্তু আপনি:

```text
500,000 rows
      ↓
Database
      ↓
NestJS
      ↓
500,000 rows
```

নিয়ে আসছেন।

এটা unnecessary।

---

## ভালো query

```ts
const products = await prisma.product.findMany({
  select: {
    id: true,
    title: true,
    price: true,
    image: true,
  },

  take: 20,
});
```

এখন database থেকে শুধু দরকারি data নিচ্ছেন।

---

## Query optimization-এর মূল idea

নিজেকে প্রশ্ন করবেন:

```text
আমি কি প্রয়োজনের চেয়ে বেশি data আনছি?
```

যেমন:

```text
❌ SELECT *
```

এর পরিবর্তে প্রয়োজনীয় column:

```text
✅ SELECT id, title, price
```

এবং:

```text
❌ সব 500,000 rows
```

এর পরিবর্তে:

```text
✅ 20 rows
```

---

# 3. N+1 Problem

Performance-এর খুব common database problem।

ধরুন:

```text
Get 100 products
```

প্রথম query:

```sql
SELECT * FROM products;
```

তারপর প্রতিটা product-এর farmer বের করতে:

```text
Product 1 → farmer query
Product 2 → farmer query
Product 3 → farmer query
...
Product 100 → farmer query
```

তাহলে:

```text
1 + 100 = 101 queries
```

এটাই **N+1 problem**।

---

## Better approach

Relation eager/include করে প্রয়োজনীয় data একসাথে আনা যায়।

Prisma:

```ts
const products = await prisma.product.findMany({
  include: {
    farmer: true,
  },
});
```

তখন ORM database query strategy অনুযায়ী relation data efficiently fetch করার চেষ্টা করবে।

মূল ধারণা:

```text
❌ 101 queries

      বনাম

✅ প্রয়োজন অনুযায়ী কম query
```

---

# 4. Caching

ধরুন:

```http
GET /products/categories
```

এই data খুব কম পরিবর্তন হয়।

কিন্তু প্রতিবার:

```text
Client
 ↓
NestJS
 ↓
Database
 ↓
Categories
```

করলে database-এর উপর unnecessary load পড়ে।

Cache ব্যবহার করলে:

```text
Client
 ↓
NestJS
 ↓
Redis
 ↓
Categories
```

---

## Cache flow

প্রথম request:

```text
Request
  ↓
Redis
  ↓
Not found
  ↓
Database
  ↓
Save to Redis
  ↓
Response
```

পরের request:

```text
Request
  ↓
Redis
  ↓
Found
  ↓
Response
```

Database-এ যাওয়া লাগল না।

---

## Cache Example

Conceptually:

```ts
const cached = await redis.get('products');

if (cached) {
  return JSON.parse(cached);
}

const products = await this.productService.findAll();

await redis.set(
  'products',
  JSON.stringify(products),
  'EX',
  60,
);

return products;
```

এখানে:

```text
EX 60
```

মানে cache 60 seconds-এর জন্য থাকবে।

---

# 5. Cache Invalidation

Caching-এর famous problem:

> **"There are only two hard things in Computer Science: cache invalidation and naming things."**

ধরুন:

```text
Redis:
Product price = 100
```

Database:

```text
Product price = 120
```

আপনি database update করলেন কিন্তু cache update/remove করলেন না।

তখন user পাবে:

```text
100
```

যদিও actual price:

```text
120
```

তাই update-এর সময় cache invalidate করতে হয়।

```ts
await prisma.product.update(...);

await redis.del(
  `product:${productId}`,
);
```

---

# 6. Pagination

ধরুন:

```http
GET /products
```

Database-এ:

```text
1,000,000 products
```

সব একবারে পাঠানো যাবে না।

Pagination:

```text
Page 1 → 20 products
Page 2 → 20 products
Page 3 → 20 products
```

---

## Offset Pagination

Request:

```http
GET /products?page=2&limit=20
```

Calculation:

```text
skip = (page - 1) * limit

skip = (2 - 1) * 20
     = 20
```

Prisma:

```ts
const page = 2;
const limit = 20;

const products = await prisma.product.findMany({
  skip: (page - 1) * limit,
  take: limit,
});
```

---

# 7. Cursor Pagination

Large dataset-এ cursor pagination অনেক ক্ষেত্রে useful।

ধরুন:

```text
GET /products?cursor=product_100
```

মানে:

> product_100-এর পরের products দিন।

Concept:

```text
Product 1
Product 2
Product 3
...
Product 100 ← cursor
Product 101
Product 102
...
```

Cursor-based pagination বড় dataset এবং infinite scrolling-এর মতো use case-এ ভালো fit হতে পারে।

---

# 8. Lazy Loading বনাম Eager Loading

এটা database relation-এর ক্ষেত্রে গুরুত্বপূর্ণ।

ধরুন:

```text
User
 ↓
Orders
 ↓
Products
```

### Eager Loading

শুরুতেই related data load করা:

```text
User
 ├── Orders
 │    └── Products
```

Prisma:

```ts
const user = await prisma.user.findUnique({
  where: {
    id: userId,
  },

  include: {
    orders: true,
  },
});
```

---

### Lazy Loading

প্রথমে শুধু user:

```text
User
```

যখন orders দরকার:

```text
User
 ↓
Load Orders
```

তারপর products দরকার হলে:

```text
Orders
 ↓
Load Products
```

---

## কোনটা better?

একটা সবসময় better না।

যদি related data দরকারই না হয়:

```text
Lazy
```

useful হতে পারে।

যদি response-এর জন্য relation অবশ্যই দরকার হয়:

```text
Eager/include
```

appropriate হতে পারে।

মূল লক্ষ্য:

> **যে data দরকার নেই সেটা unnecessarily load করবেন না।**

---

# 9. Connection Pooling

এটা database performance-এর জন্য খুব গুরুত্বপূর্ণ।

ধরুন প্রতি request-এ নতুন database connection তৈরি করছেন:

```text
Request 1 → DB connection
Request 2 → DB connection
Request 3 → DB connection
...
```

হাজার request হলে সমস্যা।

Connection তৈরি করা এবং maintain করা expensive হতে পারে।

---

## Connection Pool

Pool আগে থেকেই কিছু connection রাখে:

```text
Database Connection Pool

┌──────┐
│ Conn1│
├──────┤
│ Conn2│
├──────┤
│ Conn3│
├──────┤
│ Conn4│
└──────┘
```

Request এলো:

```text
Request
   ↓
Take available connection
   ↓
Query
   ↓
Return connection to pool
```

অর্থাৎ প্রতিবার নতুন connection create করতে হচ্ছে না।

---

## NestJS + Prisma

Prisma নিজেই connection management/pooling-এর mechanisms ব্যবহার করে, deployment ও datasource configuration-এর উপর নির্ভর করে।

আপনার কাজ হলো:

```text
এক request-এর জন্য
প্রতিবার নতুন PrismaClient তৈরি না করা।
```

ভুল pattern:

```ts
async function getUsers() {
  const prisma = new PrismaClient();

  return prisma.user.findMany();
}
```

এটা বারবার করলে connection/resource সমস্যা হতে পারে।

সাধারণত NestJS application lifecycle-এর সাথে একটি Prisma service/client manage করা হয়।

---

# 10. Compression

ধরুন আপনার API response:

```text
2 MB JSON
```

Client-এর কাছে পাঠাতে হবে।

Compression করলে:

```text
2 MB
 ↓
Gzip/Brotli
 ↓
300 KB
```

Network bandwidth কম লাগে।

NestJS/Express application-এ compression middleware ব্যবহার করা যায়।

```bash
npm install compression
```

তারপর:

```ts
import compression from 'compression';

app.use(compression());
```

এতে compressible response-এর network payload ছোট হতে পারে।

---

# 11. Response Optimization

ধরুন database:

```json
{
  "id": "123",
  "name": "Aziz",
  "email": "aziz@example.com",
  "passwordHash": "...",
  "internalNotes": "...",
  "createdAt": "...",
  "updatedAt": "..."
}
```

Frontend-এর দরকার:

```json
{
  "id": "123",
  "name": "Aziz"
}
```

তাহলে সব data পাঠানো উচিত না।

---

## DTO দিয়ে response shape control

```ts
export class UserResponseDto {
  id: string;
  name: string;
}
```

Response:

```json
{
  "id": "123",
  "name": "Aziz"
}
```

এতে:

```text
Payload ↓
Network usage ↓
Serialization work ↓
Unnecessary data exposure ↓
```

---

# 12. Database থেকে প্রয়োজনীয় field নিন

Response optimize করার আগে database query-ও optimize করুন।

Prisma:

```ts
const user = await prisma.user.findUnique({
  where: {
    id: userId,
  },

  select: {
    id: true,
    name: true,
    email: true,
  },
});
```

এখন unnecessary columns database থেকে আনছেন না।

---

# 13. Node.js Event Loop

এটা NestJS performance-এর core concept।

Node.js সাধারণত JavaScript code execute করে একটি main thread-এর event loop-এর মাধ্যমে।

সহজভাবে:

```text
                 ┌─────────────┐
Request ────────>│ Event Loop  │
                 └──────┬──────┘
                        │
             ┌──────────┼──────────┐
             ▼          ▼          ▼
           Task       Task       Task
```

Node.js-এর শক্তি হলো:

> I/O operation-এর জন্য main JavaScript thread-কে block না করে asynchronous কাজ handle করা।

---

# 14. I/O-bound কাজ

যেমন:

```text
Database query
File read
HTTP request
Redis request
```

ধরুন:

```ts
const user = await prisma.user.findUnique({
  where: {
    id,
  },
});
```

Database response-এর জন্য অপেক্ষা করার সময় Node অন্য কাজ handle করতে পারে।

Conceptually:

```text
Request A
   ↓
DB query ────────────────┐
                         │
Request B                │
   ↓                     │
Process B                │
                         │
Request C                │
   ↓                     │
Process C                │
                         │
                  DB response
                         │
                         ▼
                    Continue A
```

এটাই Node.js-এর async I/O model-এর বড় advantage।

---

# 15. CPU-heavy Tasks

এখন ধরুন:

```ts
while (true) {
  // huge computation
}
```

অথবা বড় computation:

```text
10 billion calculations
```

যদি main JavaScript thread-এ করেন:

```text
Event Loop
   ↓
CPU-heavy task
   ↓
BLOCKED
```

তখন অন্য request-এর response delay হবে।

---

## Example

```ts
@Get('heavy')
heavyTask() {
  let result = 0;

  for (let i = 0; i < 10_000_000_000; i++) {
    result += i;
  }

  return result;
}
```

এই computation চলার সময় event loop আটকে যেতে পারে।

অন্য user:

```text
GET /users
```

করলেও response delay হতে পারে।

---

# 16. Event Loop Blocking

ধরুন একই server:

```text
User A → /heavy
User B → /users
User C → /products
```

`/heavy` CPU-heavy।

```text
User A
  ↓
CPU-heavy
  ↓
████████████████
Event Loop BLOCKED
████████████████
  ↓
User B waits
  ↓
User C waits
```

এটা production application-এ সমস্যা।

---

# 17. Worker Threads

CPU-heavy JavaScript computation-এর জন্য Node.js-এর **Worker Threads** ব্যবহার করা যায়।

Concept:

```text
Main Thread
     │
     ├── API requests
     ├── Event Loop
     │
     └──────> Worker Thread
                  │
                  ▼
             Heavy calculation
                  │
                  ▼
                Result
                  │
                  ▼
             Main Thread
```

Main event loop free থাকে।

---

# 18. Worker Thread Example

`worker.ts`:

```ts
import { parentPort } from 'node:worker_threads';

let result = 0;

for (let i = 0; i < 1_000_000_000; i++) {
  result += i;
}

parentPort?.postMessage(result);
```

Main code:

```ts
import {
  Worker,
} from 'node:worker_threads';

const worker = new Worker(
  './worker.js',
);

worker.on('message', (result) => {
  console.log('Result:', result);
});
```

এখানে heavy calculation worker thread-এ চলছে।

---

# 19. Worker Threads কখন?

যেমন:

```text
Image processing
Large calculation
Data transformation
CPU-heavy encryption/computation
Large file processing
```

যখন কাজটা CPU-bound এবং main event loop block করার সম্ভাবনা থাকে।

---

# 20. Worker Thread বনাম Async I/O

এই distinction খুব গুরুত্বপূর্ণ।

### Database query

```ts
await prisma.user.findMany();
```

এটা I/O-bound।

সাধারণত Worker Thread দরকার নেই।

---

### Heavy computation

```ts
for (let i = 0; i < 10_000_000_000; i++) {
  // computation
}
```

এটা CPU-bound।

Worker Thread বিবেচনা করা যায়।

---

# 21. Performance Optimization-এর পুরো Picture

একটা request:

```text
Client
  ↓
NestJS
  ↓
Controller
  ↓
Service
  ↓
Database
```

এখানে bottleneck যেকোনো জায়গায় হতে পারে।

---

## Database slow হলে

ব্যবহার করুন:

```text
Indexes
Query optimization
Pagination
Connection pooling
```

---

## Same data বারবার লাগলে

```text
Caching
```

---

## অনেক data return করলে

```text
Pagination
Response optimization
Compression
```

---

## Relation data বেশি load করলে

```text
Lazy / Eager loading
```

---

## CPU-heavy কাজ হলে

```text
Worker Threads
```

---

## Overall Node performance

```text
Don't block Event Loop
```

---

# 22. একটি Real Example

ধরুন আপনার:

```http
GET /products
```

API আছে।

Database:

```text
10 million products
```

খারাপ implementation:

```ts
const products = await prisma.product.findMany();
```

তারপর:

```text
10 million rows
     ↓
NestJS
     ↓
JSON serialization
     ↓
Compression
     ↓
Network
     ↓
Client
```

ভয়াবহ inefficient হতে পারে।

---

ভালো approach:

```text
GET /products?page=1&limit=20
              │
              ▼
          Pagination
              │
              ▼
        Indexed Query
              │
              ▼
       Select required fields
              │
              ▼
            Cache
              │
              ▼
         Small response
              │
              ▼
         Compression
              │
              ▼
           Client
```

---

# 23. Performance-এর Mental Model

এই flow-টা মনে রাখুন:

```text
                    PERFORMANCE
                         │
        ┌────────────────┼────────────────┐
        │                │                │
     Database         Backend          Network
        │                │                │
   ┌────┼────┐       ┌───┴────┐       ┌──┴────┐
   │    │    │       │        │       │       │
 Index Query Cache  Event   Workers Compression
       │             Loop
 Pagination
       │
 Connection Pool
```

আর আপনার roadmap-এর প্রতিটা বিষয় এক লাইনে:

```text
Database Index
→ Search দ্রুত করার structure

Query Optimization
→ Database-কে unnecessary কাজ না করানো

Caching
→ Frequently used data দ্রুত ফেরত দেওয়া

Pagination
→ একসাথে অল্প data আনা

Lazy/Eager Loading
→ Related data কখন load হবে তা control করা

Connection Pooling
→ Database connection reuse করা

Compression
→ Network payload ছোট করা

Response Optimization
→ শুধু প্রয়োজনীয় data পাঠানো

Event Loop
→ Node.js-এর async execution model

CPU-heavy Tasks
→ Main thread block করতে পারে এমন কাজ

Worker Threads
→ CPU-heavy কাজ আলাদা thread-এ চালানো
```

সবশেষে সবচেয়ে গুরুত্বপূর্ণ principle:

> **Performance optimization শুরু করবেন bottleneck কোথায় সেটা measure করে; আন্দাজ করে সব জায়গায় optimization করলে code unnecessarily complex হয়ে যেতে পারে।**
