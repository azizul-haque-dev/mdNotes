# 6. Configuration Management

Production-grade NestJS application-এ **Configuration Management** খুব important।

সহজভাবে বললে:

> Application-এর যেসব value environment অনুযায়ী পরিবর্তন হয়, সেগুলো code-এর ভিতরে না রেখে বাইরে থেকে safely manage করাই Configuration Management।

উদাহরণ:

```env
DATABASE_URL=
JWT_SECRET=
JWT_EXPIRES_IN=
REDIS_URL=
AWS_ACCESS_KEY=
```

একটা application আপনার laptop-এ যেভাবে চলবে, production server-এ ঠিক একই configuration থাকবে না।

তাই আমরা configuration আলাদা করি।

---

# 1. কেন Configuration Management দরকার?

ধরুন আপনি code-এর মধ্যে লিখলেন:

```ts
const jwtSecret = "my-super-secret-password";
```

এটা খারাপ।

কারণ:

```text
Code
 │
 ├── GitHub
 ├── Developer
 ├── CI/CD
 └── Production
```

সব জায়গায় secret চলে যেতে পারে।

তার পরিবর্তে:

```text
Code
   │
   │ reads
   ▼
Environment
   │
   ├── DATABASE_URL
   ├── JWT_SECRET
   └── REDIS_URL
```

Code-এর মধ্যে secret থাকবে না।

---

# 2. `.env` কী?

`.env` হলো environment variables রাখার একটি সাধারণ file।

ধরুন:

```env
NODE_ENV=development

PORT=3000

DATABASE_URL=postgresql://postgres:password@localhost:5432/myapp

JWT_SECRET=super-secret-key

JWT_EXPIRES_IN=15m

REDIS_URL=redis://localhost:6379
```

আপনার application এই values ব্যবহার করবে।

---

# 3. NestJS ConfigModule

NestJS-এ configuration manage করার জন্য সাধারণত:

```bash
npm install @nestjs/config
```

তারপর:

```ts
import { ConfigModule } from '@nestjs/config';

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
    }),
  ],
})
export class AppModule {}
```

এখানে:

```ts
isGlobal: true
```

দেওয়ার অর্থ হলো অন্য module-এ বারবার `ConfigModule` import করতে হবে না।

---

# 4. ConfigService

Environment variable সরাসরি:

```ts
process.env.JWT_SECRET
```

দিয়ে access করা যায়।

কিন্তু NestJS application-এ production code-এর জন্য `ConfigService` ব্যবহার করা বেশি clean।

```ts
import { ConfigService } from '@nestjs/config';

@Injectable()
export class AuthService {
  constructor(
    private readonly configService: ConfigService,
  ) {}

  getJwtSecret() {
    return this.configService.get<string>('JWT_SECRET');
  }
}
```

এখন:

```ts
this.configService.get<string>('JWT_SECRET');
```

`.env` থেকে value নিয়ে আসবে।

---

# 5. `getOrThrow()` কেন ভালো?

ধরুন:

```env
JWT_SECRET=
```

অথবা variable-টাই নেই।

আপনি যদি:

```ts
configService.get<string>('JWT_SECRET');
```

করেন, তাহলে `undefined` পাওয়া যেতে পারে।

কিন্তু JWT secret ছাড়া application চলা উচিত না।

তাই required configuration-এর ক্ষেত্রে:

```ts
const jwtSecret =
  configService.getOrThrow<string>('JWT_SECRET');
```

এটা ব্যবহার করা ভালো।

যদি value না থাকে, application configuration error দিয়ে fail করবে।

এটা production-এ ভালো behavior।

---

# 6. Environment Validation

এটা খুব গুরুত্বপূর্ণ।

ধরুন আপনার `.env`:

```env
PORT=hello
JWT_EXPIRES_IN=
DATABASE_URL=
```

Application start হয়ে গেল।

তারপর runtime-এ সমস্যা:

```text
Application started
       ↓
Request আসে
       ↓
Database connect ❌
```

এটা ভালো না।

আমরা চাই application **startup-এর সময়ই configuration validate করুক**।

```text
Application starts
       ↓
Load environment
       ↓
Validate
       ↓
Valid?
  ├── YES → Start application
  │
  └── NO → Stop application
```

---

# 7. Joi দিয়ে Validation

Install:

```bash
npm install joi
```

তারপর:

```ts
import { ConfigModule } from '@nestjs/config';
import * as Joi from 'joi';

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,

      validationSchema: Joi.object({
        NODE_ENV: Joi.string()
          .valid(
            'development',
            'test',
            'staging',
            'production',
          )
          .default('development'),

        PORT: Joi.number()
          .port()
          .default(3000),

        DATABASE_URL: Joi.string()
          .required(),

        JWT_SECRET: Joi.string()
          .min(32)
          .required(),

        JWT_EXPIRES_IN: Joi.string()
          .required(),

        REDIS_URL: Joi.string()
          .required(),
      }),
    }),
  ],
})
export class AppModule {}
```

এখন:

```env
JWT_SECRET=short
```

হলে validation fail করতে পারে কারণ আমরা minimum 32 characters চেয়েছি।

---

# 8. Environment আলাদা করা

Production application-এ সাধারণত environment থাকে:

```text
development
test
staging
production
```

এগুলো কেন?

কারণ প্রতিটি environment-এর configuration আলাদা।

---

## Development

আপনার নিজের computer:

```env
NODE_ENV=development

PORT=3000

DATABASE_URL=postgresql://localhost/myapp_dev

REDIS_URL=redis://localhost:6379

JWT_EXPIRES_IN=15m
```

এখানে local database ব্যবহার করবেন।

---

# 9. Test Environment

Automated test-এর জন্য:

```env
NODE_ENV=test

DATABASE_URL=postgresql://localhost/myapp_test

REDIS_URL=redis://localhost:6380

JWT_EXPIRES_IN=5m
```

খুব গুরুত্বপূর্ণ:

> Test environment যেন production database ব্যবহার না করে।

নাহলে test চালাতে গিয়ে production data নষ্ট হয়ে যেতে পারে।

---

# 10. Staging Environment

Staging হলো production-এর মতো একটি environment।

```text
Development
     ↓
   Staging
     ↓
 Production
```

Staging-এ সাধারণত:

```env
NODE_ENV=staging

DATABASE_URL=<staging database>
REDIS_URL=<staging redis>
JWT_SECRET=<staging secret>
```

থাকবে।

Development-এর code production-এ দেওয়ার আগে staging-এ test করা যায়।

---

# 11. Production

Production-এর configuration:

```env
NODE_ENV=production

PORT=3000

DATABASE_URL=<production database>

REDIS_URL=<production redis>

JWT_SECRET=<production secret>

JWT_EXPIRES_IN=15m
```

সবচেয়ে গুরুত্বপূর্ণ বিষয়:

```text
Development JWT_SECRET
       ≠
Staging JWT_SECRET
       ≠
Production JWT_SECRET
```

এক environment-এর secret অন্য environment-এ ব্যবহার করা উচিত না।

---

# 12. `.env` file GitHub-এ দেবেন?

সাধারণত না।

`.gitignore`:

```gitignore
.env
.env.local
.env.development
.env.test
.env.staging
.env.production
```

কারণ `.env`-এ থাকতে পারে:

```text
DATABASE_PASSWORD
JWT_SECRET
AWS_SECRET
API_KEY
```

এগুলো source control-এ commit করা উচিত না।

---

# 13. `.env.example`

কিন্তু অন্য developer যেন বুঝতে পারে কোন configuration লাগবে, তার জন্য:

```text
.env.example
```

রাখতে পারেন।

```env
NODE_ENV=development

PORT=3000

DATABASE_URL=

JWT_SECRET=

JWT_EXPIRES_IN=15m

REDIS_URL=

AWS_ACCESS_KEY=

AWS_SECRET_KEY=
```

এখানে actual secret থাকবে না।

Developer করবে:

```bash
cp .env.example .env
```

তারপর নিজের value বসাবে।

---

# 14. Configuration আলাদা file-এ রাখা

Project বড় হলে শুধু `.env` থেকে সব কিছু নেওয়া messy হয়ে যেতে পারে।

ধরুন:

```text
src/
├── config/
│   ├── app.config.ts
│   ├── database.config.ts
│   ├── auth.config.ts
│   └── redis.config.ts
│
├── app.module.ts
└── main.ts
```

এটা configuration organize করতে সাহায্য করে।

---

# 15. `registerAs()`

NestJS-এ configuration namespace তৈরি করা যায়।

ধরুন:

```ts
// auth.config.ts

import { registerAs } from '@nestjs/config';

export default registerAs('auth', () => ({
  jwtSecret: process.env.JWT_SECRET,
  jwtExpiresIn: process.env.JWT_EXPIRES_IN,
}));
```

তারপর:

```ts
ConfigModule.forRoot({
  isGlobal: true,
  load: [
    authConfig,
  ],
});
```

এখন configuration logically grouped:

```text
auth
 ├── jwtSecret
 └── jwtExpiresIn
```

---

# 16. ConfigService দিয়ে Namespace Access

```ts
const secret = configService.get<string>(
  'auth.jwtSecret',
);
```

এটা:

```env
JWT_SECRET=...
```

থেকে value নিয়ে আসবে।

আর:

```ts
const expiresIn = configService.get<string>(
  'auth.jwtExpiresIn',
);
```

---

# 17. Database Configuration

ধরুন Prisma ব্যবহার করছেন।

Configuration:

```ts
import { registerAs } from '@nestjs/config';

export default registerAs('database', () => ({
  url: process.env.DATABASE_URL,
}));
```

তারপর:

```ts
ConfigModule.forRoot({
  isGlobal: true,
  load: [
    databaseConfig,
  ],
});
```

Service/configuration code:

```ts
const databaseUrl =
  configService.getOrThrow<string>(
    'database.url',
  );
```

এতে database configuration এক জায়গায় organized থাকে।

---

# 18. JWT Configuration

আপনার auth system-এ:

```env
JWT_SECRET=very-long-secret-key
JWT_EXPIRES_IN=15m
JWT_REFRESH_EXPIRES_IN=7d
```

Configuration:

```ts
export default registerAs('auth', () => ({
  jwt: {
    secret: process.env.JWT_SECRET,
    expiresIn: process.env.JWT_EXPIRES_IN,
  },

  refreshToken: {
    expiresIn: process.env.JWT_REFRESH_EXPIRES_IN,
  },
}));
```

তারপর:

```ts
const secret = configService.getOrThrow<string>(
  'auth.jwt.secret',
);

const expiresIn = configService.getOrThrow<string>(
  'auth.jwt.expiresIn',
);
```

এভাবে configuration structured থাকে।

---

# 19. `NODE_ENV`

সবচেয়ে common environment variable:

```env
NODE_ENV=development
```

অথবা:

```env
NODE_ENV=test
```

অথবা:

```env
NODE_ENV=staging
```

অথবা:

```env
NODE_ENV=production
```

Code:

```ts
const environment =
  configService.get<string>('NODE_ENV');
```

তারপর application বুঝতে পারে:

```text
আমি কোথায় run করছি?
       │
       ├── development
       ├── test
       ├── staging
       └── production
```

---

# 20. Environment-specific `.env`

আপনি চাইলে:

```text
.env
.env.development
.env.test
.env.staging
.env.production
```

রাখতে পারেন।

উদাহরণ:

### `.env.development`

```env
NODE_ENV=development
PORT=3000
DATABASE_URL=postgresql://localhost/myapp_dev
```

### `.env.test`

```env
NODE_ENV=test
PORT=3001
DATABASE_URL=postgresql://localhost/myapp_test
```

### `.env.production`

```env
NODE_ENV=production
PORT=3000
DATABASE_URL=<production-db>
```

তারপর environment অনুযায়ী configuration load করবেন।

---

# 21. Configuration-এর একটি Production Structure

একটা clean structure হতে পারে:

```text
src/
│
├── config/
│   ├── app.config.ts
│   ├── auth.config.ts
│   ├── database.config.ts
│   ├── redis.config.ts
│   └── index.ts
│
├── auth/
├── users/
├── database/
│
├── app.module.ts
└── main.ts
```

---

# 22. Example: Complete Configuration

### `.env`

```env
NODE_ENV=development

PORT=3000

DATABASE_URL=postgresql://postgres:password@localhost:5432/myapp

JWT_SECRET=your-super-long-secret-key-here

JWT_EXPIRES_IN=15m

JWT_REFRESH_EXPIRES_IN=7d

REDIS_URL=redis://localhost:6379
```

### `auth.config.ts`

```ts
import { registerAs } from '@nestjs/config';

export default registerAs('auth', () => ({
  jwtSecret: process.env.JWT_SECRET,
  jwtExpiresIn: process.env.JWT_EXPIRES_IN,
  refreshTokenExpiresIn:
    process.env.JWT_REFRESH_EXPIRES_IN,
}));
```

### `app.config.ts`

```ts
import { registerAs } from '@nestjs/config';

export default registerAs('app', () => ({
  environment: process.env.NODE_ENV,
  port: Number(process.env.PORT),
}));
```

### `app.module.ts`

```ts
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';

import appConfig from './config/app.config';
import authConfig from './config/auth.config';

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,

      load: [
        appConfig,
        authConfig,
      ],
    }),
  ],
})
export class AppModule {}
```

### Service

```ts
@Injectable()
export class AuthService {
  constructor(
    private readonly configService: ConfigService,
  ) {}

  getConfig() {
    return {
      secret: this.configService.getOrThrow<string>(
        'auth.jwtSecret',
      ),

      expiresIn:
        this.configService.getOrThrow<string>(
          'auth.jwtExpiresIn',
        ),
    };
  }
}
```

---

# 23. Configuration-এর সবচেয়ে important rule

একজন experienced backend developer হিসেবে এই rule-গুলো মনে রাখবেন:

```text
❌ Secret hard-code করবেন না

❌ Production credentials GitHub-এ রাখবেন না

❌ Production DB test-এর জন্য ব্যবহার করবেন না

❌ Required config missing থাকলেও application start করবেন না

❌ সব configuration এক জায়গায় messy করে রাখবেন না
```

বরং:

```text
                 Configuration
                       │
             ┌─────────┴─────────┐
             │                   │
        Environment          ConfigModule
             │                   │
       ┌─────┼─────┐             │
       │     │     │             ▼
      Dev  Stage  Prod      ConfigService
                               │
                     ┌─────────┼─────────┐
                     │         │         │
                   Auth      Database   Redis
```

---

# 24. আপনার মাথায় পুরো concept-টা এমন থাকা উচিত

```text
                    NestJS
                       │
                 ConfigModule
                       │
                 ConfigService
                       │
              Environment Variables
                       │
        ┌──────────────┼──────────────┐
        │              │              │
   Development       Staging      Production
        │              │              │
        ▼              ▼              ▼
    Dev DB         Stage DB        Prod DB
    Dev Redis      Stage Redis     Prod Redis
    Dev JWT        Stage JWT       Prod JWT
```

**মূল কথা:**

> **Code একই থাকবে, কিন্তু environment অনুযায়ী configuration পরিবর্তন হবে।**

আর production-grade configuration-এর তিনটি সবচেয়ে গুরুত্বপূর্ণ বিষয় হলো:

```text
1. Secrets বাইরে রাখা
2. Startup-এর সময় configuration validate করা
3. Development / Test / Staging / Production আলাদা রাখা
```

এটাই NestJS-এর **Configuration Management**-এর core।
