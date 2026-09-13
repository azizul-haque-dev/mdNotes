অবশ্যই। এবার **NestJS → Providers / Services** শিখি। এটা NestJS-এর সবচেয়ে গুরুত্বপূর্ণ concept-গুলোর একটা। আমি একদম সহজভাবে বুঝাচ্ছি।

# ⚙️ Providers / Services কী?

আগের lesson-এ আমরা দেখেছি:

```text
👤 Client
   ↓
🎤 Controller
   ↓
⚙️ Service
   ↓
🗄️ Database
```

এখানে **Controller হলো receptionist**।

আর **Service হলো সেই worker/expert যে আসল কাজটা করে।**

---

## 🧑‍💼 একটা বাস্তব উদাহরণ

ধরুন আপনি একটা restaurant-এ গেলেন।

আপনি বললেন:

> "ভাই, আমাকে একটা Chicken Burger দেন।"

আপনি kitchen-এ গিয়ে burger বানাবেন না। 😄

আপনি:

```text
👤 Customer
   ↓
🧑‍💼 Waiter
   ↓
👨‍🍳 Chef
   ↓
🍔 Burger
```

NestJS-এ:

```text
👤 Client
   ↓
🎤 Controller
   ↓
⚙️ Service
   ↓
🗄️ Database
```

অর্থাৎ:

**Controller → request নেয়**
**Service → আসল কাজ করে**

---

# 1. Service কী?

একটা simple service:

```ts
import { Injectable } from '@nestjs/common';

@Injectable()
export class UsersService {
  getUsers() {
    return ['Rahim', 'Karim', 'Hasan'];
  }
}
```

এখানে:

```ts
@Injectable()
```

NestJS-কে বলছে:

> "এই class-টাকে NestJS manage করতে পারবে।"

এবং:

```ts
export class UsersService
```

হলো আমাদের Service।

---

# 2. Service কেন দরকার?

ধরুন আপনি Controller-এর মধ্যেই সব code লিখলেন:

```ts
@Controller('users')
export class UsersController {

  @Get()
  getUsers() {
    // database query
    // filtering
    // validation
    // business logic
    // calculations
    // etc...
  }
}
```

কিছুদিন পর Controller হয়ে যাবে:

```text
😵 500 lines
😵 1000 lines
😵 বুঝতে কষ্ট
😵 maintain করা কঠিন
```

তাই আমরা বলি:

> Controller শুধু request handle করুক।
> আসল কাজ Service করুক।

---

# 3. Controller + Service

### Service

```ts
@Injectable()
export class UsersService {

  getUsers() {
    return ['Rahim', 'Karim', 'Hasan'];
  }

}
```

### Controller

```ts
@Controller('users')
export class UsersController {

  constructor(
    private readonly usersService: UsersService,
  ) {}

  @Get()
  getUsers() {
    return this.usersService.getUsers();
  }

}
```

এখন flow:

```text
GET /users
     ↓
UsersController
     ↓
usersService.getUsers()
     ↓
['Rahim', 'Karim', 'Hasan']
```

---

# 🧠 কিন্তু একটা প্রশ্ন

এইটা:

```ts
private readonly usersService: UsersService
```

এখানে `usersService` কোথা থেকে এলো?

আমরা তো লিখিনি:

```ts
const usersService = new UsersService();
```

এখানেই আসে NestJS-এর **Dependency Injection (DI)**।

---

# ⭐ Dependency Injection কী?

ভয় পাওয়ার কিছু নেই। 😄

এর সহজ অর্থ:

> **যে জিনিস আপনার দরকার, NestJS সেটা আপনার class-এর কাছে দিয়ে দেবে।**

ধরুন:

```text
Controller বলছে:

"আমার UsersService দরকার।"
              ↓
        NestJS শুনলো
              ↓
"ঠিক আছে, আমি দিয়ে দিচ্ছি।"
```

তাই:

```ts
constructor(
  private readonly usersService: UsersService,
) {}
```

এর অর্থ:

> "আমার `UsersService` দরকার। NestJS, আপনি আমাকে এটা দিন।"

---

# 4. তাহলে `new UsersService()` কেন লিখি না?

Traditional JavaScript/TypeScript-এ আপনি হয়তো করতেন:

```ts
const usersService = new UsersService();
```

কিন্তু NestJS-এ সাধারণত আপনি নিজে object তৈরি করেন না।

NestJS নিজে manage করে:

```text
NestJS
  │
  ├── UsersController
  │
  └── UsersService
```

NestJS জানে:

> UsersController-এর UsersService দরকার।

তাই NestJS সেটা inject করে দেয়।

এটাই:

# 💉 Dependency Injection

---

# 5. Provider কী তাহলে?

এখানে একটা important distinction আছে।

**Service হলো Provider-এর একটা type।**

Provider মানে broadly:

> **যে dependency NestJS নিজে create এবং manage করতে পারে।**

যেমন:

```ts
@Injectable()
export class UsersService {}
```

এটা একটা Provider।

Service:

```ts
UsersService
```

হলো Provider-এর সবচেয়ে common example।

কিন্তু Provider শুধু Service-ই হতে হবে এমন না।

---

# 🧩 Provider-এর উদাহরণ

```text
Providers
│
├── UsersService
├── AuthService
├── EmailService
├── PaymentService
├── FileService
└── NotificationService
```

সবগুলোই Provider হতে পারে।

---

# 6. Module-এ Provider register করতে হয়

আমরা আগে Module-এ দেখেছিলাম:

```ts
@Module({
  controllers: [],
  providers: [],
})
export class UsersModule {}
```

এখন:

```ts
@Module({
  controllers: [UsersController],
  providers: [UsersService],
})
export class UsersModule {}
```

এখানে আমরা NestJS-কে বলছি:

> "UsersModule-এর মধ্যে `UsersService` নামে একটা Provider আছে।"

তারপর Controller-এ:

```ts
constructor(
  private readonly usersService: UsersService,
) {}
```

NestJS বুঝতে পারে:

```text
UsersController
      ↓
needs UsersService
      ↓
UsersModule
      ↓
UsersService
      ↓
Inject করে দিল
```

---

# 7. পুরো Example

### `users.service.ts`

```ts
import { Injectable } from '@nestjs/common';

@Injectable()
export class UsersService {

  getUsers() {
    return [
      'Rahim',
      'Karim',
      'Hasan',
    ];
  }

  getUser(id: string) {
    return `User ${id}`;
  }

}
```

---

### `users.controller.ts`

```ts
import {
  Controller,
  Get,
  Param,
} from '@nestjs/common';

import { UsersService } from './users.service';

@Controller('users')
export class UsersController {

  constructor(
    private readonly usersService: UsersService,
  ) {}

  @Get()
  getUsers() {
    return this.usersService.getUsers();
  }

  @Get(':id')
  getUser(@Param('id') id: string) {
    return this.usersService.getUser(id);
  }

}
```

---

### `users.module.ts`

```ts
import { Module } from '@nestjs/common';

import { UsersController } from './users.controller';
import { UsersService } from './users.service';

@Module({
  controllers: [UsersController],
  providers: [UsersService],
})
export class UsersModule {}
```

---

# 🔥 Request গেলে কী হয়?

আপনি request পাঠালেন:

```http
GET /users/25
```

তখন:

```text
             GET /users/25
                    ↓
            🎤 UsersController
                    ↓
          @Get(':id') চালু
                    ↓
       @Param('id') → "25"
                    ↓
      usersService.getUser("25")
                    ↓
             ⚙️ UsersService
                    ↓
          "User 25"
                    ↓
              👤 Client
```

Controller নিজে user খুঁজছে না।

Controller শুধু বলছে:

> "Service ভাই, ID 25-এর user-এর কাজটা করেন।"

Service কাজ করছে।

---

# 🏗️ Production App-এ এটা আরও সুন্দর

বাস্তবে Service database-এর সাথে কাজ করতে পারে:

```ts
@Injectable()
export class UsersService {

  async getUsers() {
    return this.userModel.find();
  }

  async getUser(id: string) {
    return this.userModel.findById(id);
  }

}
```

তখন architecture:

```text
                    Client
                      ↓
                  Controller
                      ↓
                   Service
                      ↓
                 Repository
                      ↓
                  Database
```

যদিও ছোট project-এ Service সরাসরি database access করতে পারে।

---

# 🧠 একটা জিনিস খুব ভালোভাবে মনে রাখবেন

### Controller:

> **"কেউ কী চাইছে?"**

### Service:

> **"কীভাবে সেটা করতে হবে?"**

উদাহরণ:

```text
GET /users
```

Controller:

> "User list চাওয়া হয়েছে।"

Service:

> "ঠিক আছে, database থেকে user list নিয়ে আসছি।"

---

# ⭐ Module + Controller + Service

এখন পর্যন্ত আমরা তিনটা জিনিস শিখলাম:

```text
              🧩 Module
                 │
        ┌────────┴────────┐
        ↓                 ↓
   🎤 Controller       ⚙️ Service
        │                 │
   Request handle      Business Logic
```

আর:

```text
Module
  │
  ├── Controller
  │
  └── Provider/Service
```

---

# 🎯 Exam-এর মতো করে মনে রাখুন

**Module**
→ Related code group করে।

**Controller**
→ HTTP request/response handle করে।

**Provider**
→ NestJS যেসব dependency manage করে।

**Service**
→ Provider-এর সবচেয়ে common type; সাধারণত business logic রাখে।

**Dependency Injection**
→ NestJS automatically প্রয়োজনীয় dependency class-এর মধ্যে দিয়ে দেয়।

---

## 🧠 সবচেয়ে সহজ analogy

```text
🏢 Company
   │
   └── Module

📞 Receptionist
   │
   └── Controller

👨‍💻 Employee/Expert
   │
   └── Service/Provider

📋 Employee-এর প্রয়োজনীয় tool
   │
   └── Dependency

🤖 Company automatically employee-কে tool দিয়ে দিল
   │
   └── Dependency Injection
```

**পরের গুরুত্বপূর্ণ ধাপ:** `Dependency Injection (DI)` আলাদা করে ভালোভাবে বুঝলে NestJS-এর architecture আপনার মাথায় অনেক পরিষ্কার হয়ে যাবে।
