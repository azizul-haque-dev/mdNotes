অবশ্যই। এবার **NestJS Controllers** একদম ১৫ বছরের একজনকে বোঝানোর মতো করে শিখি। 😄

# 🎮 Controller কী?

আগের উদাহরণে আমরা বলেছিলাম:

> **Module = একটা room/department**

এখন সেই room-এর **receptionist** হলো Controller।

ধরুন আপনার একটা **Food Delivery App** আছে।

User বলল:

> "আমাকে সব খাবারের list দাও।"

Request প্রথমে কোথায় আসবে?

```text
👤 User
   ↓
🌐 HTTP Request
   ↓
🎤 Controller
   ↓
⚙️ Service
   ↓
🗄️ Database
```

অর্থাৎ:

> **Controller-এর প্রধান কাজ হলো HTTP request গ্রহণ করা এবং response দেওয়া।**

---

# 1. Controller তৈরি করি

```ts
import { Controller } from '@nestjs/common';

@Controller('users')
export class UsersController {}
```

এখানে:

```ts
@Controller('users')
```

NestJS-কে বলছে:

> এই Controller-এর URL হবে `/users`

তাই:

```text
/users
```

এই route-এর request এই Controller-এর কাছে আসবে।

---

# 2. কিন্তু শুধু Controller লিখলে হবে?

না।

আমাদের **route handler** লাগবে।

যেমন:

```ts
import { Controller, Get } from '@nestjs/common';

@Controller('users')
export class UsersController {

  @Get()
  getUsers() {
    return 'All Users';
  }

}
```

এখন browser/Postman থেকে:

```http
GET /users
```

request গেলে:

```text
GET /users
     ↓
UsersController
     ↓
getUsers()
     ↓
"All Users"
```

Response:

```text
All Users
```

---

# 🧠 `@Get()` কী?

```ts
@Get()
```

মানে:

> **GET request এলে এই function চালাও।**

যেমন:

```ts
@Get()
getUsers() {
  return 'All Users';
}
```

তাই:

```text
GET /users
      ↓
getUsers()
```

---

# 3. Route কীভাবে তৈরি হচ্ছে?

এখানে:

```ts
@Controller('users')
```

আর:

```ts
@Get()
```

দুটো একসাথে কাজ করছে।

```text
@Controller('users')
       +
     @Get()
       ↓
GET /users
```

---

# 4. `@Get('profile')`

এখন ধরুন আপনি চান:

```http
GET /users/profile
```

তাহলে:

```ts
@Controller('users')
export class UsersController {

  @Get('profile')
  getProfile() {
    return 'My Profile';
  }

}
```

এখন:

```text
GET /users/profile
          ↓
     getProfile()
```

---

# 5. বিভিন্ন HTTP Method

Controller-এ শুধু `GET` নয়, বিভিন্ন HTTP method ব্যবহার করতে পারবেন।

### GET

তথ্য নেওয়ার জন্য:

```ts
@Get()
getUsers() {
  return 'Users';
}
```

### POST

নতুন কিছু তৈরি করার জন্য:

```ts
@Post()
createUser() {
  return 'User Created';
}
```

### PATCH

কিছু update করার জন্য:

```ts
@Patch()
updateUser() {
  return 'User Updated';
}
```

### DELETE

কিছু delete করার জন্য:

```ts
@Delete()
deleteUser() {
  return 'User Deleted';
}
```

তাহলে:

```text
GET     → Data নেওয়া
POST    → নতুন Data তৈরি
PATCH   → Data update
DELETE  → Data delete
```

---

# 6. Real Example

ধরুন User API:

```ts
@Controller('users')
export class UsersController {

  @Get()
  getUsers() {
    return 'All Users';
  }

  @Get('profile')
  getProfile() {
    return 'User Profile';
  }

  @Post()
  createUser() {
    return 'User Created';
  }

  @Patch()
  updateUser() {
    return 'User Updated';
  }

  @Delete()
  deleteUser() {
    return 'User Deleted';
  }

}
```

তাহলে আমাদের API:

```text
GET     /users
GET     /users/profile
POST    /users
PATCH   /users
DELETE  /users
```

---

# 7. URL থেকে ID নেওয়া

এখন ধরুন:

```http
GET /users/123
```

আমরা `123` কীভাবে পাব?

এর জন্য:

```ts
@Get(':id')
getUser(@Param('id') id: string) {
  return `User ID: ${id}`;
}
```

Request:

```http
GET /users/123
```

Response:

```text
User ID: 123
```

এখানে:

```ts
':id'
```

মানে URL-এর এই অংশটা dynamic।

```text
/users/:id
       ↑
    dynamic
```

আর:

```ts
@Param('id')
```

দিয়ে সেই value নেওয়া হয়।

---

# 8. Request Body নেওয়া

ধরুন frontend থেকে পাঠাল:

```json
{
  "name": "Rahim",
  "email": "rahim@example.com"
}
```

POST request:

```http
POST /users
```

Controller:

```ts
@Post()
createUser(@Body() body: any) {
  return body;
}
```

তাহলে:

```text
Frontend
   ↓
POST /users
   ↓
Request Body
   ↓
@Body()
   ↓
Controller
```

এবং response হতে পারে:

```json
{
  "name": "Rahim",
  "email": "rahim@example.com"
}
```

---

# 9. Query Parameter

ধরুন আপনি লিখলেন:

```http
GET /users?name=rahim
```

এখানে:

```text
name=rahim
```

হলো query parameter।

NestJS-এ:

```ts
@Get()
getUsers(@Query('name') name: string) {
  return `Searching for ${name}`;
}
```

Request:

```http
GET /users?name=rahim
```

Response:

```text
Searching for rahim
```

---

# 🧩 এখন Controller-এর পুরো picture

একটা Controller সাধারণত এমন দেখতে পারে:

```ts
import {
  Body,
  Controller,
  Delete,
  Get,
  Param,
  Patch,
  Post,
  Query,
} from '@nestjs/common';

@Controller('users')
export class UsersController {

  @Get()
  getUsers(@Query('name') name: string) {
    return `Users: ${name}`;
  }

  @Get(':id')
  getUser(@Param('id') id: string) {
    return `User: ${id}`;
  }

  @Post()
  createUser(@Body() body: any) {
    return body;
  }

  @Patch(':id')
  updateUser(
    @Param('id') id: string,
    @Body() body: any,
  ) {
    return {
      id,
      ...body,
    };
  }

  @Delete(':id')
  deleteUser(@Param('id') id: string) {
    return `Deleted user ${id}`;
  }
}
```

---

# 🚨 একটা খুব গুরুত্বপূর্ণ বিষয়

শুরুতে Controller-এর ভিতরে আপনি এমন code দেখতে পারেন:

```ts
@Get()
getUsers() {
  // database query
  // business logic
  // validation
  // calculations
  // ...
}
```

কিন্তু **production application-এ Controller-এর মধ্যে সব business logic রাখা ভালো practice নয়।**

Controller-এর কাজ মূলত:

```text
Request
   ↓
Controller
   ↓
Service
   ↓
Database
```

Controller বলবে:

> "Service ভাই, user-এর data নিয়ে আসেন।"

Service কাজটা করবে।

---

# ⭐ Controller + Service

### Controller

```ts
@Controller('users')
export class UsersController {

  constructor(
    private readonly usersService: UsersService,
  ) {}

  @Get()
  getUsers() {
    return this.usersService.findAll();
  }
}
```

### Service

```ts
@Injectable()
export class UsersService {

  findAll() {
    return ['Rahim', 'Karim', 'Hasan'];
  }
}
```

তাহলে:

```text
👤 Client
    ↓
GET /users
    ↓
🎤 UsersController
    ↓
⚙️ UsersService
    ↓
🗄️ Database
```

এখানে Controller শুধু request receive করে Service-কে কাজ করতে দেয়।

---

# 🧠 আজকের Lesson-এর Cheat Sheet

| NestJS          | কাজ                       |
| --------------- | ------------------------- |
| `@Controller()` | Controller তৈরি করে       |
| `@Get()`        | GET request handle করে    |
| `@Post()`       | POST request handle করে   |
| `@Patch()`      | PATCH request handle করে  |
| `@Delete()`     | DELETE request handle করে |
| `@Param()`      | URL parameter নেয়         |
| `@Query()`      | Query parameter নেয়       |
| `@Body()`       | Request body নেয়          |
| `@Headers()`    | Request headers নেয়       |

সবচেয়ে সহজভাবে মনে রাখুন:

```text
              HTTP Request
                   ↓
             🎤 Controller
             ↙     ↓      ↘
          @Get    @Post   @Delete
                   ↓
                Service
                   ↓
                Database
```

**Module = কোন feature-এর code একসাথে থাকবে**
**Controller = HTTP request কোথায় আসবে**
**Service = আসল কাজ/Business Logic করবে**

পরের concept হিসেবে **Providers + Dependency Injection (DI)** শিখলে Controller-এর এই অংশটা—`constructor(private readonly usersService: UsersService)`—একদম পরিষ্কার হয়ে যাবে।
