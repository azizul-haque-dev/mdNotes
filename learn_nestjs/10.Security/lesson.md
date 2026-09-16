# 11. Security 🛡️

Production-grade NestJS backend বানাতে **Security** আলাদা করে শেখা খুব জরুরি।

প্রথমে এই ৩টা লাইন মাথায় রাখুন:

```text
Authentication ≠ Authorization
Validation ≠ Sanitization
Authentication ≠ Security
```

এগুলোর মানে বুঝলে পুরো chapter অনেক সহজ হয়ে যাবে।

---

# 1. Authentication ≠ Authorization

দুটো এক জিনিস না।

## Authentication — "আপনি কে?"

ধরুন আপনি login করছেন:

```text
Email
Password
   ↓
NestJS
   ↓
Credentials যাচাই
   ↓
User authenticated ✅
```

অর্থাৎ:

> **Authentication = user-এর identity verify করা।**

উদাহরণ:

```http
POST /auth/login
```

```json
{
  "email": "user@example.com",
  "password": "secret"
}
```

সঠিক হলে:

```json
{
  "accessToken": "..."
}
```

---

## Authorization — "আপনি কী করতে পারবেন?"

User authenticated:

```text
User = Aziz
```

কিন্তু সে কি:

```text
Delete user
Create product
Delete product
Manage users
```

করতে পারবে?

এটা authorization-এর বিষয়।

```text
Authentication
↓
আমি কে?

Authorization
↓
আমি কী করতে পারি?
```

উদাহরণ:

```text
USER
├── Read products ✓
├── Create product ❌
└── Delete user ❌

ADMIN
├── Read products ✓
├── Create product ✓
└── Delete user ✓
```

---

# 2. Authentication ≠ Security

ধরুন আপনি JWT authentication implement করেছেন।

তার মানে এই না যে application secure।

আপনার application-এ এখনও থাকতে পারে:

```text
XSS
CSRF
SQL Injection
Brute-force
Malicious file upload
Weak password
CORS misconfiguration
Large request attack
```

তাই:

```text
Authentication
       +
Authorization
       +
Input validation
       +
Secure headers
       +
Rate limiting
       +
Secure cookies
       +
Safe database queries
       +
File security
       +
Secret management
       =
Better application security
```

---

# 3. Helmet

HTTP response-এর security-related headers সেট করার জন্য **Helmet** খুব common middleware।

Install:

```bash
npm install helmet
```

NestJS:

```ts
import helmet from 'helmet';

async function bootstrap() {
  const app =
    await NestFactory.create(AppModule);

  app.use(helmet());

  await app.listen(3000);
}
```

Helmet বিভিন্ন security header সেট করতে সাহায্য করে।

যেমন:

```text
Content-Security-Policy
X-Content-Type-Options
Referrer-Policy
Strict-Transport-Security
```

ইত্যাদি।

---

# 4. কেন Security Headers দরকার?

Browser কিছু security behavior headers-এর মাধ্যমে control করে।

উদাহরণ:

```text
Browser
   ↓
HTTP Response
   ↓
Security Headers
   ↓
Browser security rules
```

Helmet-এর কাজ মূলত security-related HTTP headers-এর safer defaults দেওয়া।

তবে Helmet ব্যবহার করলেই application automatically secure হয়ে যায় না।

---

# 5. CORS

CORS = **Cross-Origin Resource Sharing**

ধরুন আপনার frontend:

```text
https://myapp.com
```

আর backend:

```text
https://api.myapp.com
```

Browser-এর security policy অনুযায়ী cross-origin request-এর জন্য server-কে নির্দিষ্টভাবে allow করতে হতে পারে।

NestJS:

```ts
app.enableCors({
  origin: 'https://myapp.com',
});
```

Multiple origins হলে:

```ts
app.enableCors({
  origin: [
    'https://myapp.com',
    'https://admin.myapp.com',
  ],
});
```

---

# 6. Dangerous CORS Configuration

Development-এ অনেক সময় দেখা যায়:

```ts
app.enableCors({
  origin: '*',
});
```

সব application-এর জন্য এটা automatically wrong না, কিন্তু authenticated/private API-এর ক্ষেত্রে এটা blindly ব্যবহার করা উচিত না।

বিশেষ করে credentials ব্যবহার করলে:

```ts
credentials: true
```

origin configuration carefully করতে হয়।

Production-এ:

```text
Known frontend origins
        ↓
Allow
```

অজানা origin:

```text
Unknown origin
        ↓
Don't allow
```

---

# 7. Rate Limiting

Rate limiting মানে:

> নির্দিষ্ট সময়ের মধ্যে একটি client কতগুলো request করতে পারবে তার limit দেওয়া।

ধরুন:

```text
POST /auth/login
```

Rule:

```text
একটি IP
↓
1 minute
↓
5 requests
```

6th request:

```text
429 Too Many Requests
```

NestJS-এ `@nestjs/throttler` ব্যবহার করা যায়।

```bash
npm install @nestjs/throttler
```

Example:

```ts
import { ThrottlerModule } from '@nestjs/throttler';

@Module({
  imports: [
    ThrottlerModule.forRoot([
      {
        ttl: 60000,
        limit: 100,
      },
    ]),
  ],
})
export class AppModule {}
```

এখানে conceptually:

```text
100 requests
within 60 seconds
```

এর বেশি হলে limit apply হবে।

---

# 8. Brute-force Protection

Rate limiting এবং brute-force protection related, কিন্তু একেবারে একই concept না।

ধরুন attacker login করার চেষ্টা করছে:

```text
admin@example.com
password123
password1234
password12345
...
```

হাজার হাজার password try করতে পারে।

এটা brute-force attack।

Authentication endpoint-এর জন্য আরও strict protection রাখা যায়:

```text
POST /auth/login

IP:
5 attempts / minute

Account:
Additional failed-attempt protection
```

আর failed attempts track করতে Redis ব্যবহার করা যায়।

```text
Redis

login:attempts:user@example.com
        ↓
        5
```

তারপর temporarily block/delay করা যায়।

---

# 9. Input Validation

User যা পাঠাচ্ছে সেটা blindly বিশ্বাস করবেন না।

ধরুন API:

```http
POST /users
```

Expected:

```json
{
  "name": "Aziz",
  "age": 20
}
```

কিন্তু user পাঠালো:

```json
{
  "name": 123456,
  "age": "hello"
}
```

Validation সেটা reject করবে।

NestJS-এ:

```bash
npm install class-validator class-transformer
```

DTO:

```ts
import {
  IsEmail,
  IsInt,
  IsString,
  Min,
} from 'class-validator';

export class CreateUserDto {
  @IsString()
  name: string;

  @IsEmail()
  email: string;

  @IsInt()
  @Min(13)
  age: number;
}
```

Global validation:

```ts
app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,
    forbidNonWhitelisted: true,
    transform: true,
  }),
);
```

---

# 10. `whitelist`

ধরুন DTO:

```ts
class CreateUserDto {
  name: string;
  email: string;
}
```

User পাঠালো:

```json
{
  "name": "Aziz",
  "email": "a@example.com",
  "isAdmin": true
}
```

`whitelist: true` unwanted properties remove করতে সাহায্য করে।

আর:

```ts
forbidNonWhitelisted: true
```

দিলে unexpected properties থাকলে request reject করা যায়।

---

# 11. Validation ≠ Sanitization

এটা খুব গুরুত্বপূর্ণ।

### Validation

> Data valid কি না check করা।

উদাহরণ:

```text
email = "hello"
```

Validation বলবে:

```text
❌ Invalid email
```

### Sanitization

> Input-কে নিরাপদ/উপযুক্ত format-এ পরিবর্তন করা।

ধরুন:

```text
"  Aziz  "
```

trim করে:

```text
"Aziz"
```

করা।

তাই:

```text
Validation
↓
"Data acceptable কি?"

Sanitization
↓
"Data-কে safe/clean form-এ কীভাবে আনব?"
```

---

# 12. SQL Injection

ধরুন আপনি raw SQL বানালেন:

```ts
const query = `
  SELECT *
  FROM users
  WHERE email = '${email}'
`;
```

এখানে user-controlled `email` সরাসরি query-এর মধ্যে ঢুকছে।

এটা dangerous।

Attacker malicious SQL input দিতে পারে।

---

# 13. Parameterized Query

Safe approach:

```ts
const query = `
  SELECT *
  FROM users
  WHERE email = $1
`;

await database.query(query, [email]);
```

এখানে:

```text
SQL
+
Parameter
```

আলাদা থাকে।

ORM ব্যবহার করলেও একই principle মনে রাখবেন:

> **User input-কে raw query-এর মধ্যে unsafeভাবে concatenate করবেন না।**

---

# 14. Prisma-এর ক্ষেত্রে

Prisma-এর normal query:

```ts
const user =
  await prisma.user.findUnique({
    where: {
      email,
    },
  });
```

এখানে ORM query structure handle করছে।

Raw SQL দরকার হলে Prisma-র safe parameterization ব্যবহার করতে হবে।

অর্থাৎ:

```text
ORM
↓
Safer query construction
```

কিন্তু ORM ব্যবহার করলেই সব security problem magically disappear করে না।

---

# 15. NoSQL Injection

MongoDB-এর মতো NoSQL database-এর ক্ষেত্রেও malicious query input সমস্যা করতে পারে।

যেমন user input directly query object হিসেবে ব্যবহার করা:

```ts
const user =
  await users.findOne(req.body);
```

যদি request body attacker-controlled হয়, unexpected query operators ঢোকার সুযোগ তৈরি হতে পারে।

তাই:

```text
❌ Entire request body → database query

✓ Validate DTO
✓ Pick allowed fields
✓ Build query explicitly
```

---

# 16. XSS

XSS = **Cross-Site Scripting**

ধরুন user একটি comment লিখল:

```html
<script>
  alert('Hacked');
</script>
```

আপনি সেটা website-এ unsafeভাবে render করলে browser সেটাকে code হিসেবে execute করতে পারে।

---

# 17. XSS Prevention

মূল ধারণা:

```text
User Input
   ↓
Treat as untrusted
   ↓
Safe output encoding
   ↓
Browser
```

Frontend framework যেমন React সাধারণ text rendering-এ escaping করে।

কিন্তু dangerous HTML render করলে:

```tsx
dangerouslySetInnerHTML
```

বিশেষ সতর্কতা দরকার।

Backend-এও user-generated HTML গ্রহণ করলে sanitization strategy প্রয়োজন হতে পারে।

---

# 18. CSRF

CSRF = **Cross-Site Request Forgery**

ধরুন user আপনার website-এ login করা।

তার browser-এ authentication cookie আছে।

Attacker অন্য website থেকে এমন request trigger করার চেষ্টা করতে পারে:

```text
Malicious Website
       ↓
POST /change-email
       ↓
Your API
```

Browser যদি automatically authentication credentials পাঠায় এবং API যথেষ্ট protection না রাখে, সমস্যা হতে পারে।

---

# 19. CSRF কখন বেশি Relevant?

CSRF বিশেষভাবে গুরুত্বপূর্ণ যখন authentication:

```text
Cookie-based
```

এবং browser automatically সেই cookie request-এর সাথে পাঠায়।

যদি JWT শুধু:

```text
Authorization: Bearer <token>
```

header দিয়ে manually পাঠানো হয়, classic CSRF risk আলাদা ধরনের হয়।

তবুও authentication architecture অনুযায়ী protection design করতে হবে।

---

# 20. Secure Cookies

Authentication cookie ব্যবহার করলে cookie configuration গুরুত্বপূর্ণ।

```ts
res.cookie('refreshToken', token, {
  httpOnly: true,
  secure: true,
  sameSite: 'strict',
});
```

এখানে:

### `httpOnly`

JavaScript থেকে cookie access করা কঠিন/অসম্ভব করে:

```text
Browser JS
   ↓
❌ refreshToken access
```

XSS-এর ক্ষেত্রে token theft-এর risk কমাতে সাহায্য করে।

### `secure`

HTTPS connection ছাড়া cookie পাঠানো থেকে browser-কে বিরত রাখে।

### `sameSite`

Cross-site cookie behavior control করতে সাহায্য করে এবং CSRF risk কমাতে সহায়ক।

---

# 21. Password Hashing

Password কখনো database-এ plain text হিসেবে রাখবেন না।

❌:

```text
password = "123456"
```

Database:

```text
123456
```

এটা ভয়ংকর।

বরং:

```text
Password
   ↓
Hash Function
   ↓
Password Hash
   ↓
Database
```

Popular choices:

```text
Argon2
bcrypt
```

---

# 22. Password Hash Example

bcrypt:

```bash
npm install bcrypt
```

```ts
import * as bcrypt from 'bcrypt';

const hash =
  await bcrypt.hash(password, 12);
```

Database:

```text
$2b$12$....
```

Login:

```ts
const isValid =
  await bcrypt.compare(
    password,
    user.passwordHash,
  );
```

Plain password database-এ রাখা হয়নি।

---

# 23. JWT Security

JWT ব্যবহার করলেই secure authentication হয়ে যায় না।

JWT-এর ক্ষেত্রে:

```text
Access Token
Refresh Token
Secret/private key
Expiration
Rotation
Storage
Revocation
```

সব consider করতে হয়।

---

## Short Access Token

Access token-এর lifetime relatively short রাখা যায়:

```text
Access Token
↓
15 minutes
```

Expire হলে refresh flow:

```text
Access Token expired
       ↓
Refresh Token
       ↓
New Access Token
```

---

# 24. JWT Secret

এটা কখনো code-এর মধ্যে hardcode করবেন না।

❌:

```ts
const secret =
  'my-super-secret-key';
```

বরং:

```env
JWT_SECRET=very-long-random-secret
```

আর application configuration থেকে:

```ts
configService.get<string>(
  'JWT_SECRET',
);
```

---

# 25. JWT Payload-এ কী রাখবেন?

JWT payload ছোট এবং প্রয়োজনীয় রাখুন।

যেমন:

```json
{
  "sub": "user_123",
  "role": "USER"
}
```

অপ্রয়োজনীয় sensitive data ঢোকাবেন না।

যেমন:

```text
❌ password
❌ password hash
❌ private secrets
❌ sensitive personal data
```

JWT payload সাধারণত encrypted না; signed।

তাই token-এর payload secret storage হিসেবে ভাববেন না।

---

# 26. Secret Management

আপনার application-এ থাকতে পারে:

```env
DATABASE_URL=
JWT_SECRET=
REDIS_URL=
AWS_ACCESS_KEY=
AWS_SECRET_KEY=
```

এগুলো:

```text
❌ GitHub-এ commit করবেন না
❌ source code-এ hardcode করবেন না
❌ logs-এ print করবেন না
```

Development:

```env
.env
```

Production-এ সাধারণত platform/secret manager-এর মাধ্যমে secrets inject করা হয়।

---

# 27. File Upload Security

ধরুন API:

```http
POST /upload
```

User file পাঠালো।

আপনি blindly accept করলে attacker পাঠাতে পারে:

```text
virus.exe
malicious.svg
huge-file.zip
```

তাই file upload-এ check করতে হবে:

```text
File size
File type
MIME type
Extension
Filename
Content
Storage location
```

---

# 28. File Size Limit

ধরুন profile image:

```text
Maximum = 5 MB
```

কেউ পাঠালো:

```text
2 GB file
```

এটা server resource consume করতে পারে।

তাই upload limit:

```text
Allowed
↓
≤ 5 MB

Rejected
↓
> 5 MB
```

NestJS/Multer configuration-এ limits দেওয়া যায়।

Concept:

```ts
limits: {
  fileSize: 5 * 1024 * 1024,
}
```

---

# 29. File Type Validation

শুধু filename দেখে trust করবেন না:

```text
photo.jpg
```

এটা সত্যিই image কিনা verify করার strategy দরকার।

Allowlist approach ভালো:

```text
image/jpeg
image/png
image/webp
```

যা দরকার নেই:

```text
.exe
.js
.php
```

accept না করাই ভালো।

---

# 30. Request Size Limits

File upload ছাড়াও পুরো HTTP request-এর size limit থাকা উচিত।

ধরুন:

```text
POST /api/data

Body = 500 MB
```

এটা server-এর memory/CPU consume করতে পারে।

তাই:

```text
Normal API
↓
Reasonable body size

File upload
↓
Separate appropriate limit
```

NestJS/underlying HTTP adapter অনুযায়ী body parser limits configure করা যায়।

---

# 31. Authentication Endpoint Security

বিশেষ করে এগুলো protect করবেন:

```text
POST /login
POST /register
POST /forgot-password
POST /reset-password
POST /verify-otp
```

এখানে:

```text
Rate limiting
Brute-force protection
Input validation
Generic error messages
Secure token handling
```

গুরুত্বপূর্ণ।

---

# 32. Error Message-এ Sensitive Information দেবেন না

❌:

```json
{
  "message":
  "User with email admin@example.com exists"
}
```

কিছু ক্ষেত্রে attacker এই ধরনের response দিয়ে account enumeration করতে পারে।

Authentication-related response carefully design করা ভালো।

যেমন:

```json
{
  "message":
  "Invalid credentials"
}
```

---

# 33. Security Layers

একটা production request ভাবুন:

```text id="security-layer-flow"
                Client
                   │
                   ▼
              HTTPS/TLS
                   │
                   ▼
               CORS
                   │
                   ▼
              Rate Limit
                   │
                   ▼
                Helmet
                   │
                   ▼
             Validation
                   │
                   ▼
           Authentication
                   │
                   ▼
           Authorization
                   │
                   ▼
              Controller
                   │
                   ▼
               Service
                   │
                   ▼
              Database
```

প্রতিটি layer-এর আলাদা দায়িত্ব।

---

# 34. আপনার তিনটা Important Statement

এখন আপনার roadmap-এর এই তিনটা statement আবার দেখি।

## Authentication ≠ Authorization

```text
Authentication
↓
Who are you?

Authorization
↓
What can you do?
```

---

## Validation ≠ Sanitization

```text
Validation
↓
Is this data valid?

Sanitization
↓
Can we clean/normalize this data safely?
```

---

## Authentication ≠ Security

```text
Authentication
       ↓
User identity verified

Security
       ↓
Whole application protected
```

Security-এর মধ্যে আরও আছে:

```text
Helmet
CORS
Rate limiting
Brute-force protection
Validation
Injection protection
XSS protection
CSRF protection
Secure cookies
Password hashing
JWT security
Secret management
File security
Request limits
```

---

# 35. পুরো Security Mental Model

এটা মনে রাখুন:

```text
                       SECURITY
                           │
       ┌───────────────────┼───────────────────┐
       │                   │                   │
 Authentication       Authorization        Input
       │                   │              Protection
       │                   │                   │
       ▼                   ▼                   ▼
     JWT/Session        Roles/Guards       Validation
                                           Sanitization
       │                                       │
       └───────────────────┬───────────────────┘
                           │
                           ▼
                    Application Security
                           │
       ┌──────────┬────────┼────────┬──────────┐
       │          │        │        │          │
      CORS     Helmet    Rate     CSRF       XSS
                        Limit
       │
       ├── Password Hashing
       ├── SQL/NoSQL Injection Protection
       ├── Secure Cookies
       ├── Secret Management
       ├── File Upload Security
       └── Request Size Limits
```

সবচেয়ে important idea:

> **Security কোনো একটি package বা feature না। এটা application-এর প্রতিটি layer-এর design decision।**

আপনি `JWT + Helmet + CORS` ব্যবহার করলেই secure application তৈরি হয়ে যায় না। Authentication, authorization, input handling, database queries, cookies, secrets, uploads, rate limits এবং HTTP behavior—সব layer একসাথে নিরাপদভাবে design করতে হয়।
