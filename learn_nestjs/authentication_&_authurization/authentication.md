অবশ্যই। আমি একজন **experienced NestJS/backend engineer** হিসেবে বিষয়গুলো শেখাব, কিন্তু এমনভাবে যেন ১৫ বছরের একজন beginner-ও বুঝতে পারে।

আমরা একটি বাস্তব **NestJS Authentication System** ধরে এগোব:

```text
User
 │
 ├── Register
 │      ↓
 │   Password Hash
 │      ↓
 │   Database
 │
 ├── Login
 │      ↓
 │   Access Token + Refresh Token
 │
 ├── Access protected API
 │      ↓
 │   JWT verification
 │
 ├── Access Token expired
 │      ↓
 │   Refresh Token
 │      ↓
 │   New Access Token
 │
 ├── Logout
 │      ↓
 │   Refresh Token invalid
 │
 ├── Email Verification
 │
 ├── Forgot Password
 │      ↓
 │   Reset Password
 │
 └── Google OAuth
```

---

# 1. JWT

### JWT কী?

JWT-এর পূর্ণরূপ:

**JSON Web Token**

সহজভাবে বললে, JWT হলো এমন একটি **signed token**, যেটা ব্যবহার করে server বুঝতে পারে:

> "এই request কোন user-এর?"

ধরুন আপনি login করলেন:

```http
POST /auth/login
```

আপনি পাঠালেন:

```json
{
  "email": "arman@example.com",
  "password": "123456"
}
```

Server password check করে বলল:

> Login successful.

তারপর server আপনাকে একটি JWT দিল:

```text
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

পরবর্তীতে আপনি protected API call করলে:

```http
GET /users/profile
Authorization: Bearer eyJhbGciOiJIUzI1Ni...
```

Server JWT verify করবে।

---

## JWT-এর ভিতরে কী থাকে?

JWT সাধারণত ৩টি অংশ:

```text
HEADER.PAYLOAD.SIGNATURE
```

যেমন:

```text
xxxxx.yyyyy.zzzzz
```

### Header

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

### Payload

```json
{
  "sub": "user_123",
  "email": "arman@example.com",
  "role": "user"
}
```

### Signature

Server secret key দিয়ে token sign করে:

```text
signature = sign(header + payload, SECRET)
```

এর ফলে client payload পরিবর্তন করলে signature আর match করবে না।

### NestJS example

```ts
import { JwtService } from '@nestjs/jwt';

@Injectable()
export class AuthService {
  constructor(private readonly jwtService: JwtService) {}

  generateToken(userId: string) {
    return this.jwtService.sign({
      sub: userId,
    });
  }
}
```

JWT-এর গুরুত্বপূর্ণ বিষয়:

> **JWT encrypted নয়।**

তাই JWT payload-এ password, credit card number বা secret information রাখবেন না।

---

# 2. Access Token

Access token হলো এমন token যেটা ব্যবহার করে user protected API access করে।

ধরুন:

```http
GET /users/profile
```

Request:

```http
Authorization: Bearer ACCESS_TOKEN
```

Server:

```text
Request
   ↓
JWT Guard
   ↓
Verify Access Token
   ↓
User identified
   ↓
Controller
```

### Example

Login করার পরে:

```json
{
  "accessToken": "eyJhbGciOi..."
}
```

তারপর:

```http
GET /products
Authorization: Bearer eyJhbGciOi...
```

NestJS-এ সাধারণত JWT Guard ব্যবহার করা হয়।

```ts
@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {}
```

Controller:

```ts
@UseGuards(JwtAuthGuard)
@Get('profile')
getProfile(@Request() req) {
  return req.user;
}
```

---

## Access token কতক্ষণ valid থাকবে?

সাধারণত short-lived:

```text
5 minutes
15 minutes
30 minutes
```

যেমন:

```ts
this.jwtService.sign(
  {
    sub: user.id,
  },
  {
    expiresIn: '15m',
  },
);
```

কেন short expiration?

কারণ token চুরি হয়ে গেলে attacker যেন অনেকক্ষণ ব্যবহার করতে না পারে।

---

# 3. Refresh Token

এখন ধরুন:

```text
Access Token = 15 minutes
```

15 মিনিট পরে access token expired।

তাহলে কি user-কে আবার email/password দিয়ে login করতে হবে?

না।

এখানেই Refresh Token আসে।

---

## Refresh token কী?

Refresh token হলো দীর্ঘসময় valid থাকা একটি token যেটা ব্যবহার করে নতুন access token পাওয়া যায়।

Example:

```text
Access Token
15 minutes

Refresh Token
7 days
```

Flow:

```text
Login
  ↓
Access Token
15 min

Refresh Token
7 days
```

15 মিনিট পরে:

```text
Access Token expired
       ↓
Refresh Token পাঠানো
       ↓
Server validates refresh token
       ↓
New Access Token
```

---

## NestJS example

```ts
const accessToken = this.jwtService.sign(
  {
    sub: user.id,
  },
  {
    expiresIn: '15m',
  },
);

const refreshToken = this.jwtService.sign(
  {
    sub: user.id,
  },
  {
    expiresIn: '7d',
  },
);
```

Response:

```json
{
  "accessToken": "...",
  "refreshToken": "..."
}
```

Production application-এ refresh token সাধারণত **HttpOnly Secure cookie**-তে রাখা ভালো।

---

# 4. Refresh Token Rotation

এটা production authentication-এর খুব গুরুত্বপূর্ণ concept।

ধরুন:

```text
Refresh Token A
```

ব্যবহার করে নতুন access token চাওয়া হলো।

সাধারণ system:

```text
Refresh A
   ↓
New Access Token
```

কিন্তু refresh token একই থেকে যায়।

এটা তুলনামূলকভাবে দুর্বল।

---

## Rotation কী?

Rotation মানে:

```text
Refresh Token A
      ↓
   /refresh
      ↓
Refresh Token B
```

অর্থাৎ পুরোনো refresh token invalid হয়ে যাবে।

Flow:

```text
Login
 ↓
Access A
Refresh A
 ↓
Access expired
 ↓
Refresh A
 ↓
Access B
Refresh B
 ↓
Refresh A = INVALID
```

আবার:

```text
Refresh B
 ↓
Access C
Refresh C
```

---

## Database-এ কী রাখা হয়?

Production system-এ refresh token পুরোটা plain text হিসেবে না রেখে hash রাখা ভালো।

ধরুন:

```text
User
 ├── id
 ├── email
 └── passwordHash

RefreshToken
 ├── id
 ├── userId
 ├── tokenHash
 ├── expiresAt
 ├── revokedAt
 └── createdAt
```

Refresh request:

```text
Client
  ↓
Refresh Token
  ↓
Hash token
  ↓
Database hash-এর সাথে compare
  ↓
Valid?
  ↓
Old token revoke
  ↓
New refresh token
  ↓
New access token
```

---

# 5. Token Expiration

Token expiration মানে token কতক্ষণ valid থাকবে।

JWT-তে সাধারণত `exp` claim থাকে।

Example:

```json
{
  "sub": "user123",
  "iat": 1760000000,
  "exp": 1760000900
}
```

এখানে:

```text
iat = issued at
exp = expiration time
```

NestJS:

```ts
this.jwtService.sign(
  {
    sub: user.id,
  },
  {
    expiresIn: '15m',
  },
);
```

Refresh:

```ts
this.jwtService.sign(
  {
    sub: user.id,
  },
  {
    expiresIn: '7d',
  },
);
```

Expired token দিলে:

```text
JWT verification
       ↓
Token expired
       ↓
401 Unauthorized
```

---

# 6. Password Hashing

এটা খুব গুরুত্বপূর্ণ।

ধরুন user password:

```text
MyPassword123
```

আপনি database-এ এটা রাখবেন না:

```json
{
  "password": "MyPassword123"
}
```

কারণ database leak হলে সবার password চলে যাবে।

এর পরিবর্তে password hash করতে হবে:

```text
MyPassword123
      ↓
   Hashing
      ↓
$2b$12$....
```

Database:

```json
{
  "passwordHash": "$2b$12$..."
}
```

---

## Hashing কী?

Hashing হলো one-way mathematical transformation।

```text
password
   ↓
hash()
   ↓
hash
```

কিন্তু:

```text
hash
 ↓
password
```

এভাবে reverse করা যায় না।

---

## Login-এর সময় কী হয়?

Register:

```text
Password
   ↓
Hash
   ↓
Database
```

Login:

```text
Password
   ↓
Compare with stored hash
   ↓
Match?
```

Password decrypt করা হয় না।

---

# 7. bcrypt / argon2

Password hashing-এর জন্য popular দুইটি algorithm:

```text
bcrypt
argon2
```

---

## bcrypt

NestJS application-এ bcrypt ব্যবহার করতে পারেন।

```bash
npm install bcrypt
npm install -D @types/bcrypt
```

Hash:

```ts
import * as bcrypt from 'bcrypt';

const passwordHash = await bcrypt.hash(password, 12);
```

এখানে:

```text
12 = salt rounds / cost factor
```

Compare:

```ts
const isValid = await bcrypt.compare(
  password,
  user.passwordHash,
);
```

তারপর:

```ts
if (!isValid) {
  throw new UnauthorizedException(
    'Invalid credentials',
  );
}
```

---

## Argon2

Argon2 বর্তমানে password hashing-এর জন্য খুব শক্তিশালী modern option।

Install:

```bash
npm install argon2
```

Hash:

```ts
import * as argon2 from 'argon2';

const passwordHash = await argon2.hash(password);
```

Compare:

```ts
const isValid = await argon2.verify(
  user.passwordHash,
  password,
);
```

---

## bcrypt বনাম Argon2

সহজভাবে:

|                  | bcrypt     | Argon2     |
| ---------------- | ---------- | ---------- |
| Mature           | খুব mature | খুব mature |
| Secure           | হ্যাঁ      | হ্যাঁ      |
| Password hashing | হ্যাঁ      | হ্যাঁ      |
| Memory-hard      | না         | হ্যাঁ      |
| Modern choice    | ভালো       | খুব ভালো   |

Production application-এর জন্য **Argon2id** একটি excellent choice।

---

# 8. Login / Logout

## Register

ধরুন:

```http
POST /auth/register
```

Request:

```json
{
  "email": "arman@example.com",
  "password": "StrongPassword123"
}
```

Backend:

```ts
const passwordHash = await argon2.hash(password);

const user = await this.usersService.create({
  email,
  passwordHash,
});
```

---

## Login

```http
POST /auth/login
```

Request:

```json
{
  "email": "arman@example.com",
  "password": "StrongPassword123"
}
```

Server:

```ts
const user = await this.usersService.findByEmail(email);

if (!user) {
  throw new UnauthorizedException('Invalid credentials');
}

const valid = await argon2.verify(
  user.passwordHash,
  password,
);

if (!valid) {
  throw new UnauthorizedException('Invalid credentials');
}
```

তারপর token তৈরি:

```ts
const accessToken = this.jwtService.sign(
  { sub: user.id },
  { expiresIn: '15m' },
);

const refreshToken = this.jwtService.sign(
  { sub: user.id },
  { expiresIn: '7d' },
);
```

---

## Logout

Logout-এর মূল উদ্দেশ্য:

> Refresh token আর ব্যবহার করা যাবে না।

যদি database-এ refresh token থাকে:

```ts
await this.refreshTokenService.revoke(tokenId);
```

তারপর cookie clear:

```ts
response.clearCookie('refreshToken');
```

Flow:

```text
Logout
  ↓
Refresh Token revoke
  ↓
Cookie clear
  ↓
User logged out
```

Access token যদি stateless JWT হয়, সেটা সঙ্গে সঙ্গে server-side "delete" করা হয় না। সাধারণত তার short expiration পর্যন্ত invalidation strategy আলাদাভাবে handle করতে হয়; refresh token revoke করাই মূল logout mechanism।

---

# 9. Email Verification

User registration করার পর email verify করানো হয়।

Example:

```text
Register
   ↓
User created
   ↓
emailVerified = false
   ↓
Verification email
   ↓
User clicks link
   ↓
Backend verifies token
   ↓
emailVerified = true
```

Database:

```ts
{
  email: "arman@example.com",
  emailVerified: false
}
```

---

## Verification token

Server একটি random token তৈরি করতে পারে:

```ts
const token = randomBytes(32).toString('hex');
```

Database-এ ideally token-এর hash রাখবেন:

```text
verificationTokenHash
verificationExpiresAt
```

Email:

```text
https://example.com/verify-email?token=abc123
```

User click করলে:

```http
GET /auth/verify-email?token=abc123
```

Server:

```text
Token
 ↓
Hash
 ↓
Database
 ↓
Match?
 ↓
Expired?
 ↓
emailVerified = true
```

তারপর:

```ts
await this.usersService.markEmailAsVerified(user.id);
```

---

# 10. Password Reset

Password reset এবং password change এক জিনিস নয়।

### Change password

User logged in:

```text
Old password
+
New password
```

### Reset password

User password ভুলে গেছে:

```text
Forgot password
      ↓
Email
      ↓
Reset link
      ↓
New password
```

---

## Flow

```http
POST /auth/forgot-password
```

Request:

```json
{
  "email": "arman@example.com"
}
```

Server:

```text
Find user
   ↓
Generate random reset token
   ↓
Hash token
   ↓
Store hash + expiry
   ↓
Send email
```

Email:

```text
https://example.com/reset-password?token=abc123
```

তারপর:

```http
POST /auth/reset-password
```

```json
{
  "token": "abc123",
  "password": "NewStrongPassword123"
}
```

Backend:

```text
Token
 ↓
Hash
 ↓
Compare DB
 ↓
Check expiration
 ↓
Hash new password
 ↓
Update password
 ↓
Invalidate reset token
```

---

## খুব গুরুত্বপূর্ণ

Forgot-password endpoint-এ এমন response দেওয়া উচিত নয়:

```json
{
  "message": "This email does not exist"
}
```

কারণ attacker email enumeration করতে পারে।

বরং:

```json
{
  "message": "If an account exists, a reset email has been sent."
}
```

---

# 11. Google OAuth

OAuth মানে user আপনার application-এ password না দিয়েও Google account ব্যবহার করে login করতে পারবে।

Flow:

```text
Your App
   ↓
Google
   ↓
User Login
   ↓
Google verifies identity
   ↓
Google sends authorization code
   ↓
Your Backend
   ↓
Google user information
   ↓
Find/Create User
   ↓
Your Access + Refresh Token
```

---

## Example

User clicks:

```text
Continue with Google
```

তারপর:

```text
Google Login
     ↓
Google consent
     ↓
Google callback
     ↓
Backend
```

Google থেকে আপনি সাধারণত identity information পাবেন, যেমন:

```json
{
  "sub": "google-user-id",
  "email": "arman@gmail.com",
  "name": "Arman",
  "picture": "..."
}
```

আপনার database:

```text
User
 ├── id
 ├── email
 ├── name
 └── ...

Account
 ├── userId
 ├── provider
 └── providerAccountId
```

যেমন:

```json
{
  "provider": "google",
  "providerAccountId": "123456789"
}
```

---

## NestJS Passport concept

NestJS-এ Passport Google strategy ব্যবহার করা যায়।

```bash
npm install @nestjs/passport passport passport-google-oauth20
```

Strategy:

```ts
@Injectable()
export class GoogleStrategy extends PassportStrategy(
  Strategy,
  'google',
) {
  constructor() {
    super({
      clientID: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
      callbackURL: process.env.GOOGLE_CALLBACK_URL!,
      scope: ['email', 'profile'],
    });
  }

  async validate(
    accessToken: string,
    refreshToken: string,
    profile: any,
  ) {
    return {
      googleId: profile.id,
      email: profile.emails?.[0]?.value,
      name: profile.displayName,
    };
  }
}
```

তারপর:

```ts
@Get('google')
@UseGuards(AuthGuard('google'))
googleLogin() {}
```

Callback:

```ts
@Get('google/callback')
@UseGuards(AuthGuard('google'))
googleCallback(@Req() req) {
  return this.authService.googleLogin(req.user);
}
```

Google identity পাওয়ার পরে **আপনার application-এর নিজের access/refresh token system** তৈরি করবেন।

---

# 12. Session vs JWT

এটা খুব গুরুত্বপূর্ণ distinction।

## Session Authentication

Session-based system:

```text
Login
 ↓
Server creates session
 ↓
Database / Redis
 ↓
Session ID
 ↓
Cookie
```

Example:

```text
Cookie:
sessionId=abc123
```

Server:

```text
abc123
 ↓
Redis/DB
 ↓
User ID = 123
```

অর্থাৎ server session-এর state রাখে।

---

## JWT Authentication

JWT system:

```text
Login
 ↓
JWT তৈরি
 ↓
Client
 ↓
Authorization Header
 ↓
Server verifies JWT
```

Server সাধারণত প্রতিটি access request-এর জন্য session lookup করে না।

```text
JWT
 ↓
Verify signature
 ↓
Decode claims
 ↓
User identified
```

---

## Main difference

### Session

```text
Client
  ↓
Session ID
  ↓
Server
  ↓
Redis/Database
  ↓
User
```

### JWT

```text
Client
  ↓
JWT
  ↓
Server
  ↓
Verify signature
  ↓
User
```

---

## Session-এর সুবিধা

Server session immediately revoke করতে পারে।

```text
sessionId
 ↓
delete session
 ↓
logged out
```

কিন্তু distributed application-এ shared session storage যেমন Redis লাগতে পারে।

---

## JWT-এর সুবিধা

JWT stateless হওয়ায় API scaling সহজ হতে পারে:

```text
              ┌── Server 1
Client ───────┼── Server 2
              └── Server 3
```

সব server একই JWT secret/public key দিয়ে token verify করতে পারে।

কিন্তু JWT revoke/invalidation-এর বিষয়টি session-এর মতো সরল নয়।

---

# পুরো Authentication Architecture

আপনার NestJS production application-এ বিষয়গুলো একসাথে এমন হতে পারে:

```text
                    ┌──────────────┐
                    │    Client    │
                    └──────┬───────┘
                           │
                     Login/Register
                           │
                           ▼
                    ┌──────────────┐
                    │ Auth Module  │
                    └──────┬───────┘
                           │
             ┌─────────────┼─────────────┐
             ▼             ▼             ▼
        Password        Google        Email
        Auth            OAuth        Verification
             │
             ▼
        Password Hash
        (Argon2id)
             │
             ▼
          Database
             │
             ▼
       Access Token
          15 min
             +
       Refresh Token
           7 days
             │
             ▼
      Refresh Rotation
             │
             ▼
      New Access Token
```

### পুরো login flow:

```text
POST /auth/login
       ↓
Find User
       ↓
Verify Password
       ↓
Generate Access Token
       ↓
Generate Refresh Token
       ↓
Store Refresh Token Hash
       ↓
Return/Set Tokens
```

### Protected API:

```text
GET /users/me
       ↓
Authorization: Bearer JWT
       ↓
JWT Guard
       ↓
Verify Signature
       ↓
Check Expiration
       ↓
req.user
       ↓
Controller
```

### Access token expired:

```text
Access Token expired
       ↓
POST /auth/refresh
       ↓
Validate Refresh Token
       ↓
Revoke Old Refresh Token
       ↓
Generate New Refresh Token
       ↓
Generate New Access Token
       ↓
Return New Tokens
```

### Logout:

```text
POST /auth/logout
       ↓
Revoke Refresh Token
       ↓
Clear Cookie
       ↓
Logged out
```

এটাই হলো একটি **modern NestJS JWT authentication system-এর মূল foundation**।
