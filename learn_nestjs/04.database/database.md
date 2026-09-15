অবশ্যই। আমরা **PostgreSQL + NestJS** ধরে একদম শুরু থেকে **production-grade database thinking** শিখব। আমি প্রতিটি বিষয় এমনভাবে explain করব যেন আপনি ১৫ বছরের, কিন্তু code এবং architecture হবে একজন experienced NestJS/backend engineer-এর মতো।

---

# 4. Database — PostgreSQL

প্রথমে পুরো picture-টা দেখুন:

```text
NestJS Application
       │
       ▼
   Service Layer
       │
       ▼
 ORM / Query Builder
 (Prisma / TypeORM)
       │
       ▼
 Connection Pool
       │
       ▼
   PostgreSQL
       │
       ├── Tables
       ├── Relations
       ├── Indexes
       ├── Constraints
       └── Transactions
```

একটা real application-এ Database শুধু data রাখার জায়গা না।

Database-এর দায়িত্ব হলো:

* data সঠিক রাখা
* data-এর relationship maintain করা
* duplicate/invalid data আটকানো
* concurrent request handle করা
* query দ্রুত করা
* transaction-এর মাধ্যমে consistency রাখা
* schema পরিবর্তনের history রাখা

এখন এক এক করে দেখি।

---

# 1. PostgreSQL

PostgreSQL হলো একটি **relational database**।

সহজভাবে বললে:

> PostgreSQL-এ data table আকারে রাখা হয় এবং table-এর মধ্যে relationship তৈরি করা যায়।

ধরুন আমাদের application-এ user আছে।

### users

| id | name  | email                                     |
| -- | ----- | ----------------------------------------- |
| 1  | Aziz  | [aziz@gmail.com](mailto:aziz@gmail.com)   |
| 2  | Rahim | [rahim@gmail.com](mailto:rahim@gmail.com) |

আর তাদের orders আছে।

### orders

| id | user_id | amount |
| -- | ------: | -----: |
| 1  |       1 |    500 |
| 2  |       1 |    800 |
| 3  |       2 |    300 |

এখানে:

```text
User
 │
 ├── Order
 ├── Order
 └── Order
```

`orders.user_id` user-এর সাথে relationship তৈরি করছে।

---

# 2. Prisma অথবা TypeORM

NestJS নিজে database-এর সাথে কথা বলার জন্য ORM দেয় না।

তাই আমরা সাধারণত ব্যবহার করি:

```text
NestJS
   │
   ├── Prisma
   │
   └── TypeORM
          │
          ▼
      PostgreSQL
```

## ORM কী?

ORM = **Object Relational Mapping**

Database-এ আমরা লিখি:

```sql
SELECT * FROM users WHERE id = 1;
```

কিন্তু application code-এ আমরা চাই:

```ts
const user = await prisma.user.findUnique({
  where: {
    id: 1,
  },
});
```

ORM আমাদের application code এবং SQL database-এর মধ্যে bridge হিসেবে কাজ করে।

---

# Prisma

Prisma-এর schema এমন হতে পারে:

```prisma
model User {
  id        Int      @id @default(autoincrement())
  name      String
  email     String   @unique
  createdAt DateTime @default(now())

  orders    Order[]
}

model Order {
  id        Int      @id @default(autoincrement())
  amount    Decimal
  createdAt DateTime @default(now())

  userId    Int
  user      User     @relation(fields: [userId], references: [id])
}
```

তারপর NestJS service:

```ts
@Injectable()
export class UsersService {
  constructor(private readonly prisma: PrismaService) {}

  async findUser(id: number) {
    return this.prisma.user.findUnique({
      where: {
        id,
      },
    });
  }
}
```

---

# TypeORM

TypeORM-এ একই concept entity দিয়ে করা হয়।

```ts
@Entity()
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  name: string;

  @Column({ unique: true })
  email: string;

  @OneToMany(() => Order, order => order.user)
  orders: Order[];
}
```

Service:

```ts
@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private readonly usersRepository: Repository<User>,
  ) {}

  async findUser(id: number) {
    return this.usersRepository.findOne({
      where: { id },
    });
  }
}
```

### কোনটা বুঝবেন?

দুটোর concept বুঝুন:

```text
Prisma
    ↓
Schema-first
    ↓
Prisma Client

TypeORM
    ↓
Entity-first
    ↓
Repository
```

আপনি Prisma ব্যবহার করলেও database fundamentals একই থাকবে।

---

# 3. Relations

এটা Database-এর সবচেয়ে important concept-এর একটি।

ধরুন:

```text
User
 │
 ├── Order
 ├── Order
 └── Order
```

একজন user-এর অনেক order।

এটা:

## One-to-Many

```text
One User
   │
   ├── Many Orders
   ├── Many Orders
   └── Many Orders
```

Database:

```sql
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100)
);

CREATE TABLE orders (
  id SERIAL PRIMARY KEY,
  amount DECIMAL(10,2),
  user_id INTEGER NOT NULL,

  FOREIGN KEY (user_id)
  REFERENCES users(id)
);
```

এখানে:

```sql
user_id
```

হলো foreign key।

---

## One-to-One

ধরুন:

```text
User
 │
 └── Profile
```

একজন user-এর একটি profile।

```prisma
model User {
  id      Int      @id @default(autoincrement())
  profile Profile?
}

model Profile {
  id     Int  @id @default(autoincrement())
  userId Int  @unique

  user User @relation(fields: [userId], references: [id])
}
```

এখানে `userId @unique` থাকার কারণে একজন user-এর একটির বেশি profile হতে পারবে না।

---

## Many-to-Many

ধরুন:

```text
Student
   │
   ├──── Course
   ├──── Course
   │
   └──── Course

Course
   │
   ├──── Student
   ├──── Student
   └──── Student
```

একজন student অনেক course নিতে পারে।

একটা course-এ অনেক student থাকতে পারে।

সরাসরি দুইটা table দিয়ে এটা সাধারণত করা হয় না।

আমরা junction table ব্যবহার করি:

```text
students
courses
student_courses
```

```sql
CREATE TABLE student_courses (
  student_id INTEGER REFERENCES students(id),
  course_id INTEGER REFERENCES courses(id),

  PRIMARY KEY (student_id, course_id)
);
```

---

# 4. Transactions

Transaction মানে:

> কয়েকটি database operation-কে একটি single unit হিসেবে execute করা।

সব operation সফল হবে, অথবা সব rollback হবে।

ধরুন bank transfer:

```text
Rahim
  ↓
-100 টাকা

Karim
  ↓
+100 টাকা
```

যদি প্রথম query successful হয়:

```sql
UPDATE accounts
SET balance = balance - 100
WHERE id = 1;
```

কিন্তু দ্বিতীয় query fail করে:

```sql
UPDATE accounts
SET balance = balance + 100
WHERE id = 2;
```

তাহলে database-এর অবস্থা ভয়ংকর হতে পারে:

```text
Rahim: 900
Karim: 500

100 টাকা হারিয়ে গেল!
```

Transaction এই problem solve করে।

```text
BEGIN

  deduct money

  add money

COMMIT
```

কোনো জায়গায় error হলে:

```text
ROLLBACK
```

---

## Prisma transaction

```ts
await prisma.$transaction(async (tx) => {
  await tx.account.update({
    where: {
      id: senderId,
    },
    data: {
      balance: {
        decrement: 100,
      },
    },
  });

  await tx.account.update({
    where: {
      id: receiverId,
    },
    data: {
      balance: {
        increment: 100,
      },
    },
  });
});
```

যদি দ্বিতীয় operation fail করে:

```text
First operation
      ↓
Second operation ❌
      ↓
ROLLBACK
      ↓
First operation-ও undo
```

---

# 5. Indexes

Index-এর সবচেয়ে সহজ example হলো বই।

ধরুন ১০০০ পৃষ্ঠার বইয়ে আপনি খুঁজছেন:

```text
"PostgreSQL"
```

Index না থাকলে আপনাকে:

```text
Page 1
Page 2
Page 3
...
Page 1000
```

চেক করতে হতে পারে।

কিন্তু বইয়ের শেষে যদি index থাকে:

```text
PostgreSQL → Page 735
```

তাহলে সরাসরি সেখানে যেতে পারবেন।

Database index-ও একই idea।

---

ধরুন:

```sql
SELECT *
FROM users
WHERE email = 'aziz@gmail.com';
```

`email`-এ index থাকলে database দ্রুত user খুঁজে পেতে পারে।

```sql
CREATE INDEX idx_users_email
ON users(email);
```

তবে যদি email unique হয়:

```sql
email VARCHAR(255) UNIQUE
```

তাহলে PostgreSQL সাধারণত unique constraint-এর জন্য index তৈরি করে।

---

## Index-এর downside

Index magic না।

ধরুন table:

```text
1 million users
```

আপনি index বানালেন:

```sql
CREATE INDEX idx_users_name
ON users(name);
```

Read দ্রুত হবে।

কিন্তু:

```text
INSERT
UPDATE
DELETE
```

এর সময় index maintain করতে হবে।

তাই:

> প্রয়োজন ছাড়া প্রতিটি column-এ index দেবেন না।

---

# 6. Constraints

Constraint হলো database-এর security guard।

Application ভুল data পাঠালেও database যেন invalid data গ্রহণ না করে।

---

## NOT NULL

```sql
name VARCHAR(100) NOT NULL
```

মানে:

```text
name অবশ্যই থাকতে হবে
```

এটা invalid:

```json
{
  "name": null
}
```

---

## UNIQUE

```sql
email VARCHAR(255) UNIQUE
```

মানে:

```text
একই email দুইবার থাকতে পারবে না।
```

---

## PRIMARY KEY

```sql
id SERIAL PRIMARY KEY
```

প্রতিটি row-এর unique identity।

```text
User 1
User 2
User 3
```

---

## FOREIGN KEY

```sql
user_id INTEGER REFERENCES users(id)
```

এটা database-কে বলে:

> এই order-এর user অবশ্যই users table-এ থাকতে হবে।

---

## CHECK

ধরুন price negative হতে পারবে না:

```sql
price DECIMAL(10,2)
CHECK (price >= 0)
```

তাহলে:

```text
price = -500
```

database reject করবে।

---

## Constraint-এর philosophy

শুধু NestJS validation-এর উপর depend করবেন না।

```text
Frontend validation
        ↓
NestJS DTO validation
        ↓
Database constraints
```

শেষ protection হলো database।

---

# 7. Migration

Migration হলো database schema পরিবর্তনের **version history**।

ধরুন প্রথমে User table ছিল:

```text
users

id
name
email
```

পরে আপনার requirement হলো:

```text
password
```

যোগ করতে হবে।

Migration:

```sql
ALTER TABLE users
ADD COLUMN password VARCHAR(255);
```

এখন history:

```text
Migration 001
    ↓
Create users

Migration 002
    ↓
Add password

Migration 003
    ↓
Add created_at
```

এটা production-এর জন্য খুব important।

---

## Prisma example

Schema পরিবর্তন:

```prisma
model User {
  id       Int    @id @default(autoincrement())
  name     String
  email    String @unique
  password String
}
```

তারপর:

```bash
npx prisma migrate dev --name add_password
```

Prisma migration তৈরি করবে।

Production-এ:

```bash
npx prisma migrate deploy
```

Migration-এর সুবিধা হলো:

```text
Developer A
      ↓
Migration 001

Developer B
      ↓
Migration 002

Production
      ↓
001 → 002
```

সব environment একই schema-এর দিকে যাবে।

---

# 8. N+1 Problem

এটা backend developer হিসেবে অবশ্যই বুঝতে হবে।

ধরুন:

```text
100 users
```

আপনি প্রথমে করেন:

```sql
SELECT * FROM users;
```

এটা হলো:

```text
1 query
```

তারপর প্রতিটি user-এর orders:

```sql
SELECT * FROM orders WHERE user_id = 1;
SELECT * FROM orders WHERE user_id = 2;
SELECT * FROM orders WHERE user_id = 3;
...
```

100 users হলে:

```text
1 + 100
= 101 queries
```

এটাই:

# N+1 Problem

---

## Bad

```ts
const users = await prisma.user.findMany();

for (const user of users) {
  user.orders = await prisma.order.findMany({
    where: {
      userId: user.id,
    },
  });
}
```

ধরুন:

```text
100 users
```

তাহলে প্রায়:

```text
101 DB queries
```

---

## Better

একবারেই relation load করুন:

```ts
const users = await prisma.user.findMany({
  include: {
    orders: true,
  },
});
```

Conceptually database-এর কাজ হবে:

```text
Users
   +
Orders
   ↓
combined result
```

তবে `include` ব্যবহার করলেই blindly সব relation load করা উচিত না। প্রয়োজন অনুযায়ী data select করবেন।

```ts
const users = await prisma.user.findMany({
  select: {
    id: true,
    name: true,
    orders: {
      select: {
        id: true,
        amount: true,
      },
    },
  },
});
```

---

# 9. Query Optimization

Query optimization মানে:

> একই কাজ কম database resource এবং কম সময়ে করা।

ধরুন:

```sql
SELECT *
FROM users
WHERE email = 'aziz@gmail.com';
```

প্রথম প্রশ্ন:

```text
email-এর index আছে?
```

তারপর query:

```sql
EXPLAIN ANALYZE
SELECT *
FROM users
WHERE email = 'aziz@gmail.com';
```

PostgreSQL আপনাকে query কীভাবে execute করেছে তার plan দেখাবে।

---

## `SELECT *` সবসময় দরকার নেই

যদি আপনার শুধু:

```text
id
name
```

লাগে:

```ts
const user = await prisma.user.findUnique({
  where: {
    id,
  },
  select: {
    id: true,
    name: true,
  },
});
```

পুরো user object নেওয়ার দরকার নেই।

---

## Pagination

ধরুন 1 million products:

```text
products = 1,000,000
```

এভাবে করা খারাপ:

```sql
SELECT *
FROM products;
```

বরং:

```sql
SELECT *
FROM products
LIMIT 20;
```

আর page-based pagination:

```sql
SELECT *
FROM products
ORDER BY id
LIMIT 20
OFFSET 100;
```

বড় dataset-এ deep `OFFSET` slow হতে পারে।

তখন cursor-based pagination ভালো:

```ts
const products = await prisma.product.findMany({
  take: 20,

  cursor: {
    id: lastProductId,
  },

  orderBy: {
    id: 'asc',
  },
});
```

---

# 10. Connection Pooling

এটা খুব important production concept।

ধরুন NestJS-এর 1000 request এসেছে।

প্রতিটি request যদি PostgreSQL-এর জন্য নতুন connection তৈরি করে:

```text
Request 1 → DB connection
Request 2 → DB connection
Request 3 → DB connection
...
Request 1000 → DB connection
```

এটা database-এর জন্য terrible।

Connection pool এই problem solve করে।

```text
NestJS
   │
   ▼
Connection Pool
   │
   ├── Connection 1
   ├── Connection 2
   ├── Connection 3
   ├── Connection 4
   └── Connection 5
             │
             ▼
         PostgreSQL
```

Request এলে existing connection ব্যবহার করবে।

```text
Request
   ↓
Get connection from pool
   ↓
Execute query
   ↓
Return connection to pool
```

তাই production application-এ connection pool configuration গুরুত্বপূর্ণ।

---

# 11. Database Locking

এটা আপনার list-এ না থাকলেও database-এর core অংশ হিসেবে জানা দরকার।

ধরুন account balance:

```text
1000
```

একই সময়ে দুইটা request এলো:

```text
Request A → withdraw 800
Request B → withdraw 800
```

দুই request যদি একই সময়ে দেখে:

```text
balance = 1000
```

তাহলে দুইজনই ভাবতে পারে:

```text
1000 >= 800
```

এবং দুই withdrawal allow হয়ে যেতে পারে।

এখানে locking/concurrency control দরকার।

PostgreSQL-এ row lock:

```sql
SELECT *
FROM accounts
WHERE id = 1
FOR UPDATE;
```

এতে transaction শেষ না হওয়া পর্যন্ত ওই row নিয়ে concurrent operation আটকে রাখা যায়।

---

# 12. ACID

Transaction বুঝতে ACID জানা দরকার।

```text
A → Atomicity
C → Consistency
I → Isolation
D → Durability
```

### Atomicity

সব হবে:

```text
SUCCESS
```

অথবা কিছুই হবে না:

```text
ROLLBACK
```

### Consistency

Database valid state-এ থাকবে।

যেমন:

```text
balance < 0
```

যদি allowed না হয়, database সেটা prevent করবে।

### Isolation

এক transaction-এর কাজ অন্য concurrent transaction-এর উপর uncontrolled effect ফেলবে না।

### Durability

Transaction commit হওয়ার পর data হারিয়ে যাওয়ার কথা নয়—even after restart/crash, subject to the database's durability guarantees.

---

# 13. Soft Delete

অনেক production application-এ সরাসরি data delete না করে:

```text
deletedAt
```

রাখা হয়।

```prisma
model User {
  id        Int       @id @default(autoincrement())
  email     String    @unique
  deletedAt DateTime?
}
```

Delete:

```ts
await prisma.user.update({
  where: {
    id,
  },
  data: {
    deletedAt: new Date(),
  },
});
```

তারপর normal query:

```ts
const users = await prisma.user.findMany({
  where: {
    deletedAt: null,
  },
});
```

মানে:

```text
Database
│
├── Active users
│
└── Deleted users
      ↓
   still stored
```

এটা বিশেষ করে audit/history প্রয়োজন হলে useful।

---

# 14. Data Modeling / Schema Design

Database শেখার আরেকটি বড় অংশ হলো:

> Data কীভাবে table-এ structure করবেন?

ধরুন e-commerce:

```text
users
products
categories
orders
order_items
payments
```

এভাবে আলাদা table:

```text
User
 │
 └── Orders
       │
       └── OrderItems
              │
              └── Products
```

Order-এর মধ্যে product-এর current price দরকার হলে শুধু product-এর current price-এর উপর depend করলে সমস্যা হতে পারে।

তাই:

```text
order_items
----------------
id
order_id
product_id
quantity
unit_price
```

এখানে `unit_price` order-এর সময়ের price preserve করে।

এটাই production database design-এর গুরুত্বপূর্ণ চিন্তা:

> Database শুধু আজকের data নয়, business history-ও preserve করে।

---

# 15. Normalization

Normalization-এর উদ্দেশ্য হলো data duplication কমানো এবং data consistency রাখা।

Bad design:

```text
orders

id
customer_name
customer_email
customer_phone
product_name
product_price
```

ধরুন একই customer 100 order করেছে।

তাহলে customer information বারবার থাকবে।

Better:

```text
users
orders
products
order_items
```

```text
users
   ↓
orders
   ↓
order_items
   ↓
products
```

তবে সবকিছু blindly normalize করাও goal নয়। Read-heavy system-এ business requirement অনুযায়ী controlled denormalization করা হতে পারে।

---

# 16. PostgreSQL Data Types

Production schema design-এ type choice গুরুত্বপূর্ণ।

যেমন:

```sql
name VARCHAR(100)

email VARCHAR(255)

age INTEGER

price NUMERIC(12,2)

is_active BOOLEAN

created_at TIMESTAMPTZ
```

Date/time-এর ক্ষেত্রে PostgreSQL-এ সাধারণত:

```sql
TIMESTAMPTZ
```

ব্যবহার করা ভালো, বিশেষ করে distributed application হলে।

---

# 17. UUID বনাম Auto Increment ID

দুই ধরনের ID দেখতে পারেন:

```text
1
2
3
4
```

অথবা:

```text
550e8400-e29b-41d4-a716-446655440000
```

দ্বিতীয়টি UUID।

Prisma:

```prisma
id String @id @default(uuid())
```

অথবা integer:

```prisma
id Int @id @default(autoincrement())
```

কোনটি ব্যবহার করবেন সেটা system-এর requirements-এর উপর নির্ভর করে।

---

# 18. Foreign Key + Cascade

ধরুন:

```text
User
 │
 └── Orders
```

User delete করলে orders-এর কী হবে?

Database relationship-এ define করা যায়:

```sql
FOREIGN KEY (user_id)
REFERENCES users(id)
ON DELETE CASCADE
```

মানে:

```text
Delete User
     ↓
Delete related Orders
```

কিন্তু production-এ `CASCADE` খুব ভেবে ব্যবহার করতে হয়।

কারণ ভুল relationship হলে:

```text
Delete A
  ↓
Delete B
  ↓
Delete C
  ↓
Delete D
```

অনেক data একসাথে delete হয়ে যেতে পারে।

---

# 19. Database Security

Database credentials code-এর মধ্যে hard-code করবেন না।

❌

```ts
const password = "mypassword";
```

বরং environment variable:

```env
DATABASE_URL="postgresql://user:password@localhost:5432/app"
```

NestJS:

```ts
const databaseUrl = configService.get<string>(
  'DATABASE_URL',
);
```

এবং production environment-এ secrets secure secret-management mechanism দিয়ে manage করবেন।

---

# 20. Database-এর Complete Picture

আপনার roadmap-এ ছিল:

```text
PostgreSQL
│
├── Prisma / TypeORM
│
├── Relations
│
├── Transactions
│
├── Indexes
│
├── Constraints
│
├── Migration
│
├── N+1 Problem
│
├── Query Optimization
│
└── Connection Pooling
```

এগুলোর সাথে database section complete করতে আমি আরও এই core topics-গুলো রাখব:

```text
Database
│
├── PostgreSQL fundamentals
│
├── Prisma / TypeORM
│
├── Schema Design
│
├── Data Types
│
├── Primary Key
│
├── Foreign Key
│
├── Relations
│   ├── One-to-One
│   ├── One-to-Many
│   └── Many-to-Many
│
├── Constraints
│   ├── NOT NULL
│   ├── UNIQUE
│   ├── PRIMARY KEY
│   ├── FOREIGN KEY
│   └── CHECK
│
├── Transactions
├── ACID
├── Concurrency
├── Locking
│
├── Indexes
├── Query Optimization
├── EXPLAIN ANALYZE
├── N+1 Problem
├── Pagination
│
├── Connection Pooling
├── Migration
├── Normalization
├── Denormalization
├── Soft Delete
├── Cascade
│
└── Database Security
```

**একজন production-grade NestJS backend developer-এর জন্য এই database section-এর সবচেয়ে গুরুত্বপূর্ণ mental model হলো:**

```text
                    PostgreSQL
                        │
             ┌──────────┴──────────┐
             │                     │
          Schema                 Data
             │                     │
      ┌──────┼──────┐       ┌──────┼──────┐
      │      │      │       │      │      │
   Tables Relations Constraints Indexes Transactions
      │      │      │       │      │
      └──────┴──────┴───────┴──────┘
                     │
                ORM (Prisma)
                     │
                NestJS Service
                     │
                 API Request
```

অর্থাৎ **ORM-এর syntax মুখস্থ করা database শেখা নয়**। আপনাকে বুঝতে হবে database কেন এমনভাবে design করা হচ্ছে, কোন query কতবার যাচ্ছে, data consistency কীভাবে বজায় থাকছে, এবং concurrent request এলে database কীভাবে behave করছে।
