# 9. Caching & Redis ⚡

Scalable NestJS application বানাতে **Caching & Redis** খুব গুরুত্বপূর্ণ।

সহজভাবে:

> **Cache হলো frequently-used data-এর temporary copy, যেটা দ্রুত জায়গায় রেখে পরের request-এ দ্রুত return করা হয়।**

আর:

> **Redis হলো খুব fast in-memory data store, যেটাকে cache ছাড়াও session, OTP, rate limiting, distributed locks এবং queue support-এর মতো কাজে ব্যবহার করা যায়।**

আপনার দেওয়া flow-টাই আগে বুঝি:

```text
Request
   ↓
Redis Cache
   │
   ├── Cache HIT → Response
   │
   └── Cache MISS
          ↓
       Database
          ↓
        Redis
          ↓
       Response
```

---

# 1. Cache কেন দরকার?

ধরুন:

```http
GET /products
```

প্রতিবার database-এ query করলে:

```text
User
 ↓
NestJS
 ↓
PostgreSQL
 ↓
Products
 ↓
NestJS
 ↓
User
```

ধরুন একই data 10,000 বার request হলো।

তাহলে:

```text
10,000 requests
      ↓
10,000 database queries
```

এতে database-এর উপর unnecessary load পড়ে।

Cache ব্যবহার করলে:

```text
First request
     ↓
Redis ❌
     ↓
PostgreSQL
     ↓
Redis ✅
     ↓
Response
```

পরের request:

```text
Request
   ↓
Redis ✅
   ↓
Response
```

Database-এ যেতে হলো না।

---

# 2. Cache Hit এবং Cache Miss

দুটি শব্দ খুব ভালোভাবে মনে রাখবেন।

## Cache Hit

Redis-এ data পাওয়া গেছে।

```text
Request
   ↓
Redis
   ↓
Data found ✅
   ↓
Response
```

এটাই:

```text
CACHE HIT
```

---

## Cache Miss

Redis-এ data নেই।

```text
Request
   ↓
Redis
   ↓
Data নেই ❌
   ↓
Database
   ↓
Redis-এ save
   ↓
Response
```

এটাই:

```text
CACHE MISS
```

---

# 3. Redis কী?

Redis হলো একটি **in-memory data store**।

`in-memory` মানে data মূলত RAM-এ রাখা হয়।

RAM খুব fast।

তাই:

```text
Redis
   ↓
RAM
   ↓
Very fast
```

Database যেমন PostgreSQL সাধারণত persistent data-এর জন্য:

```text
PostgreSQL
   ↓
Permanent application data
```

Redis সাধারণত খুব দ্রুত temporary বা short-lived data access-এর জন্য ব্যবহার করা হয়।

---

# 4. PostgreSQL বনাম Redis

সহজ comparison:

```text
PostgreSQL
├── Users
├── Products
├── Orders
├── Payments
└── Permanent business data


Redis
├── Cache
├── OTP
├── Sessions
├── Rate limits
├── Locks
└── Temporary data
```

একটা গুরুত্বপূর্ণ rule:

> Redis-এ cache রাখা মানে PostgreSQL-এর replacement বানানো না।

আপনার primary database PostgreSQL থাকতে পারে, আর Redis হবে fast supporting layer।

---

# 5. NestJS-এ Redis Setup

Redis server চালু থাকতে হবে।

তারপর Node.js থেকে Redis ব্যবহার করার জন্য popular package:

```bash
npm install ioredis
```

একটা Redis provider:

```ts
import { Injectable } from '@nestjs/common';
import Redis from 'ioredis';

@Injectable()
export class RedisService {
  private readonly redis = new Redis(
    process.env.REDIS_URL,
  );

  async get(key: string) {
    return this.redis.get(key);
  }

  async set(
    key: string,
    value: string,
  ) {
    return this.redis.set(key, value);
  }

  async delete(key: string) {
    return this.redis.del(key);
  }
}
```

Environment:

```env
REDIS_URL=redis://localhost:6379
```

---

# 6. Redis-এর Basic Operations

Redis-এর সবচেয়ে basic operations:

```text
SET
GET
DEL
EXPIRE
```

### SET

```ts
await redis.set(
  'user:123',
  'Aziz',
);
```

মানে:

```text
key:
user:123

value:
Aziz
```

---

### GET

```ts
const value = await redis.get(
  'user:123',
);
```

ফলাফল:

```text
Aziz
```

---

### DELETE

```ts
await redis.del(
  'user:123',
);
```

---

# 7. Object Cache করা

Redis সাধারণত string store করে।

তাই object হলে:

```ts
const user = {
  id: '123',
  name: 'Aziz',
  email: 'aziz@example.com',
};
```

JSON বানিয়ে store করতে পারেন:

```ts
await redis.set(
  'user:123',
  JSON.stringify(user),
);
```

পরে:

```ts
const cachedUser =
  await redis.get('user:123');
```

তারপর:

```ts
const user = cachedUser
  ? JSON.parse(cachedUser)
  : null;
```

---

# 8. TTL — Time To Live

এটা Redis-এর খুব important concept।

**TTL = Time To Live**

মানে কোনো data কতক্ষণ Redis-এ থাকবে।

ধরুন:

```text
OTP
↓
5 minutes
```

তাহলে 5 মিনিট পর Redis নিজে থেকে সেটা expire করতে পারবে।

---

# 9. Redis-এ TTL Set করা

```ts
await redis.set(
  'otp:user:123',
  '583921',
  'EX',
  300,
);
```

এখানে:

```text
EX
↓
Expiration

300
↓
300 seconds
↓
5 minutes
```

মানে:

```text
otp:user:123
      ↓
5 minutes
      ↓
Automatically expire
```

---

# 10. Cache TTL

Product cache:

```ts
await redis.set(
  'products:featured',
  JSON.stringify(products),
  'EX',
  300,
);
```

মানে cache 5 minutes থাকবে।

Flow:

```text
0 min
 ↓
Cache created

1 min
 ↓
Cache HIT

3 min
 ↓
Cache HIT

5 min
 ↓
Expired ❌

Next request
 ↓
Database
 ↓
New cache
```

---

# 11. TTL কেন দরকার?

ধরুন:

```text
Product price = 100
```

Redis-এ cache আছে:

```text
price = 100
```

কিন্তু database-এ price change হলো:

```text
price = 120
```

যদি cache কখনো expire না হয়:

```text
Redis → 100
Database → 120
```

User পুরোনো data পেতে পারে।

TTL দিলে:

```text
Cache
 ↓
Expire
 ↓
Fresh database data
 ↓
New cache
```

---

# 12. Cache-Aside Pattern

আপনার দেওয়া example মূলত **Cache-Aside Pattern**।

এটা খুব common।

Flow:

```text
Request
   ↓
Check Redis
   │
   ├── HIT
   │    ↓
   │  Return
   │
   └── MISS
        ↓
     Database
        ↓
     Save Redis
        ↓
      Return
```

---

# 13. NestJS Service Example

ধরুন:

```ts
@Injectable()
export class ProductService {
  constructor(
    private readonly prisma: PrismaService,
    private readonly redis: RedisService,
  ) {}

  async findAll() {
    const cacheKey = 'products:all';

    // 1. Redis check
    const cached =
      await this.redis.get(cacheKey);

    if (cached) {
      return JSON.parse(cached);
    }

    // 2. Database
    const products =
      await this.prisma.product.findMany();

    // 3. Cache
    await this.redis.set(
      cacheKey,
      JSON.stringify(products),
      'EX',
      300,
    );

    // 4. Return
    return products;
  }
}
```

এখন:

### First request

```text
GET /products
      ↓
Redis ❌
      ↓
PostgreSQL
      ↓
Redis SET
      ↓
Response
```

### Next request

```text
GET /products
      ↓
Redis ✅
      ↓
Response
```

---

# 14. Cache Invalidation

Caching-এর সবচেয়ে tricky অংশগুলোর একটি:

# Cache কখন delete/update করবেন?

ধরুন:

```text
Database
Product price = 100

Redis
Product price = 100
```

আপনি update করলেন:

```text
Database
Product price = 150
```

কিন্তু Redis:

```text
Redis
Product price = 100 ❌
```

এখন stale cache।

তাই product update করার সময় cache invalidate করতে হবে।

---

# 15. Cache Invalidation Example

```ts
async updateProduct(
  id: string,
  data: UpdateProductDto,
) {
  const product =
    await this.prisma.product.update({
      where: { id },
      data,
    });

  await this.redis.del(
    'products:all',
  );

  await this.redis.del(
    `product:${id}`,
  );

  return product;
}
```

Flow:

```text
Update Product
      ↓
PostgreSQL UPDATE
      ↓
Delete related Redis cache
      ↓
Next GET
      ↓
Cache MISS
      ↓
Database
      ↓
Fresh cache
```

---

# 16. Cache Key Design

Cache ব্যবহার করার সময় key naming important।

খারাপ:

```text
products
user
data
cache1
```

Better:

```text
product:123
user:123
products:list
products:category:vegetable
user:123:profile
```

একটা common pattern:

```text
resource:identifier
```

যেমন:

```text
user:123
product:456
order:789
```

---

# 17. Session-এর জন্য Redis

Redis শুধু cache না।

Session store হিসেবেও ব্যবহার করা যায়।

ধরুন:

```text
Session ID
↓
sess_abc123
```

Redis:

```text
sess_abc123
    ↓
userId: 123
    ↓
expires: 7 days
```

User request করলে:

```text
Request
 ↓
Session ID
 ↓
Redis
 ↓
User session
```

এটা distributed application-এ useful।

---

# 18. Rate Limiting

ধরুন:

```http
POST /auth/login
```

কেউ প্রতি second-এ হাজারবার request করছে।

এটা abuse হতে পারে।

আমরা rule দিতে পারি:

```text
একটি IP
↓
1 minute
↓
5 login attempts
```

Redis এখানে counter হিসেবে ব্যবহার করা যায়।

Concept:

```text
login:attempts:192.168.1.1
        ↓
        5
```

প্রতিটি request:

```text
Request
 ↓
Redis counter
 ↓
< 5 ?
 ├── YES → Allow
 └── NO  → Reject
```

---

# 19. Rate Limiting Example

Conceptually:

```ts
const key =
  `login:attempts:${ip}`;

const attempts =
  await redis.incr(key);

if (attempts === 1) {
  await redis.expire(
    key,
    60,
  );
}

if (attempts > 5) {
  throw new TooManyRequestsException(
    'Too many login attempts',
  );
}
```

এখানে:

```text
INCR
↓
Counter +1

EXPIRE
↓
Counter কতক্ষণ থাকবে
```

এটা Redis-এর rate limiting-এর basic idea।

---

# 20. OTP-এর জন্য Redis

OTP-এর জন্য Redis খুব useful।

ধরুন:

```text
User
↓
Request OTP
```

Generate:

```text
583921
```

Redis:

```text
otp:user:123
    ↓
583921
    ↓
TTL = 5 minutes
```

Code:

```ts
await redis.set(
  'otp:user:123',
  '583921',
  'EX',
  300,
);
```

Verify:

```ts
const otp =
  await redis.get(
    'otp:user:123',
  );

if (otp !== submittedOtp) {
  throw new BadRequestException(
    'Invalid OTP',
  );
}
```

Successful verification-এর পরে:

```ts
await redis.del(
  'otp:user:123',
);
```

---

# 21. Distributed Locks

এটা একটু advanced concept।

ধরুন একই সময়ে দুইটা server একই কাজ করতে যাচ্ছে:

```text
Server A
   ↓
Process payment

Server B
   ↓
Process payment
```

দুটোই যদি একই payment process করে:

```text
Payment
↓
Duplicate ❌
```

Redis দিয়ে distributed lock তৈরি করা যায়।

Concept:

```text
Redis
  ↓
lock:payment:123
  ↓
Server A owns lock
```

তখন:

```text
Server A → ✓ Process
Server B → ❌ Wait/Reject
```

---

# 22. Distributed Lock-এর Basic Idea

Redis-এর atomic `SET` operation-এর options ব্যবহার করে lock নেওয়া যায়:

```ts
const acquired =
  await redis.set(
    'lock:payment:123',
    'server-a',
    'NX',
    'EX',
    30,
  );
```

এখানে:

```text
NX
↓
Key না থাকলে set করো

EX 30
↓
30 seconds পরে expire
```

যদি:

```text
acquired === 'OK'
```

তাহলে lock পাওয়া গেছে।

অন্য server:

```text
acquired === null
```

পেতে পারে।

---

# 23. Queue Support

Redis queue system-এর backend হিসেবেও ব্যবহার করা যায়।

NestJS application-এ **BullMQ** খুব common choice।

Concept:

```text
User
 ↓
NestJS
 ↓
Queue
 ↓
Redis
 ↓
Worker
 ↓
Process Job
```

ধরুন user registration-এর পরে email পাঠাতে হবে।

আপনি request-এর মধ্যে email send না করে:

```text
Register User
     ↓
Create user
     ↓
Add email job
     ↓
Response
```

তারপর worker:

```text
Queue
 ↓
Send email
```

এতে API response দ্রুত হতে পারে।

---

# 24. Cache বনাম Queue

এগুলো confuse করবেন না।

### Cache

```text
Redis
 ↓
Frequently accessed data
```

### Queue

```text
Redis
 ↓
Pending jobs
```

উদাহরণ:

```text
Cache:
product:123 → product data

Queue:
email-job-123 → send welcome email
```

একই Redis infrastructure ব্যবহার হতে পারে, কিন্তু purpose আলাদা।

---

# 25. Redis-এর সব Use Case একসাথে

আপনার roadmap:

```text
Redis
│
├── Cache
│   └── Fast data access
│
├── TTL
│   └── Automatic expiration
│
├── Session
│   └── Session storage
│
├── Rate Limiting
│   └── Request counters
│
├── OTP
│   └── Temporary verification code
│
├── Distributed Locks
│   └── Prevent concurrent operations
│
└── Queue Support
    └── Background jobs
```

---

# 26. Redis + PostgreSQL Architecture

Production application-এ সাধারণ flow:

```text
                    Client
                      │
                      ▼
                    NestJS
                      │
              ┌───────┴───────┐
              │               │
              ▼               ▼
            Redis         PostgreSQL
              │               │
              │               │
         Fast/temporary    Persistent
             data             data
```

Cache request:

```text
Client
  ↓
NestJS
  ↓
Redis
  │
  ├── HIT → Response
  │
  └── MISS
       ↓
   PostgreSQL
       ↓
     Redis
       ↓
   Response
```

---

# 27. একটি Real Example

ধরুন আপনার Arabic Master application-এ:

```http
GET /words/popular
```

প্রথম request:

```text
GET /words/popular
        ↓
Redis
   ❌ cache miss
        ↓
PostgreSQL
        ↓
100 words
        ↓
Redis
TTL = 10 minutes
        ↓
Response
```

পরের 10 মিনিট:

```text
GET /words/popular
        ↓
Redis
   ✅ cache hit
        ↓
Response
```

Database query করার প্রয়োজন নেই।

10 মিনিট পরে:

```text
Redis
 ↓
Expired
 ↓
Cache MISS
 ↓
PostgreSQL
 ↓
Fresh data
 ↓
Redis
```

---

# 28. সবচেয়ে গুরুত্বপূর্ণ সমস্যা: Stale Data

Caching-এর সবচেয়ে common সমস্যা:

```text
Database
   ↓
New data

Redis
   ↓
Old data
```

এটাকে **stale cache** বলা হয়।

তাই cache strategy ঠিক করতে হবে:

```text
Read
 ↓
Cache first

Write
 ↓
Database
 ↓
Invalidate cache
```

Simple এবং common approach:

```text
READ
Redis → DB → Redis

WRITE
DB → Delete Redis cache
```

---

# 29. সব Data Cache করবেন না

এটা খুব important।

❌ এমন না:

```text
Every database query
       ↓
Cache
```

Cache করার আগে ভাববেন:

```text
এই data কি frequently read হয়?
       ↓
হ্যাঁ

Database query কি expensive?
       ↓
হ্যাঁ

Data কি কিছু সময় stale হতে পারবে?
       ↓
হ্যাঁ

→ Cache useful
```

যেমন:

```text
Popular products
Categories
Public configuration
Frequently accessed profiles
Frequently accessed content
```

এসব cache করার ভালো candidate হতে পারে।

---

# 30. Cache-এর Mental Model

এই flow-টা সবচেয়ে ভালোভাবে মনে রাখুন:

```text
                    Request
                       │
                       ▼
                    Redis
                       │
              ┌────────┴────────┐
              │                 │
           CACHE HIT        CACHE MISS
              │                 │
              ▼                 ▼
           Return           PostgreSQL
                                │
                                ▼
                              Redis
                                │
                                ▼
                             Return
```

আর Redis-এর broader role:

```text
                         Redis
                           │
       ┌──────────┬────────┼────────┬──────────┐
       │          │        │        │          │
     Cache      Session   OTP    Rate Limit   Lock
       │
       │
       └──────────── Queue / Jobs
```

**সবচেয়ে গুরুত্বপূর্ণ ৫টা জিনিস মনে রাখবেন:**

```text
1. Redis = fast in-memory data store

2. Cache HIT = Redis-এ data পাওয়া গেছে

3. Cache MISS = Redis-এ data নেই → Database

4. TTL = data কতক্ষণ Redis-এ থাকবে

5. Write-এর পরে related cache invalidate করতে হবে
```

এটাই আপনার NestJS **Caching & Redis** chapter-এর core foundation।
