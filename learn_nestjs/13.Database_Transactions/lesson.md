# 15. Database Transactions — NestJS

এটা financial এবং e-commerce application-এর জন্য **খুবই গুরুত্বপূর্ণ concept**।

সহজ ভাষায়:

> **Transaction হলো database-এর কয়েকটা operation-কে এমনভাবে একসাথে চালানো, যাতে সবগুলো সফল হয় অথবা কোনোটা সফল না হলে সব পরিবর্তন ফিরিয়ে দেওয়া হয়।**

ধরুন আপনি একটা order করছেন।

```text
Create Order
     ↓
Decrease Stock
     ↓
Create Payment
     ↓
Create Transaction
```

এখানে ৪টা database operation আছে।

ধরুন:

```text
Create Order       ✅
Decrease Stock     ✅
Create Payment     ❌
```

তাহলে কী হবে?

Transaction না থাকলে:

```text
Order created       ✅
Stock decreased     ✅
Payment created     ❌
```

Database এখন inconsistent।

কিন্তু transaction থাকলে:

```text
Create Order       ✅
Decrease Stock     ✅
Create Payment     ❌
        ↓
     ROLLBACK
        ↓
Create Order       ❌
Decrease Stock     ❌
```

অর্থাৎ database আগের অবস্থায় ফিরে যাবে।

---

# 1. Transaction আসলে কী?

একটা real-life example দেখি।

ধরুন আপনি ব্যাংক থেকে:

```text
আপনার account থেকে 1000 টাকা কমাবেন
        ↓
অন্য account-এ 1000 টাকা যোগ করবেন
```

প্রথম operation হলো:

```text
A = A - 1000
```

দ্বিতীয়:

```text
B = B + 1000
```

যদি প্রথমটা হয় কিন্তু দ্বিতীয়টা fail করে?

```text
A থেকে টাকা চলে গেছে
B টাকা পায়নি
```

এটা ভয়ংকর সমস্যা।

তাই database বলে:

```text
START TRANSACTION

A থেকে 1000 কমাও
B-তে 1000 যোগ করো

সব successful?
    ↓
   COMMIT

কোনো সমস্যা?
    ↓
   ROLLBACK
```

---

# 2. COMMIT এবং ROLLBACK

দুইটা শব্দ খুব ভালোভাবে মনে রাখুন।

## COMMIT

`COMMIT` মানে:

> সব operation সফল হয়েছে। পরিবর্তনগুলো permanent করো।

```text
START
  ↓
Operation 1 ✅
  ↓
Operation 2 ✅
  ↓
Operation 3 ✅
  ↓
COMMIT
```

---

## ROLLBACK

`ROLLBACK` মানে:

> কোনো সমস্যা হয়েছে। transaction-এর মধ্যে করা পরিবর্তনগুলো বাতিল করো।

```text
START
  ↓
Operation 1 ✅
  ↓
Operation 2 ✅
  ↓
Operation 3 ❌
  ↓
ROLLBACK
  ↓
সব আগের অবস্থায়
```

---

# 3. ACID

Database transaction-এর সাথে **ACID** খুব গুরুত্বপূর্ণ।

```text
A → Atomicity
C → Consistency
I → Isolation
D → Durability
```

---

## A — Atomicity

Atomicity মানে:

> Transaction-এর সব operation একসাথে সফল হবে, অথবা সবগুলো বাতিল হবে।

ধরুন:

```text
Create Order       ✅
Decrease Stock     ✅
Create Payment     ❌
```

তাহলে:

```text
ROLLBACK
```

ফলে:

```text
Order → বাতিল
Stock → আগের অবস্থায়
Payment → তৈরি হয়নি
```

এটাই Atomicity।

সহজভাবে:

```text
ALL or NOTHING
```

---

# 4. C — Consistency

Consistency মানে transaction-এর আগে এবং পরে database valid state-এ থাকবে।

ধরুন product:

```text
stock = 10
```

User 3টা কিনলো:

```text
stock = 7
```

এটা valid।

কিন্তু application bug-এর কারণে:

```text
stock = -500
```

হয়ে গেলে database business rules ভেঙে ফেলছে।

Transaction এবং database constraints মিলে data consistency বজায় রাখতে সাহায্য করে।

---

# 5. I — Isolation

একই সময়ে অনেক user database-এর সাথে কাজ করতে পারে।

ধরুন:

```text
User A → product কিনছে
User B → একই product কিনছে
```

দুইজন একই সময়ে stock update করলে সমস্যা হতে পারে।

ধরুন stock:

```text
1
```

দুইজনই দেখল:

```text
stock = 1
```

তারপর দুজনই order করল।

তাহলে theoretically:

```text
1 product
```

কিন্তু:

```text
2 orders
```

হয়ে যেতে পারে।

এই ধরনের concurrency problem handle করার জন্য transaction isolation গুরুত্বপূর্ণ।

---

# 6. D — Durability

Transaction `COMMIT` হওয়ার পরে data permanent থাকার guarantee হলো Durability।

```text
Transaction
    ↓
COMMIT
    ↓
Database
    ↓
Data persisted
```

Server restart হলেও committed data থাকার কথা।

---

# 7. E-commerce Example

এখন আপনার example:

```text
Create Order
     ↓
Decrease Stock
     ↓
Create Payment
     ↓
Create Transaction
```

ধরুন:

```text
Product stock = 10
```

User 2টা product কিনল।

Transaction-এর মধ্যে:

```text
1. Create Order

2. Stock:
   10 → 8

3. Create Payment

4. Create Transaction
```

সব successful:

```text
COMMIT
```

Final:

```text
Order       ✅
Stock = 8   ✅
Payment     ✅
Transaction ✅
```

---

# 8. মাঝখানে Failure

এবার ধরুন:

```text
Create Order       ✅
Decrease Stock     ✅
Create Payment     ❌
```

Transaction ছাড়া:

```text
Order exists
Stock decreased
Payment missing
```

এখন order system confused।

Transaction সহ:

```text
Create Order       ✅
Decrease Stock     ✅
Create Payment     ❌
        ↓
     ROLLBACK
```

Final:

```text
Order       ❌
Stock = 10  ✅
Payment     ❌
Transaction ❌
```

Database আগের অবস্থায় ফিরে গেল।

---

# 9. NestJS-এ Transaction

এখানে একটা গুরুত্বপূর্ণ বিষয়:

**NestJS নিজে transaction system না।**

Transaction আপনার database/ORM/ODM system handle করে।

যেমন:

```text
NestJS
   ↓
Prisma
   ↓
PostgreSQL
```

অথবা:

```text
NestJS
   ↓
TypeORM
   ↓
PostgreSQL
```

অথবা MongoDB হলে:

```text
NestJS
   ↓
Mongoose
   ↓
MongoDB
```

Transaction-এর API database library অনুযায়ী আলাদা হয়।

---

# 10. Prisma Transaction Example

ধরুন NestJS + Prisma।

```ts
async createOrder(userId: string, productId: string) {
  return this.prisma.$transaction(async (tx) => {

    const product = await tx.product.findUnique({
      where: {
        id: productId,
      },
    });

    if (!product) {
      throw new Error('Product not found');
    }

    if (product.stock <= 0) {
      throw new Error('Product is out of stock');
    }

    const order = await tx.order.create({
      data: {
        userId,
        productId,
        quantity: 1,
      },
    });

    await tx.product.update({
      where: {
        id: productId,
      },
      data: {
        stock: {
          decrement: 1,
        },
      },
    });

    await tx.payment.create({
      data: {
        orderId: order.id,
        amount: product.price,
        status: 'PENDING',
      },
    });

    return order;
  });
}
```

এখানে:

```ts
this.prisma.$transaction(...)
```

transaction শুরু করছে।

---

# 11. এখানে Transaction কীভাবে কাজ করছে?

এই code:

```ts
return this.prisma.$transaction(async (tx) => {
```

মানে:

```text
START TRANSACTION
        ↓
product read
        ↓
order create
        ↓
stock decrease
        ↓
payment create
        ↓
return order
        ↓
COMMIT
```

যদি সব successful হয়:

```text
COMMIT
```

কিন্তু:

```ts
await tx.payment.create(...)
```

এখানে error হলে:

```text
ROLLBACK
```

হবে।

---

# 12. `tx` কেন ব্যবহার করছি?

Transaction-এর ভিতরে:

```ts
tx.order.create()
tx.product.update()
tx.payment.create()
```

ব্যবহার করছি।

`tx` হলো সেই transaction-এর database client/context।

সহজভাবে:

```text
prisma
   │
   ├── normal query
   │
   └── $transaction()
           │
           └── tx
                ├── order
                ├── product
                └── payment
```

Transaction-এর operation-গুলো একই transaction-এর অংশ হিসেবে চালাতে হয়।

---

# 13. Transaction-এর বাইরে Query করলে সমস্যা

ধরুন:

```ts
return this.prisma.$transaction(async (tx) => {

  await tx.order.create(...);

  await this.prisma.product.update(...);

});
```

এখানে দ্বিতীয় query:

```ts
this.prisma.product.update()
```

ব্যবহার করা হয়েছে, `tx` নয়।

অর্থাৎ সেটা transaction-এর context-এর বাইরে চলে যেতে পারে।

তাই transaction-এর ভিতরে:

```ts
tx.order.create()
tx.product.update()
tx.payment.create()
```

ব্যবহার করবেন।

---

# 14. Mongoose Transaction

আপনি যেহেতু MongoDB/Mongoose নিয়ে কাজ করছেন, এটাও গুরুত্বপূর্ণ।

Mongoose-এ transaction-এর জন্য session ব্যবহার করা হয়।

```ts
const session = await this.connection.startSession();

try {
  session.startTransaction();

  const order = await this.orderModel.create(
    [
      {
        userId,
        productId,
        quantity: 1,
      },
    ],
    {
      session,
    },
  );

  await this.productModel.updateOne(
    {
      _id: productId,
      stock: { $gt: 0 },
    },
    {
      $inc: {
        stock: -1,
      },
    },
    {
      session,
    },
  );

  await this.paymentModel.create(
    [
      {
        orderId: order[0]._id,
        status: 'PENDING',
      },
    ],
    {
      session,
    },
  );

  await session.commitTransaction();

} catch (error) {

  await session.abortTransaction();

  throw error;

} finally {

  await session.endSession();

}
```

এখানে:

```ts
session.startTransaction();
```

transaction শুরু করে।

```ts
session.commitTransaction();
```

সব সফল হলে commit করে।

আর:

```ts
session.abortTransaction();
```

failure হলে rollback করে।

---

# 15. Mongoose Flow

উপরের code-টা conceptually:

```text
startSession()
      ↓
startTransaction()
      ↓
Create Order
      ↓
Decrease Stock
      ↓
Create Payment
      ↓
Everything OK?
   ↙          ↘
 YES           NO
  ↓             ↓
COMMIT       ABORT
  ↓             ↓
Permanent    ROLLBACK
```

---

# 16. Financial Application Example

ধরুন wallet transfer:

```text
User A balance = 5000
User B balance = 2000
```

A পাঠাবে:

```text
1000
```

Transaction:

```text
START
   ↓
A balance: 5000 → 4000
   ↓
B balance: 2000 → 3000
   ↓
Create transaction record
   ↓
COMMIT
```

Final:

```text
A = 4000
B = 3000
Transaction = created
```

---

## যদি B update fail করে?

```text
START
   ↓
A = 4000
   ↓
B update ❌
   ↓
ROLLBACK
```

Final:

```text
A = 5000
B = 2000
Transaction = not created
```

এটাই financial system-এর জন্য transaction-এর গুরুত্ব।

---

# 17. Transaction কখন ব্যবহার করবেন?

যখন একাধিক database operation **একসাথে সফল হওয়া business-এর জন্য জরুরি**, তখন transaction দরকার হতে পারে।

যেমন:

```text
Order
  +
Stock
  +
Payment
```

অথবা:

```text
Wallet debit
  +
Wallet credit
  +
Transaction record
```

অথবা:

```text
Investment
  +
Balance update
  +
Ledger entry
```

মূল চিন্তা:

```text
Operation A
Operation B
Operation C

এই তিনটা কি logically একটা কাজ?
        ↓
      YES
        ↓
Consider Transaction
```

---

# 18. Transaction-এর ভিতরে External API

এখানে একটা গুরুত্বপূর্ণ concept আছে।

ধরুন:

```text
START TRANSACTION
      ↓
Create Order
      ↓
Decrease Stock
      ↓
Call Stripe
      ↓
Create Payment
      ↓
COMMIT
```

`Stripe` database transaction-এর অংশ নয়।

আপনি database rollback করতে পারবেন:

```text
ROLLBACK database
```

কিন্তু external payment provider-এর network request-কে একই database transaction-এর মতো rollback করতে পারবেন না।

তাই:

```text
Database Transaction
```

আর:

```text
External API / Payment System
```

দুইটা আলাদা system।

এই কারণে real financial/e-commerce architecture-এ payment workflow carefully design করতে হয়।

---

# 19. Transaction-এর ভিতরে কী রাখা উচিত?

মূল principle:

```text
Transaction
   ↓
শুধু প্রয়োজনীয় database operations
```

যেমন:

```ts
await prisma.$transaction(async (tx) => {

  await tx.order.create(...);

  await tx.product.update(...);

  await tx.payment.create(...);

});
```

Transaction-এর মধ্যে অপ্রয়োজনীয় দীর্ঘ কাজ রাখলে transaction অনেকক্ষণ open থাকতে পারে।

---

# 20. পুরো বিষয়টা একবার দেখুন

আপনার e-commerce application:

```text
                    CREATE ORDER
                         │
                         ▼
                 START TRANSACTION
                         │
                         ▼
                  Create Order
                         │
                         ▼
                   Check Stock
                         │
                         ▼
                  Decrease Stock
                         │
                         ▼
                  Create Payment
                         │
                         ▼
                Create Transaction
                         │
                  ┌──────┴──────┐
                  │             │
               SUCCESS       FAILURE
                  │             │
                  ▼             ▼
                COMMIT       ROLLBACK
                  │             │
                  ▼             ▼
             Changes stay    Changes gone
```

সবচেয়ে সহজভাবে মনে রাখবেন:

```text
TRANSACTION

START
  ↓
A
  ↓
B
  ↓
C
  ↓
সব ঠিক?
 ├── YES → COMMIT
 └── NO  → ROLLBACK
```

**Transaction-এর মূল idea হলো:**

> **একটা business operation-এর একাধিক database change-কে এক unit হিসেবে treat করা—সব সফল হলে commit, কোনো critical step fail হলে rollback।**
