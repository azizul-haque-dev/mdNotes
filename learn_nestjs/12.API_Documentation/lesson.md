# 14. API Documentation — NestJS + Swagger/OpenAPI

ধরুন আপনি একটা backend বানালেন। আপনার API আছে:

```text
POST /auth/login
POST /auth/refresh
GET  /users/me
GET  /products
POST /products
```

এখন অন্য একজন developer আপনার frontend বানাবে।

সে কীভাবে জানবে:

* কোন endpoint call করবে?
* `POST /auth/login`-এ কী data পাঠাবে?
* response কেমন আসবে?
* কোন endpoint-এ JWT লাগবে?
* কোন field required?
* ভুল data দিলে কী error আসবে?

এই সমস্যার সমাধান হলো **API Documentation**।

---

# 1. Swagger / OpenAPI কী?

দুইটা term আগে আলাদা করি।

### OpenAPI

**OpenAPI হলো API-এর structure describe করার standard।**

এটা বলে:

```text
আমার API-তে কী কী endpoint আছে
কী request নেয়
কী response দেয়
কী authentication লাগে
কী error হতে পারে
```

### Swagger

Swagger হলো OpenAPI specification-এর সাথে কাজ করার tools-এর ecosystem।

NestJS-এ আমরা সাধারণত:

```text
@nestjs/swagger
```

ব্যবহার করি।

এটা দিয়ে automatically interactive API documentation তৈরি করা যায়।

---

# 2. Swagger UI দেখতে কেমন?

ধরুন আপনার API:

```text
POST /auth/login
```

Swagger UI-তে সেটা প্রায় এমন দেখাবে:

```text
AUTH
────────────────────────────

POST /auth/login

Login user

[ Try it out ]

Request body
{
  "email": "user@example.com",
  "password": "password123"
}

[ Execute ]

Response 200
{
  "accessToken": "...",
  "refreshToken": "..."
}
```

অর্থাৎ developer documentation পড়েই API বুঝতে পারবে।

আর সবচেয়ে সুন্দর ব্যাপার হলো Swagger UI থেকেই API test করা যায়।

---

# 3. Swagger Install

NestJS project-এ:

```bash
npm install @nestjs/swagger
```

তারপর `main.ts`-এ Swagger setup করবেন।

```ts
import { NestFactory } from '@nestjs/core';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';

import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const config = new DocumentBuilder()
    .setTitle('My API')
    .setDescription('My application API documentation')
    .setVersion('1.0')
    .build();

  const document = SwaggerModule.createDocument(
    app,
    config,
  );

  SwaggerModule.setup('docs', app, document);

  await app.listen(3000);
}

bootstrap();
```

এখন:

```text
http://localhost:3000/docs
```

open করলে Swagger UI দেখতে পাবেন।

---

# 4. DocumentBuilder

এই অংশ:

```ts
const config = new DocumentBuilder()
  .setTitle('My API')
  .setDescription('My application API documentation')
  .setVersion('1.0')
  .build();
```

Swagger documentation-এর basic information তৈরি করে।

### Title

```ts
.setTitle('My API')
```

Swagger page-এর title।

---

### Description

```ts
.setDescription(
  'My application API documentation',
)
```

API সম্পর্কে description।

---

### Version

```ts
.setVersion('1.0')
```

API version।

---

# 5. Controller Automatically Documentation-এ আসবে?

ধরুন:

```ts
@Controller('products')
export class ProductsController {

  @Get()
  findAll() {
    return [];
  }

}
```

Swagger অনেক information automatically detect করতে পারে।

Swagger-এ দেখা যাবে:

```text
GET /products
```

কিন্তু professional documentation-এর জন্য শুধু endpoint দেখানো যথেষ্ট না।

আমাদের আরও information দিতে হবে।

এর জন্য decorators ব্যবহার করি।

---

# 6. `@ApiTags()`

একই ধরনের API group করার জন্য:

```ts
import { ApiTags } from '@nestjs/swagger';

@ApiTags('Products')
@Controller('products')
export class ProductsController {

  @Get()
  findAll() {
    return [];
  }

}
```

Swagger UI:

```text
Products
  GET /products
```

Auth controller:

```ts
@ApiTags('Authentication')
@Controller('auth')
export class AuthController {}
```

তাহলে:

```text
Authentication
  POST /auth/login
  POST /auth/refresh
```

এতে documentation organized থাকে।

---

# 7. API Summary

ধরুন:

```ts
@Get()
findAll() {
  return [];
}
```

Swagger-এ আরও সুন্দর description দিতে:

```ts
import { ApiOperation } from '@nestjs/swagger';

@ApiOperation({
  summary: 'Get all products',
})
@Get()
findAll() {
  return [];
}
```

Swagger:

```text
GET /products

Get all products
```

আরও detailed description:

```ts
@ApiOperation({
  summary: 'Get all products',
  description:
    'Returns a paginated list of products.',
})
```

---

# 8. DTO Documentation

এটা খুব গুরুত্বপূর্ণ।

ধরুন login DTO:

```ts
export class LoginDto {
  email: string;
  password: string;
}
```

Swagger automatically সবসময় আপনার desired schema বুঝবে না।

তাই:

```ts
import { ApiProperty } from '@nestjs/swagger';

export class LoginDto {
  @ApiProperty({
    example: 'user@example.com',
    description: 'User email address',
  })
  email: string;

  @ApiProperty({
    example: 'Password123!',
    description: 'User password',
  })
  password: string;
}
```

এখন Swagger জানবে:

```text
email
  type: string
  example: user@example.com

password
  type: string
  example: Password123!
```

---

# 9. `@ApiProperty()`

একটা DTO:

```ts
export class CreateProductDto {
  @ApiProperty({
    example: 'Fresh Mango',
  })
  title: string;

  @ApiProperty({
    example: 250,
  })
  price: number;

  @ApiProperty({
    example: 50,
  })
  stock: number;
}
```

Swagger দেখাবে:

```json
{
  "title": "Fresh Mango",
  "price": 250,
  "stock": 50
}
```

এটা অন্য developer-এর জন্য অনেক useful।

---

# 10. Required বনাম Optional

ধরুন:

```ts
title
price
description
```

এর মধ্যে description optional।

DTO:

```ts
import {
  ApiProperty,
  ApiPropertyOptional,
} from '@nestjs/swagger';

export class CreateProductDto {
  @ApiProperty({
    example: 'Fresh Mango',
  })
  title: string;

  @ApiProperty({
    example: 250,
  })
  price: number;

  @ApiPropertyOptional({
    example: 'Fresh Saudi mango',
  })
  description?: string;
}
```

এখন Swagger বুঝবে:

```text
title       required
price       required
description optional
```

---

# 11. DTO + class-validator + Swagger

Production application-এ DTO সাধারণত শুধু documentation-এর জন্য না।

একই DTO-তে validation-ও থাকবে।

```ts
import {
  IsEmail,
  IsString,
  MinLength,
} from 'class-validator';

import {
  ApiProperty,
} from '@nestjs/swagger';

export class LoginDto {
  @ApiProperty({
    example: 'user@example.com',
  })
  @IsEmail()
  email: string;

  @ApiProperty({
    example: 'Password123!',
  })
  @IsString()
  @MinLength(8)
  password: string;
}
```

এখানে:

```text
@ApiProperty()
        ↓
Swagger documentation


@IsEmail()
@MinLength()
        ↓
Request validation
```

একই DTO দুইটা কাজ করছে।

---

# 12. Request Body Document করা

Controller:

```ts
@Post('login')
login(
  @Body() dto: LoginDto,
) {
  return this.authService.login(dto);
}
```

যেহেতু `LoginDto` ব্যবহার করা হয়েছে এবং DTO-তে Swagger decorators আছে, Swagger request body schema তৈরি করতে পারবে।

Swagger UI-তে:

```text
POST /auth/login

Request body

{
  "email": "user@example.com",
  "password": "Password123!"
}
```

---

# 13. Response Documentation

এবার response।

ধরুন login করলে:

```json
{
  "accessToken": "eyJ...",
  "refreshToken": "eyJ...",
  "user": {
    "id": "123",
    "email": "user@example.com"
  }
}
```

একটা response DTO বানাতে পারেন:

```ts
import { ApiProperty } from '@nestjs/swagger';

export class LoginResponseDto {
  @ApiProperty({
    example: 'eyJhbGciOiJIUzI1NiIs...',
  })
  accessToken: string;

  @ApiProperty({
    example: 'eyJhbGciOiJIUzI1NiIs...',
  })
  refreshToken: string;
}
```

তারপর:

```ts
@ApiResponse({
  status: 200,
  description: 'Login successful',
  type: LoginResponseDto,
})
@Post('login')
login(
  @Body() dto: LoginDto,
) {
  return this.authService.login(dto);
}
```

Swagger এখন জানবে:

```text
200 Login successful

Response:
{
  "accessToken": "...",
  "refreshToken": "..."
}
```

---

# 14. `@ApiOkResponse()`

200 response-এর জন্য:

```ts
@ApiOkResponse({
  description: 'User profile',
  type: UserResponseDto,
})
@Get('me')
getMe() {
  return this.usersService.getMe();
}
```

---

# 15. `@ApiCreatedResponse()`

POST request-এ নতুন resource তৈরি হলে সাধারণত 201:

```ts
@ApiCreatedResponse({
  description: 'Product created successfully',
  type: ProductResponseDto,
})
@Post()
create(
  @Body() dto: CreateProductDto,
) {
  return this.productsService.create(dto);
}
```

Swagger:

```text
201
Product created successfully
```

---

# 16. Error Documentation

শুধু success response document করলে documentation incomplete।

ধরুন login API-তে:

```text
200 → Login successful
401 → Invalid credentials
422 → Validation error
```

Swagger-এ document করতে পারেন:

```ts
@ApiOkResponse({
  description: 'Login successful',
  type: LoginResponseDto,
})
@ApiUnauthorizedResponse({
  description: 'Invalid email or password',
})
@ApiBadRequestResponse({
  description: 'Invalid request data',
})
@Post('login')
login(
  @Body() dto: LoginDto,
) {
  return this.authService.login(dto);
}
```

এখন অন্য developer আগে থেকেই জানবে:

```text
200
→ login successful

400
→ request invalid

401
→ credentials invalid
```

---

# 17. Common HTTP Errors

যেমন product API:

```ts
@ApiNotFoundResponse({
  description: 'Product not found',
})
@ApiBadRequestResponse({
  description: 'Invalid product data',
})
@ApiUnauthorizedResponse({
  description: 'Authentication required',
})
@ApiForbiddenResponse({
  description: 'You do not have permission',
})
```

Documentation:

```text
GET /products/:id

200 → Product found
400 → Invalid ID
401 → Authentication required
403 → Forbidden
404 → Product not found
```

---

# 18. Authentication Document করা

এটা খুব গুরুত্বপূর্ণ।

ধরুন আপনার API JWT authentication ব্যবহার করে।

Request:

```http
GET /users/me
Authorization: Bearer <access_token>
```

Swagger-কে জানাতে হবে:

> এই API JWT Bearer token চায়।

প্রথমে Swagger config:

```ts
const config = new DocumentBuilder()
  .setTitle('My API')
  .setDescription('My application API')
  .setVersion('1.0')
  .addBearerAuth()
  .build();
```

তারপর protected controller/route:

```ts
@ApiBearerAuth()
@Get('me')
getMe() {
  return this.usersService.getMe();
}
```

Swagger UI-তে এখন:

```text
Authorize 🔒
```

button দেখতে পাবেন।

---

# 19. Swagger Authorize

User:

```text
Authorize
```

button click করবে।

তারপর token:

```text
eyJhbGciOiJIUzI1Ni...
```

দেবে।

Swagger subsequent request-এ:

```http
Authorization: Bearer eyJ...
```

পাঠাবে।

তাই Swagger থেকেই protected API test করা যায়।

---

# 20. পুরো Auth Controller Example

এখন আপনার auth API-এর একটা complete example:

```ts
import {
  Body,
  Controller,
  Post,
} from '@nestjs/common';

import {
  ApiBadRequestResponse,
  ApiBearerAuth,
  ApiOkResponse,
  ApiOperation,
  ApiTags,
  ApiUnauthorizedResponse,
} from '@nestjs/swagger';

@ApiTags('Authentication')
@Controller('auth')
export class AuthController {

  @ApiOperation({
    summary: 'Login user',
  })
  @ApiOkResponse({
    description: 'Login successful',
    type: LoginResponseDto,
  })
  @ApiBadRequestResponse({
    description: 'Invalid request data',
  })
  @ApiUnauthorizedResponse({
    description: 'Invalid credentials',
  })
  @Post('login')
  login(
    @Body() dto: LoginDto,
  ) {
    return this.authService.login(dto);
  }


  @ApiOperation({
    summary: 'Refresh access token',
  })
  @ApiOkResponse({
    description: 'New access token generated',
  })
  @Post('refresh')
  refresh(
    @Body() dto: RefreshTokenDto,
  ) {
    return this.authService.refresh(dto);
  }
}
```

Swagger structure:

```text
Authentication
│
├── POST /auth/login
│   ├── Request
│   ├── Response 200
│   ├── Error 400
│   └── Error 401
│
└── POST /auth/refresh
    ├── Request
    └── Response 200
```

---

# 21. Users API Example

```ts
@ApiTags('Users')
@Controller('users')
export class UsersController {

  @ApiOperation({
    summary: 'Get current user',
  })
  @ApiBearerAuth()
  @ApiOkResponse({
    description: 'Current user profile',
    type: UserResponseDto,
  })
  @ApiUnauthorizedResponse({
    description: 'Authentication required',
  })
  @Get('me')
  getMe() {
    return this.usersService.getMe();
  }
}
```

Documentation:

```text
Users

GET /users/me

🔒 Bearer Authentication

200
{
  "id": "123",
  "email": "user@example.com",
  "role": "USER"
}

401
Authentication required
```

---

# 22. Products API Example

```ts
@ApiTags('Products')
@Controller('products')
export class ProductsController {

  @ApiOperation({
    summary: 'Get all products',
  })
  @ApiOkResponse({
    description: 'Products retrieved successfully',
    type: [ProductResponseDto],
  })
  @Get()
  findAll() {
    return this.productsService.findAll();
  }


  @ApiOperation({
    summary: 'Create product',
  })
  @ApiBearerAuth()
  @ApiCreatedResponse({
    description: 'Product created successfully',
    type: ProductResponseDto,
  })
  @ApiBadRequestResponse({
    description: 'Invalid product data',
  })
  @ApiUnauthorizedResponse({
    description: 'Authentication required',
  })
  @Post()
  create(
    @Body() dto: CreateProductDto,
  ) {
    return this.productsService.create(dto);
  }
}
```

Swagger:

```text
Products
│
├── GET /products
│
└── POST /products
      🔒 Bearer Auth
```

---

# 23. Query Parameters Document করা

ধরুন:

```http
GET /products?page=1&limit=20&search=mango
```

DTO:

```ts
export class ProductQueryDto {

  @ApiPropertyOptional({
    example: 1,
  })
  page?: number;

  @ApiPropertyOptional({
    example: 20,
  })
  limit?: number;

  @ApiPropertyOptional({
    example: 'mango',
  })
  search?: string;
}
```

Controller:

```ts
@Get()
findAll(
  @Query() query: ProductQueryDto,
) {
  return this.productsService.findAll(query);
}
```

Swagger বুঝবে:

```text
GET /products

Query Parameters:

page
limit
search
```

---

# 24. Enum Document করা

ধরুন product category:

```ts
enum ProductCategory {
  FRUIT = 'FRUIT',
  VEGETABLE = 'VEGETABLE',
  GRAIN = 'GRAIN',
}
```

DTO:

```ts
@ApiProperty({
  enum: ProductCategory,
  example: ProductCategory.FRUIT,
})
category: ProductCategory;
```

Swagger dropdown-এর মতো দেখাতে পারবে:

```text
category

FRUIT
VEGETABLE
GRAIN
```

এটা developer-এর জন্য খুব useful।

---

# 25. Swagger + DTO = Contract

এই concept-টা খুব ভালোভাবে বুঝুন।

Frontend developer এবং backend developer-এর মধ্যে একটা agreement দরকার।

যেমন backend বলল:

```json
POST /products
```

Request:

```json
{
  "title": "Mango",
  "price": 250,
  "stock": 20
}
```

Response:

```json
{
  "id": "123",
  "title": "Mango",
  "price": 250,
  "stock": 20
}
```

এটাই API contract।

Swagger সেই contract-টা visible করে।

```text
Backend
   │
   │ OpenAPI documentation
   ▼
Swagger
   │
   ▼
Frontend Developer
```

---

# 26. আপনার API Documentation-এর Final Structure

আপনার production NestJS API হলে Swagger-এ ideally এমন structure থাকবে:

```text
API Documentation
│
├── Authentication
│   ├── POST /auth/register
│   ├── POST /auth/login
│   ├── POST /auth/refresh
│   ├── POST /auth/forgot-password
│   └── POST /auth/reset-password
│
├── Users
│   ├── GET /users/me
│   └── PATCH /users/me
│
├── Products
│   ├── GET /products
│   ├── GET /products/:id
│   ├── POST /products
│   ├── PATCH /products/:id
│   └── DELETE /products/:id
│
└── ...
```

প্রতিটি endpoint-এর মধ্যে:

```text
Endpoint
   │
   ├── Method
   ├── Description
   ├── Authentication
   ├── Parameters
   ├── Request DTO
   ├── Request example
   ├── Success response
   ├── Response DTO
   └── Error responses
```

---

# 27. পুরো Concept একবারে

একটা API:

```text
POST /products
```

এর documentation হতে পারে:

```text
POST /products
────────────────────────────

Summary:
Create a new product

Authentication:
Bearer JWT 🔒

Request Body:
CreateProductDto

{
  "title": "Fresh Mango",
  "price": 250,
  "stock": 50
}

201 Created:

{
  "id": "123",
  "title": "Fresh Mango",
  "price": 250,
  "stock": 50
}

400 Bad Request:
Invalid product data

401 Unauthorized:
Authentication required

403 Forbidden:
Insufficient permission
```

এই একটা documentation দেখেই frontend developer বুঝতে পারবে API কীভাবে ব্যবহার করতে হবে।

---

## মনে রাখার সহজ formula

```text
OpenAPI
   ↓
API-এর standard description


Swagger
   ↓
OpenAPI documentation + UI


NestJS
   ↓
@nestjs/swagger
   ↓
Decorators
   ↓
Swagger UI
```

আর সবচেয়ে গুরুত্বপূর্ণ decorators:

```text
@ApiTags()
        ↓
API group


@ApiOperation()
        ↓
API description


@ApiProperty()
        ↓
DTO field documentation


@ApiPropertyOptional()
        ↓
Optional DTO field


@ApiBearerAuth()
        ↓
JWT authentication


@ApiOkResponse()
        ↓
200 response


@ApiCreatedResponse()
        ↓
201 response


@ApiBadRequestResponse()
        ↓
400 error


@ApiUnauthorizedResponse()
        ↓
401 error


@ApiForbiddenResponse()
        ↓
403 error


@ApiNotFoundResponse()
        ↓
404 error
```

অর্থাৎ **Swagger-এর আসল লক্ষ্য শুধু সুন্দর UI বানানো না**—আপনার backend API-এর **request, response, authentication, DTO এবং errors-এর একটা পরিষ্কার, machine-readable এবং developer-friendly contract তৈরি করা।**
