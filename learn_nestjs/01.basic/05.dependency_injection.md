অবশ্যই। এবার **Dependency Injection (DI)** এমনভাবে বুঝি যেন আপনি একদম নতুন। এটা NestJS-এর **core concept**—এটা ভালোভাবে বুঝলে NestJS অনেক সহজ হয়ে যাবে।

# 💉 Dependency Injection কী?

প্রথমে শুধু এই কথাটা মনে রাখুন:

> **Dependency Injection মানে হলো—একটা class-এর যে জিনিসটা দরকার, সেটা class নিজে তৈরি না করে বাইরে থেকে তাকে দিয়ে দেওয়া।**

শুনতে কঠিন লাগছে? 😄
একটা গল্প দিয়ে বুঝি।

---

# 🧑‍🍳 ধরুন আপনি একজন Chef

আপনি একজন Chef।

আপনার Burger বানাতে লাগবে:

```text
🍞 Bread
🥩 Meat
🧀 Cheese
```

আপনি কি প্রতিবার নিজে:

```text
Bread factory বানাবেন
Meat farm বানাবেন
Cheese factory বানাবেন
```

তারপর Burger বানাবেন? 😂

না।

কেউ আপনাকে এগুলো **supply** করবে।

```text
🍞 Bread ─────┐
🥩 Meat ──────┼──→ 👨‍🍳 Chef
🧀 Cheese ────┘
```

Chef শুধু তার কাজ করবে:

```text
Burger বানানো
```

এটাই Dependency Injection-এর basic idea।

---

# 🧠 Dependency মানে কী?

**Dependency = আপনার কাজ করার জন্য যে জিনিসের উপর আপনার নির্ভর করতে হয়।**

ধরুন:

```ts
class Car {
}
```

Car-এর engine দরকার।

```text
🚗 Car
 ↓
🔧 Engine
```

তাহলে `Engine` হলো `Car`-এর dependency।

কারণ Car Engine ছাড়া কাজ করতে পারবে না।

---

# ❌ Dependency নিজে তৈরি করা

ধরুন:

```ts
class Engine {
  start() {
    return 'Engine started';
  }
}

class Car {
  private engine = new Engine();

  drive() {
    this.engine.start();
    return 'Car is driving';
  }
}
```

এখানে `Car` নিজেই:

```ts
new Engine()
```

করছে।

অর্থাৎ:

```text
Car
 │
 └── নিজেই Engine বানাচ্ছে
```

এতে সমস্যা কী?

Car এখন `Engine` তৈরির দায়িত্বও নিয়ে ফেলেছে।

---

# ✅ Dependency Injection

এবার Engine বাইরে থেকে দিয়ে দিই:

```ts
class Engine {
  start() {
    return 'Engine started';
  }
}

class Car {
  constructor(
    private engine: Engine,
  ) {}

  drive() {
    this.engine.start();
    return 'Car is driving';
  }
}
```

এখন:

```text
        Engine
          ↓
       injection
          ↓
         Car
```

Car বলছে:

> "আমার একটা Engine দরকার।"

কিন্তু Car নিজে Engine বানাচ্ছে না।

এটাই **Dependency Injection**।

---

# 🚀 এবার NestJS-এ আসি

NestJS-এ আমরা সাধারণত এমন করি:

```ts
@Injectable()
export class UsersService {

  findAll() {
    return ['Rahim', 'Karim'];
  }

}
```

এখন Controller-এর `UsersService` দরকার।

```ts
@Controller('users')
export class UsersController {

  constructor(
    private usersService: UsersService,
  ) {}

}
```

এখানে:

```ts
UsersService
```

হলো Controller-এর **dependency**।

আর:

```ts
constructor(
  private usersService: UsersService,
) {}
```

এর মাধ্যমে NestJS সেই dependency **inject** করে।

---

# 🔥 NestJS কী করছে?

আপনি লিখলেন:

```ts
constructor(
  private usersService: UsersService,
) {}
```

NestJS বুঝলো:

> "UsersController-এর UsersService দরকার।"

তারপর NestJS:

```text
UsersService তৈরি করে
        ↓
UsersController-এ দেয়
        ↓
Controller সেটা ব্যবহার করে
```

Conceptually এমন:

```ts
const usersService = new UsersService();

const usersController =
  new UsersController(usersService);
```

আপনাকে এগুলো manually করতে হয় না।

**NestJS এগুলো manage করে।**

---

# 🤖 এটাই NestJS-এর magic

আপনি সাধারণত লিখবেন না:

```ts
const usersService = new UsersService();
```

বরং লিখবেন:

```ts
constructor(
  private usersService: UsersService,
) {}
```

NestJS বলবে:

> "ঠিক আছে, আমি `UsersService` তৈরি করে আপনাকে দিয়ে দিচ্ছি।"

---

# 🧩 কিন্তু NestJS জানবে কীভাবে?

এখানে `@Injectable()` এবং Module-এর `providers` গুরুত্বপূর্ণ।

### Service

```ts
@Injectable()
export class UsersService {
  findAll() {
    return ['Rahim', 'Karim'];
  }
}
```

### Module

```ts
@Module({
  controllers: [UsersController],
  providers: [UsersService],
})
export class UsersModule {}
```

এখানে:

```ts
providers: [UsersService]
```

NestJS-কে বলছে:

> "`UsersService` আমাদের dependency container-এর মধ্যে রাখুন এবং প্রয়োজন হলে provide করুন।"

---

# 🧠 Dependency Injection Container

NestJS-এর ভিতরে একটা **container** আছে বলে ভাবতে পারেন।

```text
        NestJS Container
        ┌─────────────────┐
        │                 │
        │ UsersService    │
        │ AuthService     │
        │ EmailService    │
        │ PaymentService  │
        │                 │
        └─────────────────┘
                 │
                 ↓
       প্রয়োজন হলে inject
```

যখন কোনো class বলে:

```ts
constructor(
  private usersService: UsersService,
) {}
```

NestJS container-এর দিকে তাকিয়ে বলে:

> "আমার কাছে UsersService আছে?"

```text
        UsersController
               │
               │ needs
               ↓
        UsersService
               ↑
               │
       NestJS Container
```

তারপর NestJS দিয়ে দেয়।

---

# 🎯 একটা বাস্তব Example

ধরুন আপনার একটা Language Learning App আছে।

আপনার:

```text
AuthController
```

এর `AuthService` দরকার।

### AuthService

```ts
@Injectable()
export class AuthService {

  login(email: string, password: string) {
    // login logic
    return 'Logged in';
  }

}
```

### AuthController

```ts
@Controller('auth')
export class AuthController {

  constructor(
    private readonly authService: AuthService,
  ) {}

  @Post('login')
  login() {
    return this.authService.login(
      'test@gmail.com',
      '123456',
    );
  }
}
```

### AuthModule

```ts
@Module({
  controllers: [AuthController],
  providers: [AuthService],
})
export class AuthModule {}
```

Flow:

```text
POST /auth/login
        ↓
 AuthController
        ↓
 AuthService
        ↓
   Login Logic
        ↓
    Response
```

আর `AuthService` কে Controller-এর কাছে কে দিল?

```text
          🤖 NestJS
             ↓
      Dependency Injection
             ↓
       AuthController
             ↓
        AuthService
```

---

# 🤔 তাহলে `@Injectable()` কেন?

এটা NestJS-কে জানায়:

> "এই class-টাকে dependency হিসেবে ব্যবহার করা যেতে পারে।"

```ts
@Injectable()
export class AuthService {}
```

তারপর Module-এ:

```ts
providers: [AuthService]
```

NestJS এটাকে manage করতে পারে।

---

# 🔥 DI-এর সবচেয়ে বড় সুবিধা

ধরুন আপনার Controller সরাসরি একটা specific database service-এর উপর নির্ভর করছে।

পরে আপনি database system পরিবর্তন করলেন।

DI ব্যবহার করলে architecture অনেক flexible হয়।

উদাহরণ:

```text
Controller
    ↓
 UserService
    ↓
 UserRepository
    ↓
 Database
```

Controller-এর database-এর details জানার দরকার নেই।

Controller শুধু জানে:

> "আমার UserService দরকার।"

Service জানে:

> "User data কীভাবে আনতে হবে।"

---

# 🧪 Testing-এর সময়ও DI অনেক useful

ধরুন আপনি `UsersController` test করছেন।

Real database না ব্যবহার করে fake service দিতে পারেন:

```text
UsersController
      ↓
 FakeUsersService
      ↓
 Test Data
```

তাই DI application-কে:

* maintainable
* testable
* loosely coupled
* reusable

করতে সাহায্য করে।

---

# 🧠 একটি খুব গুরুত্বপূর্ণ শব্দ: Loose Coupling

এটা এখন শুধু basic level-এ বুঝুন।

❌ Tight coupling:

```text
Car
 │
 └── new Engine()
```

Car নিজেই Engine-এর concrete implementation তৈরি করছে।

✅ Loose coupling:

```text
Engine
   ↓
 injected
   ↓
Car
```

Car শুধু বলছে:

> "আমার Engine দরকার।"

কীভাবে Engine তৈরি হবে সেটা Car-এর দায়িত্ব নয়।

---

# ⭐ পুরো NestJS Architecture

এখন আপনার আগের lessons একসাথে মিলিয়ে দেখি:

```text
                    🏠 Module
                       │
          ┌────────────┴────────────┐
          ↓                         ↓
     🎤 Controller              ⚙️ Provider
          │                         │
          │                    UsersService
          │                         │
          └──────────┐   ┌──────────┘
                     ↓   ↓
              💉 Dependency
                 Injection
                     ↓
                NestJS Container
```

আর request flow:

```text
👤 Client
    │
    ↓
🌐 HTTP Request
    │
    ↓
🎤 Controller
    │
    ↓
💉 Injected Service
    │
    ↓
🗄️ Database
```

---

# 📝 ৩০ সেকেন্ডের Cheat Sheet

**Dependency:**
যে জিনিস একটা class-এর কাজ করার জন্য দরকার।

**Injection:**
সেই dependency বাইরে থেকে class-কে দিয়ে দেওয়া।

**Dependency Injection:**
Class নিজে dependency তৈরি না করে, বাইরে থেকে dependency গ্রহণ করে।

**NestJS:**
এই dependency তৈরি, manage এবং inject করার কাজটা automatically করে।

### সবচেয়ে গুরুত্বপূর্ণ code:

```ts
constructor(
  private readonly usersService: UsersService,
) {}
```

এটার অর্থ:

> **"UsersController-এর UsersService দরকার, NestJS আপনি আমাকে সেটা দিয়ে দিন।"**

এবং:

```ts
@Module({
  providers: [UsersService],
})
```

এর অর্থ:

> **"NestJS, UsersService-কে আপনার dependency container-এ manage করুন।"**

---

### 🧩 এখন পর্যন্ত আপনার NestJS foundation

```text
1. Module
      ↓
2. Controller
      ↓
3. Provider / Service
      ↓
4. Dependency Injection
```

এর পরের natural topic হলো **Custom Providers (`useClass`, `useValue`, `useFactory`, `useExisting`)**—এগুলো বুঝলে NestJS-এর DI system আসলে কীভাবে কাজ করে সেটা অনেক গভীরভাবে পরিষ্কার হবে।
