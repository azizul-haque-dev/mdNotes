# 16. Concurrency & Race Conditions — NestJS

এটা **senior-level backend engineering-এর খুব গুরুত্বপূর্ণ concept**, বিশেষ করে:

* Payment
* Order
* Inventory
* Wallet
* Coupon
* Booking
* Investment

এর মতো system-এ।

মূল সমস্যা হলো:

> **একই সময়ে একাধিক request এলে application কীভাবে data safe রাখবে?**

---

# 1. Concurrency কী?

ধরুন একজন user Pay button-এ একবার click করলেন।

সাধারণভাবে:

```text
User
 ↓
Request
 ↓
Server
 ↓
Payment
```

কিন্তু বাস্তবে network retry, double click, frontend bug বা user-এর multiple clicks-এর কারণে এমন হতে পারে:

```text
User clicks Pay
      │
      ├── Request 1
      ├── Request 2
      └── Request 3
           │
           ▼
        NestJS
```

তিনটা request প্রায় একই সময়ে server-এ আসছে।

এটাই **concurrency**-এর একটা simple example।

---

# 2. Race Condition কী?

**Race condition** হলো যখন multiple operations একই data-তে concurrently কাজ করে এবং কোন operation আগে/পরে execute হচ্ছে তার উপর final result নির্ভর করে।

একটা সহজ example দেখি।

ধরুন:

```text
Stock = 1
```

দুইজন user একই product কিনতে চায়।

```text
User A → Buy
User B → Buy
```

দুইটা request:

```text
Request A
Request B
```

দুটো server-এ প্রায় একই সময়ে পৌঁছাল।

---

## Unsafe code

ধরুন:

```ts
const product = await productModel.findById(productId);

if (product.stock > 0) {
  product.stock -= 1;

  await product.save();
}
```

দেখতে ঠিক মনে হচ্ছে।

কিন্তু concurrency-তে:

```text
Stock = 1

Request A → read stock = 1
Request B → read stock = 1

Request A → stock = 0
Request B → stock = 0
```

দুইজনই মনে করছে stock available।

ফলাফল:

```text
1 product
     ↓
2 orders
```

এটাই race condition।

---

# 3. Race Condition Visual

```text
Database
Stock = 1
   │
   ├───────────────┐
   │               │
   ▼               ▼
Request A       Request B
   │               │
read = 1        read = 1
   │               │
   ▼               ▼
decrease        decrease
   │               │
   └───────┬───────┘
           ▼
      Wrong result
```

সমস্যা হলো:

```text
READ
  ↓
CHECK
  ↓
WRITE
```

এই তিনটা operation আলাদা।

---

# 4. Atomic Operation

Race condition-এর একটা powerful solution হলো **atomic operation**।

Atomic operation মানে:

> Read + condition + update এমনভাবে করা যাতে অন্য request মাঝখানে inconsistent state দেখতে না পারে।

MongoDB example:

```ts
const result = await this.productModel.updateOne(
  {
    _id: productId,
    stock: { $gt: 0 },
  },
  {
    $inc: {
      stock: -1,
    },
  },
);
```

এখানে আমরা বলছি:

```text
শুধু তখনই stock -1 করো
যদি stock > 0 হয়
```

---

## কী হচ্ছে?

ধরুন:

```text
stock = 1
```

Request A:

```text
stock > 0 → YES
stock = 0
```

Request B:

```text
stock > 0 → NO
```

তাই B-এর update হবে না।

এখানে application-side:

```ts
if (product.stock > 0)
```

এর পরিবর্তে database-এর ভিতরেই condition + update করা হচ্ছে।

---

# 5. Atomic Operation-এর Result Check

শুধু update করলেই হবে না।

দেখতে হবে operation actually successful হয়েছে কিনা।

```ts
const result = await this.productModel.updateOne(
  {
    _id: productId,
    stock: { $gt: 0 },
  },
  {
    $inc: {
      stock: -1,
    },
  },
);

if (result.modifiedCount === 0) {
  throw new Error('Product is out of stock');
}
```

Flow:

```text
Request
   ↓
Atomic update
   ↓
modifiedCount?
   │
   ├── 1 → Stock successfully reserved
   │
   └── 0 → Stock unavailable
```

এটা inventory system-এ খুব useful pattern।

---

# 6. Optimistic Locking

এবার আরেকটা approach:

**Optimistic Locking**

এর idea:

> আমরা ধরে নিচ্ছি conflict সাধারণত হবে না। কিন্তু update করার সময় check করব data মাঝখানে অন্য কেউ change করেছে কিনা।

ধরুন:

```text
Product
stock = 10
version = 5
```

Request A এবং B দুজনই version `5` read করল।

```text
Request A → version 5
Request B → version 5
```

A update করল:

```text
stock = 9
version = 6
```

এখন B update করতে গিয়ে বলবে:

```text
আমি version 5 নিয়ে data read করেছিলাম।
কিন্তু database-এ এখন version 6।
```

তাই B-এর update reject হবে।

---

# 7. Optimistic Locking Example

ধরুন:

```ts
const product = await productModel.findById(productId);

const oldVersion = product.version;

product.stock -= 1;
product.version += 1;

const result = await productModel.updateOne(
  {
    _id: productId,
    version: oldVersion,
  },
  {
    $set: {
      stock: product.stock,
      version: oldVersion + 1,
    },
  },
);
```

যদি:

```ts
result.modifiedCount === 0
```

তাহলে অন্য কেউ data পরিবর্তন করেছে।

```ts
throw new Error(
  'Product was modified by another request',
);
```

---

# 8. Optimistic Locking Visual

```text
Database

stock = 10
version = 5

       ┌─────────────┐
       │             │
       ▼             ▼
   Request A      Request B
   version 5      version 5
       │             │
       ▼             ▼
   Update          Update
       │
       ▼
stock = 9
version = 6
       │
       │
       └───────────────┐
                       ▼
                  Request B
                  version = 5
                  but DB = 6
                       │
                       ▼
                     FAIL
```

---

# 9. Optimistic Locking কখন useful?

যখন conflict relatively কম হয় এবং database row/document দীর্ঘসময় lock করে রাখতে চান না।

যেমন:

```text
User profile
Product information
Order status
Document editing
```

---

# 10. Pessimistic Locking

এবার উল্টো চিন্তা।

Optimistic:

> "Conflict হবে না ধরে কাজ করি।"

Pessimistic:

> "Conflict হতে পারে, তাই আগে lock করি।"

ধরুন:

```text
Stock = 1
```

Request A product row lock করল।

```text
Request A
   ↓
LOCK product
   ↓
check stock
   ↓
decrease stock
   ↓
COMMIT
   ↓
UNLOCK
```

Request B meanwhile অপেক্ষা করবে।

```text
Request B
   ↓
LOCK চাই
   ↓
WAIT
```

A শেষ করার পরে:

```text
UNLOCK
   ↓
B gets lock
   ↓
check stock
   ↓
stock = 0
   ↓
FAIL
```

---

# 11. Pessimistic Locking Example

এটা relational database-এ খুব common।

যেমন PostgreSQL + TypeORM:

```ts
const product = await manager
  .getRepository(Product)
  .createQueryBuilder('product')
  .setLock('pessimistic_write')
  .where('product.id = :id', {
    id: productId,
  })
  .getOne();
```

এখানে:

```ts
.setLock('pessimistic_write')
```

database row lock করার জন্য ব্যবহৃত হচ্ছে।

তারপর:

```ts
if (product.stock <= 0) {
  throw new Error('Out of stock');
}

product.stock -= 1;

await manager.save(product);
```

Transaction-এর মধ্যে এটা ব্যবহার করা হয়।

---

# 12. Optimistic vs Pessimistic

| বিষয়        | Optimistic           | Pessimistic                      |
| ----------- | -------------------- | -------------------------------- |
| ধারণা       | Conflict কম হবে      | Conflict হতে পারে                |
| Lock        | সাধারণত lock নয়      | Lock করে                         |
| Conflict    | Update-এর সময় detect | আগে থেকেই prevent                |
| Performance | অনেক ক্ষেত্রে ভালো   | বেশি contention হলে ধীর হতে পারে |
| ব্যবহার     | কম conflict          | high-conflict critical data      |

সহজভাবে:

```text
Optimistic
"কাজ করি, পরে দেখি কেউ বদলেছে কিনা।"


Pessimistic
"আগে lock করি, তারপর কাজ করি।"
```

---

# 13. Distributed Lock

এখন আরও কঠিন problem।

ধরুন আপনার NestJS application-এর একটাই server নেই।

```text
                Load Balancer
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
       Server 1   Server 2   Server 3
```

একটা shared resource নিয়ে তিন server কাজ করছে।

যদি application memory-তে lock রাখেন:

```ts
let isProcessing = false;
```

তাহলে এটা কাজ করবে না।

কারণ:

```text
Server 1
isProcessing = true

Server 2
isProcessing = false
```

Server 2 জানেই না Server 1 lock করেছে।

---

# 14. Distributed Lock কেন দরকার?

ধরুন:

```text
Generate monthly payout
```

একই সময়ে:

```text
Server 1 → payout process
Server 2 → payout process
```

দুই server-ই কাজ করলে:

```text
User gets paid
User gets paid again
```

সমস্যা।

Distributed lock এমন একটা shared lock তৈরি করে যা সব server দেখতে পারে।

সাধারণ architecture:

```text
Server 1 ──┐
           │
Server 2 ──┼──> Redis
           │
Server 3 ──┘
```

Redis-এর মতো shared system ব্যবহার করে lock implement করা যায়।

---

# 15. Distributed Lock Example

Conceptually:

```ts
const lock = await redis.set(
  `lock:payment:${paymentId}`,
  'locked',
  'NX',
  'EX',
  30,
);
```

এখানে:

```text
NX
↓
key আগে না থাকলে তৈরি করো

EX 30
↓
30 seconds পরে expire
```

যদি:

```ts
lock === 'OK'
```

তাহলে server lock পেয়েছে।

```ts
if (!lock) {
  throw new Error(
    'Payment is already being processed',
  );
}
```

তারপর:

```text
Acquire lock
      ↓
Process payment
      ↓
Release lock
```

---

# 16. কিন্তু Distributed Lock একাই যথেষ্ট না

এটা খুব গুরুত্বপূর্ণ।

ধরুন:

```text
Lock acquired
     ↓
Payment processing
     ↓
Server crashes
```

তাহলে lock forever থেকে যেতে পারে।

তাই সাধারণত lock-এর সাথে expiration/lease ব্যবহার করা হয়।

```text
Lock
 ↓
TTL = 30 seconds
 ↓
Process
```

এবং real distributed locking আরও subtle; lock ownership, expiry, retries ইত্যাদি carefully handle করতে হয়।

---

# 17. Idempotency

এখন payment system-এর **সবচেয়ে গুরুত্বপূর্ণ concept**-গুলোর একটা:

# Idempotency

সহজ ভাষায়:

> **একই request একবার বা একাধিকবার execute হলেও final business result যেন একই থাকে।**

ধরুন:

```text
POST /payments
```

User Pay click করল।

Network problem হলো।

Frontend ভাবল:

```text
Payment failed
```

তাই আবার request পাঠাল।

```text
Request 1
Request 2
```

দুই request একই payment-এর জন্য।

যদি backend দুটোই process করে:

```text
Payment = 1000
Payment = 1000

Total charged = 2000
```

ভয়ংকর সমস্যা।

---

# 18. Idempotency Key

Client একটা unique key পাঠাবে:

```http
Idempotency-Key: 7b8e9c-payment-123
```

Request:

```http
POST /payments
Idempotency-Key: abc123
```

Server প্রথমবার:

```text
abc123
↓
Not found
↓
Process payment
↓
Save result
```

দ্বিতীয়বার:

```text
abc123
↓
Already exists
↓
Don't process again
↓
Return previous result
```

---

# 19. Idempotency Database Example

ধরুন:

```ts
Payment {
  id
  userId
  amount
  idempotencyKey
  status
}
```

এবং:

```text
idempotencyKey = UNIQUE
```

ধরুন প্রথম request:

```json
{
  "amount": 1000,
  "idempotencyKey": "abc123"
}
```

Database:

```text
abc123 → Payment #1
```

দ্বিতীয় request:

```json
{
  "amount": 1000,
  "idempotencyKey": "abc123"
}
```

Server দেখে:

```text
abc123 already exists
```

তাই নতুন payment create করবে না।

---

# 20. NestJS Idempotency Example

ধরুন:

```ts
@Post('payments')
async createPayment(
  @Headers('idempotency-key')
  idempotencyKey: string,
  @Body() dto: CreatePaymentDto,
) {
  const existing =
    await this.paymentModel.findOne({
      idempotencyKey,
    });

  if (existing) {
    return existing;
  }

  const payment =
    await this.paymentModel.create({
      amount: dto.amount,
      idempotencyKey,
      status: 'PENDING',
    });

  return payment;
}
```

Concept ঠিক আছে, কিন্তু production concurrency-তে শুধু এই check যথেষ্ট নয়।

কারণ দুই request একই সময়ে:

```text
Request A → find → not found
Request B → find → not found
```

দুটোই create করতে পারে।

তাই database-এ **unique constraint/index** দরকার।

---

# 21. Unique Constraint

MongoDB:

```ts
@Prop({
  unique: true,
})
idempotencyKey: string;
```

অথবা schema index:

```ts
PaymentSchema.index(
  { idempotencyKey: 1 },
  { unique: true },
);
```

তখন:

```text
Request A
   ↓
abc123 → create ✅

Request B
   ↓
abc123 → duplicate key ❌
```

এখন application duplicate request handle করতে পারে।

---

# 22. Duplicate Requests

Duplicate request অনেক কারণেই হতে পারে:

```text
User double click
      ↓
Request 1
Request 2
```

অথবা:

```text
Network timeout
      ↓
Client retries
      ↓
Same request again
```

অথবা:

```text
Mobile app
      ↓
Poor network
      ↓
Retry
```

অথবা:

```text
Message queue
      ↓
Message delivered again
```

তাই backend-কে ধরে নিতে হবে:

> **একটা request একবারই আসবে—এটা ধরে নেওয়া যাবে না।**

---

# 23. Payment System-এর Complete Protection

এখন আপনার example:

```text
User clicks Pay
      ↓
Request 1
Request 2
Request 3
```

আমরা কী করব?

### Step 1 — Idempotency Key

সব request:

```text
Idempotency-Key: payment-abc123
```

---

### Step 2 — Database Unique Constraint

```text
idempotencyKey UNIQUE
```

---

### Step 3 — Transaction

Payment-related database operations:

```text
Create Payment
Update Order
Create Transaction
```

একটা transaction-এর মধ্যে রাখা।

---

### Step 4 — Atomic State Transition

Payment:

```text
PENDING
   ↓
PROCESSING
   ↓
SUCCEEDED
```

একই payment যেন:

```text
SUCCEEDED
   ↓
SUCCEEDED again
```

না হয়।

---

# 24. State Transition

Payment-এর state:

```ts
enum PaymentStatus {
  PENDING = 'PENDING',
  PROCESSING = 'PROCESSING',
  SUCCEEDED = 'SUCCEEDED',
  FAILED = 'FAILED',
}
```

তারপর update করার সময় condition:

```ts
await paymentModel.updateOne(
  {
    _id: paymentId,
    status: 'PENDING',
  },
  {
    $set: {
      status: 'PROCESSING',
    },
  },
);
```

শুধু:

```text
PENDING
```

থাকলেই `PROCESSING` করা যাবে।

যদি অন্য request আগে করে ফেলে:

```text
Request A:
PENDING → PROCESSING

Request B:
PENDING → ❌
```

এটা atomic state transition-এর useful pattern।

---

# 25. Payment Flow

একটা robust conceptual flow:

```text
Client
  │
  │ POST /payments
  │ Idempotency-Key: abc123
  ▼
NestJS
  │
  ▼
Check Idempotency
  │
  ├── Already processed
  │        ↓
  │   Return old result
  │
  └── New request
           ↓
     Create/Find Payment
           ↓
      Atomic transition
           ↓
       PROCESSING
           ↓
      Payment Provider
           ↓
       SUCCESS / FAIL
           ↓
       DB Transaction
           ↓
        COMMIT
```

---

# 26. Race Condition + Transaction

একটা important distinction:

**Transaction আর race condition একই জিনিস না।**

Transaction:

```text
Multiple database operations
        ↓
All succeed / all rollback
```

Race condition:

```text
Multiple concurrent operations
        ↓
Conflict
```

Transaction race condition prevent করতে সাহায্য করতে পারে, কিন্তু **শুধু transaction ব্যবহার করলেই সব race condition automatically solve হয় না।**

প্রয়োজনে দরকার হতে পারে:

```text
Transaction
+
Atomic update
+
Locking
+
Unique constraint
+
Idempotency
```

কোনটা লাগবে সেটা problem-এর উপর নির্ভর করে।

---

# 27. Real E-commerce Example

ধরুন:

```text
Stock = 1
```

User A এবং B একই product কিনছে।

Robust approach:

```text
Request A
   ↓
Transaction
   ↓
Atomic stock update
   ↓
stock 1 → 0
   ↓
Create Order
   ↓
COMMIT
```

Request B:

```text
Request B
   ↓
Atomic stock update
   ↓
stock > 0 ?
   ↓
NO
   ↓
Out of stock
```

Final:

```text
Stock = 0
Orders = 1
```

---

# 28. কোন Problem-এর জন্য কোন Tool?

এটা মনে রাখার জন্য একটা table:

| Problem                          | Common solution         |
| -------------------------------- | ----------------------- |
| Stock update race                | Atomic update           |
| Multiple DB operations           | Transaction             |
| Concurrent record modification   | Optimistic locking      |
| High-contention DB resource      | Pessimistic locking     |
| Multiple server instance একই কাজ | Distributed lock        |
| Payment retry                    | Idempotency             |
| Duplicate DB record              | Unique constraint       |
| Payment state conflict           | Atomic state transition |

---

# 29. সবচেয়ে গুরুত্বপূর্ণ Mental Model

আপনার backend-এ কখনো ধরে নেবেন না:

```text
"User একবারই request পাঠাবে।"
```

বরং ধরে নেবেন:

```text
Request
Request
Request
Request
     ↓
একই resource
```

তাই backend design করবেন:

```text
                 Concurrent Requests
                        │
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
       Request 1     Request 2     Request 3
          │             │             │
          └─────────────┼─────────────┘
                        ▼
                  Safe Database
                        │
              ┌─────────┼─────────┐
              ▼         ▼         ▼
           Atomic    Locking   Idempotency
           Update
              │
              ▼
          Transaction
              │
              ▼
        Consistent Data
```

শেষে এই ৬টা শব্দ মাথায় রাখুন:

```text
Race Condition
     ↓
"একই সময়ে conflict"

Atomic Operation
     ↓
"একসাথে safe update"

Optimistic Lock
     ↓
"change হয়েছে কিনা check"

Pessimistic Lock
     ↓
"আগে lock, পরে কাজ"

Distributed Lock
     ↓
"সব server-এর জন্য shared lock"

Idempotency
     ↓
"same request বারবার হলেও duplicate effect নয়"
```

Payment/order system-এ সবচেয়ে গুরুত্বপূর্ণ mindset হলো: **concurrency exception না—এটা normal।** আপনার backend এমনভাবে design করতে হবে যেন একই resource-এর উপর একসাথে অনেক request এলেও database-এর final state ভুল না হয়।
