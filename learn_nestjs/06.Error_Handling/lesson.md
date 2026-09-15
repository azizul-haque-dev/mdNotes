# 7. Error Handling — NestJS

Production application-এ **error handling** খুবই গুরুত্বপূর্ণ।

একজন user যখন API call করে, তখন backend-এর কাজ শুধু success response দেওয়া না। কোনো সমস্যা হলে **কী সমস্যা হয়েছে, কেন হয়েছে এবং client কীভাবে সেটা handle করবে**—এটাও properly define করতে হয়।

ধরুন:

```text
Client
   │
   │ POST /auth/login
   ▼
NestJS
   │
   ├── Success → 200
   │
   └── Error
        ├── 400 Bad Request
        ├── 401 Unauthorized
        ├── 403 Forbidden
        ├── 404 Not Found
        ├── 409 Conflict
        └── 500 Internal Server Error
```

আমাদের লক্ষ্য:

```text
❌ random error
❌ raw database error
❌ stack trace user-কে দেওয়া

        ↓

✅ consistent error
✅ predictable error code
✅ proper HTTP status
✅ safe error message
```

---

# 1. Built-in Exceptions

NestJS-এ অনেক built-in HTTP exception আছে।

সবচেয়ে common:

```ts
BadRequestException
UnauthorizedException
ForbiddenException
NotFoundException
ConflictException
UnprocessableEntityException
InternalServerErrorException
```

এগুলো `@nestjs/common` থেকে আসে।

---

## `NotFoundException`

ধরুন user ID `123` পাওয়া গেল না।

```ts
import { NotFoundException } from '@nestjs/common';

throw new NotFoundException('User not found');
```

Response সাধারণত এরকম হবে:

```json
{
  "statusCode": 404,
  "message": "User not found",
  "error": "Not Found"
}
```

HTTP status:

```text
404 Not Found
```

---

# 2. `BadRequestException`

Client ভুল request পাঠিয়েছে।

```ts
throw new BadRequestException(
  'Invalid request',
);
```

Response:

```json
{
  "statusCode": 400,
  "message": "Invalid request",
  "error": "Bad Request"
}
```

যেমন:

```text
age = "hello"
```

যেখানে number প্রয়োজন।

---

# 3. `UnauthorizedException`

User authenticated নয়।

```ts
throw new UnauthorizedException(
  'Invalid credentials',
);
```

HTTP:

```text
401 Unauthorized
```

Login-এর ক্ষেত্রে:

```ts
if (!user) {
  throw new UnauthorizedException(
    'Invalid email or password',
  );
}
```

---

# 4. `ForbiddenException`

User login করা আছে, কিন্তু permission নেই।

```ts
throw new ForbiddenException(
  'You do not have permission to perform this action',
);
```

HTTP:

```text
403 Forbidden
```

উদাহরণ:

```text
User
 │
 └── product:delete ❌
```

---

# 5. `ConflictException`

Resource-এর সাথে conflict হয়েছে।

যেমন email already exists:

```ts
throw new ConflictException(
  'Email already exists',
);
```

HTTP:

```text
409 Conflict
```

---

# 6. Custom Exceptions

Built-in exception ব্যবহার করা ভালো, কিন্তু production application-এ শুধু:

```ts
'Email already exists'
```

দিলে client-এর জন্য consistent error handling কঠিন হতে পারে।

আমরা চাই:

```json
{
  "statusCode": 409,
  "code": "EMAIL_ALREADY_EXISTS",
  "message": "An account with this email already exists"
}
```

এখানে:

```text
code = EMAIL_ALREADY_EXISTS
```

হলো machine-readable error code।

---

# 7. Error Code কেন দরকার?

ধরুন frontend শুধু message দেখে:

```ts
if (error.message === 'Email already exists') {
  // show something
}
```

এটা fragile।

কারণ backend message পরিবর্তন করতে পারে:

```text
"Email already exists"
```

থেকে:

```text
"This email is already registered"
```

তখন frontend-এর logic ভেঙে যাবে।

কিন্তু:

```json
{
  "code": "EMAIL_ALREADY_EXISTS"
}
```

সবসময় একই রাখা যায়।

Frontend:

```ts
if (error.code === 'EMAIL_ALREADY_EXISTS') {
  // show email already exists UI
}
```

---

# 8. Custom Exception Class

আমরা একটা base exception তৈরি করতে পারি।

```ts
import { ConflictException } from '@nestjs/common';

export class EmailAlreadyExistsException
  extends ConflictException {
  constructor() {
    super({
      code: 'EMAIL_ALREADY_EXISTS',
      message: 'Email already exists',
    });
  }
}
```

তারপর service:

```ts
if (existingUser) {
  throw new EmailAlreadyExistsException();
}
```

Response:

```json
{
  "statusCode": 409,
  "code": "EMAIL_ALREADY_EXISTS",
  "message": "Email already exists"
}
```

এটা অনেক cleaner।

---

# 9. Custom Exception Structure

একটা production application-এ বিভিন্ন custom exception থাকতে পারে:

```text
exceptions/
├── user-not-found.exception.ts
├── email-already-exists.exception.ts
├── invalid-token.exception.ts
└── insufficient-permission.exception.ts
```

যেমন:

```ts
export class UserNotFoundException
  extends NotFoundException {
  constructor() {
    super({
      code: 'USER_NOT_FOUND',
      message: 'User not found',
    });
  }
}
```

তারপর:

```ts
throw new UserNotFoundException();
```

---

# 10. Exception Filters

এখন একটা important প্রশ্ন:

> সব জায়গায় error response manually format করব?

ধরুন 100টা service আছে।

প্রতিটি জায়গায়:

```ts
try {
  ...
} catch (error) {
  return {
    statusCode: ...,
    code: ...,
    message: ...,
  };
}
```

করলে code messy হয়ে যাবে।

এখানে আসে:

# Exception Filter

Exception filter হলো এমন একটি layer যেটা thrown exception ধরে এবং response-কে control করে।

```text
Controller / Service
        │
        │ throw error
        ▼
Exception Filter
        │
        ▼
HTTP Response
```

---

# 11. Global Exception Filter

সব controller-এর জন্য একই error handling চাইলে:

```text
Global Exception Filter
```

ব্যবহার করা হয়।

ধরুন:

```text
/auth
/users
/products
/orders
/payments
```

সব জায়গার error এক জায়গা থেকে handle হবে।

---

# 12. Basic Global Exception Filter

```ts
import {
  ArgumentsHost,
  Catch,
  ExceptionFilter,
  HttpException,
} from '@nestjs/common';

@Catch()
export class GlobalExceptionFilter
  implements ExceptionFilter
{
  catch(
    exception: unknown,
    host: ArgumentsHost,
  ) {
    const ctx = host.switchToHttp();

    const response = ctx.getResponse();

    if (exception instanceof HttpException) {
      const status = exception.getStatus();

      response.status(status).json({
        statusCode: status,
        message: exception.message,
      });

      return;
    }

    response.status(500).json({
      statusCode: 500,
      message: 'Internal server error',
    });
  }
}
```

এখন এই filter সব exception catch করতে পারবে।

---

# 13. Global Filter Register করা

`main.ts`:

```ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.useGlobalFilters(
    new GlobalExceptionFilter(),
  );

  await app.listen(3000);
}

bootstrap();
```

এখন:

```ts
throw new NotFoundException();
```

থেকে শুরু করে unexpected error পর্যন্ত global filter-এর মধ্য দিয়ে যাবে।

---

# 14. Consistent Error Response

Production API-তে সবচেয়ে important বিষয় হলো:

> সব error response predictable হওয়া।

ধরুন আমরা এই structure রাখলাম:

```json
{
  "success": false,
  "statusCode": 404,
  "code": "USER_NOT_FOUND",
  "message": "User not found",
  "path": "/users/123",
  "timestamp": "2026-09-15T05:30:00.000Z"
}
```

এখন frontend জানে:

```text
success
statusCode
code
message
```

কোথায় থাকবে।

---

# 15. Better Global Exception Filter

একটা practical version:

```ts
import {
  ArgumentsHost,
  Catch,
  ExceptionFilter,
  HttpException,
  HttpStatus,
} from '@nestjs/common';

import { Request, Response } from 'express';

@Catch()
export class GlobalExceptionFilter
  implements ExceptionFilter
{
  catch(
    exception: unknown,
    host: ArgumentsHost,
  ) {
    const ctx = host.switchToHttp();

    const request =
      ctx.getRequest<Request>();

    const response =
      ctx.getResponse<Response>();

    let statusCode =
      HttpStatus.INTERNAL_SERVER_ERROR;

    let code = 'INTERNAL_SERVER_ERROR';

    let message = 'Internal server error';

    if (exception instanceof HttpException) {
      statusCode = exception.getStatus();

      const exceptionResponse =
        exception.getResponse();

      if (
        typeof exceptionResponse === 'object' &&
        exceptionResponse !== null
      ) {
        const error =
          exceptionResponse as Record<
            string,
            unknown
          >;

        code =
          typeof error.code === 'string'
            ? error.code
            : code;

        message =
          typeof error.message === 'string'
            ? error.message
            : message;
      }
    }

    response.status(statusCode).json({
      success: false,
      statusCode,
      code,
      message,
      path: request.url,
      timestamp: new Date().toISOString(),
    });
  }
}
```

এখন response:

```json
{
  "success": false,
  "statusCode": 404,
  "code": "USER_NOT_FOUND",
  "message": "User not found",
  "path": "/users/123",
  "timestamp": "2026-09-15T05:30:00.000Z"
}
```

---

# 16. Validation Errors

NestJS-এ DTO validation করলে validation error আসবে।

ধরুন:

```ts
export class RegisterDto {
  @IsEmail()
  email: string;

  @MinLength(8)
  password: string;
}
```

User পাঠালো:

```json
{
  "email": "hello",
  "password": "123"
}
```

Validation fail করবে।

`ValidationPipe`:

```ts
app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,
    transform: true,
  }),
);
```

এতে validation error generate হবে।

---

# 17. Validation Error Response

Default response কিছুটা এমন হতে পারে:

```json
{
  "statusCode": 400,
  "message": [
    "email must be an email",
    "password must be longer than or equal to 8 characters"
  ],
  "error": "Bad Request"
}
```

কিন্তু production API-তে আমরা এটাকে consistent করতে পারি।

যেমন:

```json
{
  "success": false,
  "statusCode": 400,
  "code": "VALIDATION_ERROR",
  "message": "Request validation failed",
  "errors": [
    {
      "field": "email",
      "messages": [
        "email must be an email"
      ]
    },
    {
      "field": "password",
      "messages": [
        "password must be longer than or equal to 8 characters"
      ]
    }
  ]
}
```

এটা frontend-এর জন্য অনেক useful।

---

# 18. Validation Error আলাদা কেন?

কারণ:

```text
400
```

হল generic HTTP status।

কিন্তু:

```text
VALIDATION_ERROR
```

হলো application-level error code।

Frontend তখন সহজে বুঝতে পারে:

```text
400
 +
VALIDATION_ERROR
```

মানে user-এর input ঠিক করতে হবে।

---

# 19. Database Errors

এবার সবচেয়ে important অংশগুলোর একটি।

ধরুন database-এ:

```text
email UNIQUE
```

কিন্তু user একই email দিয়ে আবার register করল।

Database error দিতে পারে।

Prisma ব্যবহার করলে known database error পাওয়া যায়।

উদাহরণ:

```ts
try {
  await this.prisma.user.create({
    data: {
      email,
      password,
    },
  });
} catch (error) {
  // database error handling
}
```

Prisma-এর specific error type check করা যায়:

```ts
import { Prisma } from '@prisma/client';

try {
  await this.prisma.user.create({
    data: {
      email,
      password,
    },
  });
} catch (error) {
  if (
    error instanceof Prisma.PrismaClientKnownRequestError
  ) {
    if (error.code === 'P2002') {
      throw new EmailAlreadyExistsException();
    }
  }

  throw error;
}
```

এখানে:

```text
Database
   ↓
Unique constraint violation
   ↓
P2002
   ↓
EMAIL_ALREADY_EXISTS
   ↓
409 Conflict
```

User raw database error দেখবে না।

---

# 20. Raw Database Error কেন দেওয়া যাবে না?

ধরুন database error:

```text
Unique constraint failed on the fields:
(`email`)
```

এটা user-এর জন্য useful না।

আর কিছু database error internal information expose করতে পারে।

❌:

```json
{
  "error": "PrismaClientKnownRequestError...",
  "database": "...",
  "stack": "..."
}
```

Production response-এ এসব দেওয়া উচিত না।

বরং:

```json
{
  "success": false,
  "statusCode": 409,
  "code": "EMAIL_ALREADY_EXISTS",
  "message": "Email already exists"
}
```

---

# 21. Unexpected Errors

সব error আপনি আগে থেকে জানবেন না।

যেমন:

```ts
const user = await getUser();

user.profile.name.toUpperCase();
```

কোনো unexpected condition-এর কারণে:

```text
TypeError
```

হতে পারে।

এক্ষেত্রে:

```text
500 Internal Server Error
```

দেওয়া হবে।

Response:

```json
{
  "success": false,
  "statusCode": 500,
  "code": "INTERNAL_SERVER_ERROR",
  "message": "Internal server error"
}
```

User-কে:

```text
TypeError: Cannot read properties of undefined
```

দেওয়া উচিত না।

---

# 22. কিন্তু Developer কীভাবে জানবে?

এখানে application logging গুরুত্বপূর্ণ।

User দেখবে:

```json
{
  "success": false,
  "statusCode": 500,
  "code": "INTERNAL_SERVER_ERROR",
  "message": "Internal server error"
}
```

কিন্তু server log-এ থাকবে:

```text
ERROR
Request: POST /users
Error: TypeError...
Stack: ...
```

অর্থাৎ:

```text
User
 ↓
Safe error response

Developer
 ↓
Detailed logs
```

---

# 23. Error Code Design

আপনার application-এর error codes predictable হওয়া উচিত।

Authentication:

```text
INVALID_CREDENTIALS
INVALID_TOKEN
TOKEN_EXPIRED
REFRESH_TOKEN_INVALID
```

User:

```text
USER_NOT_FOUND
EMAIL_ALREADY_EXISTS
USER_ALREADY_VERIFIED
```

Authorization:

```text
INSUFFICIENT_PERMISSION
FORBIDDEN
```

Product:

```text
PRODUCT_NOT_FOUND
PRODUCT_OUT_OF_STOCK
```

Validation:

```text
VALIDATION_ERROR
```

System:

```text
INTERNAL_SERVER_ERROR
DATABASE_ERROR
```

---

# 24. HTTP Status বনাম Error Code

এই দুইটা একই জিনিস না।

উদাহরণ:

```text
HTTP Status
    ↓
409 Conflict
```

Application error:

```text
EMAIL_ALREADY_EXISTS
```

একসাথে:

```json
{
  "success": false,
  "statusCode": 409,
  "code": "EMAIL_ALREADY_EXISTS",
  "message": "Email already exists"
}
```

আরেকটি:

```json
{
  "success": false,
  "statusCode": 404,
  "code": "USER_NOT_FOUND",
  "message": "User not found"
}
```

HTTP status broad category দেয়।

Error code exact business problem বলে।

---

# 25. একটি Real Flow

ধরুন:

```http
POST /auth/register
```

User পাঠালো:

```json
{
  "email": "aziz@gmail.com",
  "password": "12345678"
}
```

Service:

```text
Register
   │
   ▼
Find existing user
   │
   ├── Not found
   │      ↓
   │   Create user
   │
   └── Found
          ↓
   EMAIL_ALREADY_EXISTS
```

Exception:

```ts
throw new EmailAlreadyExistsException();
```

তারপর:

```text
Exception
    ↓
Global Exception Filter
    ↓
HTTP 409
    ↓
JSON response
```

Response:

```json
{
  "success": false,
  "statusCode": 409,
  "code": "EMAIL_ALREADY_EXISTS",
  "message": "Email already exists",
  "path": "/auth/register",
  "timestamp": "2026-09-15T05:30:00.000Z"
}
```

---

# 26. পুরো Error Architecture

Production NestJS application-এ structure এমন হতে পারে:

```text
                    Request
                       │
                       ▼
                  Controller
                       │
                       ▼
                    Service
                       │
          ┌────────────┼────────────┐
          │            │            │
       Success     Known Error   Unknown Error
          │            │            │
          │            ▼            ▼
          │       Custom/HTTP    Global Filter
          │       Exception          │
          │            │             │
          └────────────┼─────────────┘
                       ▼
                Error Response
                       │
                       ▼
              Client / Frontend
```

---

# 27. Recommended Error Structure

আপনার NestJS API-এর জন্য এই ধরনের structure খুব ভালো:

```json
{
  "success": false,
  "statusCode": 404,
  "code": "USER_NOT_FOUND",
  "message": "User not found",
  "path": "/users/123",
  "timestamp": "2026-09-15T05:30:00.000Z"
}
```

Validation-এর ক্ষেত্রে:

```json
{
  "success": false,
  "statusCode": 400,
  "code": "VALIDATION_ERROR",
  "message": "Request validation failed",
  "errors": [
    {
      "field": "email",
      "messages": [
        "Invalid email address"
      ]
    }
  ],
  "path": "/auth/register",
  "timestamp": "2026-09-15T05:30:00.000Z"
}
```

---

# 28. শেষবার পুরো বিষয়টা একসাথে

```text
NestJS Error Handling
│
├── Built-in Exceptions
│   ├── BadRequestException
│   ├── UnauthorizedException
│   ├── ForbiddenException
│   ├── NotFoundException
│   └── ConflictException
│
├── Custom Exceptions
│   ├── UserNotFoundException
│   ├── EmailAlreadyExistsException
│   └── InvalidTokenException
│
├── Exception Filters
│   └── Global Exception Filter
│
├── Error Codes
│   ├── USER_NOT_FOUND
│   ├── EMAIL_ALREADY_EXISTS
│   ├── INVALID_TOKEN
│   └── INSUFFICIENT_PERMISSION
│
├── Validation Errors
│   └── VALIDATION_ERROR
│
├── Database Errors
│   └── Map DB error → Application error
│
└── Unexpected Errors
    ├── Return safe 500
    └── Log detailed error internally
```

সবচেয়ে গুরুত্বপূর্ণ mental model:

```text
                    ERROR
                      │
          ┌───────────┴───────────┐
          │                       │
       Expected                Unexpected
          │                       │
          ▼                       ▼
   Known Exception          Internal Error
          │                       │
          ▼                       ▼
   Meaningful code            Generic 500
          │                       │
          └───────────┬───────────┘
                      ▼
             Global Exception
                  Filter
                      │
                      ▼
             Consistent JSON
                      │
                      ▼
                   Client
```

**মূল লক্ষ্য হলো:** backend-এর ভেতরে যত complicated error-ই হোক, client যেন একটি **safe, predictable এবং consistent error response** পায়।
