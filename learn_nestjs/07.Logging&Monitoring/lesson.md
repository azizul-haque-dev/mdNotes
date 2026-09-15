# 8. Logging & Monitoring 📊

Production-grade backend-এর ক্ষেত্রে **Logging & Monitoring** মানে শুধু `console.log()` করা না।

সহজভাবে:

> **Logging বলে কী ঘটেছে, আর Monitoring বলে application এখন কেমন চলছে।**

ধরুন production server-এ একজন user বলল:

> “আমি login করতে পারছি না।”

আপনার কাছে monitoring/logging না থাকলে আপনি অন্ধকারে থাকবেন।

কিন্তু ভালো system হলে:

```text id="6w8d4m"
User Request
     │
     ▼
NestJS
     │
     ├── Request ID
     │
     ├── Structured Log
     │
     ├── Error Tracking
     │
     └── Metrics
            │
            ▼
       Monitoring System
```

তখন আপনি দেখতে পারবেন:

```text id="bqj7gc"
Request ID: req_abc123

POST /auth/login
Status: 401
Duration: 82ms

Error: INVALID_CREDENTIALS
```

এখন এক এক করে দেখি।

---

# 1. NestJS Logger

NestJS-এর built-in `Logger` আছে।

```ts id="q7z8ak"
import { Logger } from '@nestjs/common';

@Injectable()
export class UsersService {
  private readonly logger =
    new Logger(UsersService.name);

  async findUser(id: string) {
    this.logger.log(
      `Finding user: ${id}`,
    );

    // ...
  }
}
```

এখন application log করবে:

```text
[UsersService] Finding user: 123
```

---

# 2. Log Levels

সব log একই ধরনের না।

সাধারণত আপনি বিভিন্ন level ব্যবহার করবেন:

```text id="x2v3t9"
LOG
DEBUG
WARN
ERROR
VERBOSE
```

---

## LOG

Normal important information:

```ts id="f7e9w1"
this.logger.log('User registered successfully');
```

যেমন:

```text
Server started
User created
Payment completed
```

---

## DEBUG

Developer-এর জন্য detailed information।

```ts id="q0h4s1"
this.logger.debug(
  `Checking user with email: ${email}`,
);
```

Development-এ useful।

Production-এ sensitive information log করা যাবে না।

---

## WARN

কোনো অস্বাভাবিক কিন্তু application এখনও চলছে।

```ts id="m3h8s5"
this.logger.warn(
  'Redis connection is slow',
);
```

---

## ERROR

Actual error:

```ts id="k5c1zx"
this.logger.error(
  'Failed to create user',
  error.stack,
);
```

---

# 3. `console.log()` বনাম Logger

Development-এ:

```ts id="f0u5mx"
console.log(user);
```

কাজ করবে।

কিন্তু production application-এ centralized logging দরকার।

তাই:

```ts id="q7m4x2"
this.logger.log(...);
this.logger.warn(...);
this.logger.error(...);
```

ব্যবহার করা ভালো।

কারণ পরে আপনি logger-কে Pino/Winston-এর মতো structured logging system-এর সাথে integrate করতে পারবেন।

---

# 4. Structured Logging

এটা production backend-এর খুব important concept।

Traditional log:

```text id="9qf8s3"
User Aziz logged in
```

এটা মানুষের পড়ার জন্য ভালো।

কিন্তু machine-এর জন্য structured data বেশি useful।

যেমন:

```json id="w4x2k1"
{
  "level": "info",
  "event": "user_login",
  "userId": "123",
  "email": "aziz@example.com",
  "requestId": "req_abc123",
  "timestamp": "2026-09-15T05:30:00.000Z"
}
```

এখন logging system সহজে search করতে পারবে:

```text
event = user_login
userId = 123
requestId = req_abc123
```

---

# 5. কেন Structured Logging?

ধরুন production-এ 1 million log আছে।

আপনি search করলেন:

```text
requestId = req_abc123
```

তখন system বের করতে পারবে:

```text id="8a0c4m"
req_abc123

→ POST /auth/login
→ DB query
→ JWT generation
→ response 200
```

এটাই structured logging-এর power।

---

# 6. Request ID / Correlation ID

এটা অত্যন্ত গুরুত্বপূর্ণ।

ধরুন একজন user একটি request পাঠালো:

```http id="6v3y5k"
POST /orders
```

আমরা একটা unique ID দিলাম:

```text id="1v8k2m"
requestId = req_abc123
```

তারপর request-এর সাথে related সব log-এ এই ID থাকবে।

```text id="0o4n9p"
req_abc123 → POST /orders
req_abc123 → Find user
req_abc123 → Create order
req_abc123 → Payment request
req_abc123 → Response 201
```

এখন request-এর পুরো journey trace করা যায়।

---

# 7. Correlation ID কেন দরকার?

ধরুন একই সময়ে:

```text id="qv5h7p"
User A → POST /orders
User B → POST /orders
User C → POST /orders
```

Logs mix হয়ে যাবে:

```text
Create order
Find user
Payment
Create order
Find user
Payment
...
```

কিন্তু request ID থাকলে:

```text id="j4z0n7"
req_A
 ├── /orders
 ├── user lookup
 ├── payment
 └── response

req_B
 ├── /orders
 ├── user lookup
 ├── payment
 └── response
```

একটা request আলাদা করে track করা যায়।

---

# 8. Request ID Middleware

সহজভাবে middleware দিয়ে শুরু করতে পারেন:

```ts id="v3c7ka"
import {
  Injectable,
  NestMiddleware,
} from '@nestjs/common';

import { randomUUID } from 'crypto';

@Injectable()
export class RequestIdMiddleware
  implements NestMiddleware
{
  use(req: any, res: any, next: () => void) {
    const requestId =
      req.headers['x-request-id'] ??
      randomUUID();

    req.requestId = requestId;

    res.setHeader(
      'X-Request-ID',
      requestId,
    );

    next();
  }
}
```

এখন request:

```http id="p8k2y0"
X-Request-ID: req_abc123
```

থাকলে সেটাই ব্যবহার করতে পারবেন।

না থাকলে server নিজে generate করবে।

---

# 9. Request Logging

Request শুরু এবং শেষের information log করা useful।

Concept:

```text id="0g4j7v"
Request
   ↓
Start time
   ↓
Controller
   ↓
Service
   ↓
Database
   ↓
Response
   ↓
End time
```

তারপর:

```json id="n2c5w8"
{
  "method": "POST",
  "path": "/auth/login",
  "statusCode": 200,
  "duration": 82,
  "requestId": "req_abc123"
}
```

এতে আপনি বুঝতে পারবেন কোন endpoint slow।

---

# 10. Pino

Production Node.js application-এ **Pino** খুব জনপ্রিয় structured logger।

NestJS-এ Pino ব্যবহার করতে `nestjs-pino` ব্যবহার করা যায়।

Install:

```bash id="y8x1t2"
npm install nestjs-pino pino
```

তারপর configuration:

```ts id="u2d7k9"
import { LoggerModule } from 'nestjs-pino';

@Module({
  imports: [
    LoggerModule.forRoot({
      pinoHttp: {
        level: 'info',
      },
    }),
  ],
})
export class AppModule {}
```

এখন structured JSON logs পাওয়া যায়।

---

# 11. Pino কেন Useful?

Traditional:

```text
User created
```

Structured:

```json id="3c8f5j"
{
  "level": 30,
  "msg": "User created",
  "userId": "123"
}
```

এগুলো logging platform সহজে process করতে পারে।

Production application বড় হলে Pino খুব useful।

---

# 12. Winston

আরেকটি popular Node.js logger হলো **Winston**।

Concept:

```text id="s7q4p2"
NestJS
   │
   ▼
Winston
   │
   ├── Console
   ├── File
   └── External logging service
```

যেমন:

```ts id="r8v1k6"
logger.info('User created', {
  userId: user.id,
});
```

Pino এবং Winston দুটোই জানবেন।

কিন্তু এক project-এ সাধারণত একটি logging approach বেছে নিয়ে consistently ব্যবহার করবেন।

---

# 13. Sensitive Data কখনো Log করবেন না

এটা খুব important।

❌:

```ts id="k3d7m2"
this.logger.log({
  email,
  password,
  accessToken,
  refreshToken,
});
```

বিশেষ করে:

```text id="j5r8q1"
password
JWT secret
access token
refresh token
API secret
credit card information
```

log করা উচিত না।

বরং:

```ts id="v9f2c4"
this.logger.log({
  userId: user.id,
  event: 'user_login',
});
```

---

# 14. Error Tracking

Logging এবং error tracking এক জিনিস না।

Logging:

```text
কি ঘটেছে?
```

Error tracking:

```text
কোন error কতবার হচ্ছে?
কোথায় হচ্ছে?
কোন user/request-এর সাথে হচ্ছে?
Stack trace কী?
```

এখানে **Sentry** খুব popular।

Sentry

ধরুন production-এ:

```text id="7v4j1a"
TypeError
Cannot read properties of undefined
```

1000 বার ঘটছে।

Sentry এটাকে group করে দেখাতে পারে:

```text
Error: TypeError
Occurrences: 1,245

Endpoint:
GET /users/:id

First seen:
10:30 AM

Last seen:
11:45 AM
```

এটা developer-এর জন্য খুব useful।

---

# 15. Logging বনাম Error Tracking

সহজভাবে:

```text id="f8x3m1"
Logging
   ↓
"কি ঘটেছিল?"

Error Tracking
   ↓
"কোন error কোথায় এবং কতবার ঘটছে?"
```

দুটো complementary।

---

# 16. Health Checks

এখন ধরুন server চলছে:

```text id="2z6v8p"
NestJS → Running
```

কিন্তু PostgreSQL down:

```text id="q5h1s9"
NestJS → Running
PostgreSQL → DOWN
```

শুধু process running দেখলে মনে হবে সব ঠিক আছে।

Health check বলে:

> Application-এর dependencies ঠিকভাবে কাজ করছে কি না?

NestJS-এ `@nestjs/terminus` ব্যবহার করা যায়।

Install:

```bash id="z3k6v1"
npm install @nestjs/terminus
```

---

# 17. Health Endpoint

ধরুন:

```http id="q8v2n5"
GET /health
```

Response:

```json id="m4j7x9"
{
  "status": "ok"
}
```

আর database down:

```json id="c9k3w2"
{
  "status": "error",
  "database": "down"
}
```

---

# 18. Health Check কেন দরকার?

Deployment system বা monitoring service periodically call করতে পারে:

```text id="x1z5r8"
GET /health
     │
     ▼
Application
     │
     ├── Database ✓
     ├── Redis ✓
     └── Other dependency ✓
```

যদি unhealthy:

```text id="n6b2q0"
Application
     │
     ▼
Health Check ❌
     │
     ▼
Monitoring Alert
```

---

# 19. Liveness বনাম Readiness

Production system-এ এই distinction useful।

### Liveness

Application process বেঁচে আছে কি?

```text
"আমি running আছি"
```

### Readiness

Application request নেওয়ার জন্য ready কি?

```text
"আমি request handle করতে পারছি"
```

উদাহরণ:

```text id="e7r1m3"
Application
    │
    ├── Liveness → ✓
    │
    └── Readiness
          ├── DB → ✓
          └── Redis → ✓
```

---

# 20. Metrics

Logging বলে:

```text
কি ঘটেছে?
```

Metrics বলে:

```text
কতবার ঘটেছে?
কত দ্রুত?
কতজন user?
কত error?
```

যেমন:

```text id="d9f4a7"
HTTP Requests       → 50,000
HTTP Errors         → 500
Average latency     → 120ms
DB connections      → 20
CPU usage           → 65%
Memory usage        → 70%
```

---

# 21. Important Backend Metrics

আপনার application-এর জন্য useful metrics:

```text id="q3w7n2"
Request count
Error count
Error rate
Request latency
Database query latency
Database connections
Active connections
Memory usage
CPU usage
```

আর business metrics:

```text id="r8t2v5"
Registered users
Successful logins
Failed logins
Orders created
Payments failed
```

---

# 22. Prometheus

Prometheus হলো metrics collect/store করার জন্য জনপ্রিয় tool।

Concept:

```text id="u5p8x1"
NestJS
   │
   │ metrics
   ▼
Prometheus
   │
   ▼
Time-series data
```

উদাহরণ:

```text
http_requests_total = 50000
http_errors_total = 500
```

Prometheus সময়ের সাথে metrics store করে।

---

# 23. Grafana

Grafana হলো dashboard visualization-এর জন্য জনপ্রিয়।

Architecture:

```text id="c4m9z7"
NestJS
   │
   ▼
Prometheus
   │
   ▼
Grafana
```

Grafana dashboard-এ দেখতে পারেন:

```text
Requests
██████████████████

Errors
████

Latency
████████

CPU
██████████
```

মানে Prometheus data রাখছে, Grafana সেই data সুন্দরভাবে visualize করতে সাহায্য করছে।

---

# 24. Logging + Monitoring + Error Tracking

এগুলো একসাথে দেখুন:

```text id="z7q2m5"
                   NestJS
                     │
        ┌────────────┼────────────┐
        │            │            │
      Logs         Metrics      Errors
        │            │            │
        ▼            ▼            ▼
      Pino       Prometheus     Sentry
        │            │
        │            ▼
        │         Grafana
        │
        ▼
 Log Management
```

---

# 25. একটা Real Production Request

ধরুন:

```http id="h6p3v9"
POST /orders
```

Request ID:

```text
req_12345
```

Application:

```text id="t2k7x4"
req_12345
     │
     ├── Request started
     │
     ├── User authenticated
     │
     ├── Product checked
     │
     ├── Order created
     │
     ├── Database query: 45ms
     │
     └── Response: 201
```

Structured log:

```json id="s5m8q1"
{
  "level": "info",
  "requestId": "req_12345",
  "method": "POST",
  "path": "/orders",
  "statusCode": 201,
  "duration": 82,
  "event": "request_completed"
}
```

Metrics:

```text
http_requests_total +1
order_created_total +1
http_request_duration = 82ms
```

যদি error হয়:

```text id="n4x7c2"
POST /orders
      │
      ▼
Database Error
      │
      ├── Log → ERROR
      │
      ├── Sentry → Error captured
      │
      └── Metrics → error counter +1
```

User পাবে:

```json id="y8v3k6"
{
  "success": false,
  "statusCode": 500,
  "code": "INTERNAL_SERVER_ERROR",
  "message": "Internal server error"
}
```

কিন্তু developer দেখতে পারবে detailed error।

---

# 26. Development বনাম Production Logging

Development:

```text id="q2w5n8"
DEBUG
LOG
WARN
ERROR
```

অনেক detail রাখা যায়।

Production:

```text id="j7k4p1"
INFO
WARN
ERROR
```

প্রয়োজন অনুযায়ী।

বিশেষ করে:

```text id="s9x3m6"
password
token
secret
personal sensitive data
```

কখনো carelessভাবে log করবেন না।

---

# 27. Production Observability Architecture

একটা mature backend-এর picture:

```text id="b6n2r8"
                     Users
                       │
                       ▼
                  Load Balancer
                       │
                       ▼
                    NestJS
                       │
        ┌──────────────┼──────────────┐
        │              │              │
      Logs          Metrics         Errors
        │              │              │
        ▼              ▼              ▼
      Pino         Prometheus       Sentry
        │              │
        │              ▼
        │           Grafana
        │
        ▼
   Log Platform
```

এখানে তিনটি প্রশ্নের উত্তর পাওয়া যায়:

### Logs

> **কী ঘটেছে?**

### Metrics

> **কতবার/কত দ্রুত ঘটছে?**

### Error Tracking

> **কোথায় error হচ্ছে এবং কেন?**

আর health checks:

> **Application এখন healthy কি না?**

---

# 28. আপনার Roadmap-এর প্রতিটি বিষয়

আপনার list:

```text id="s1q4w7"
Logging & Monitoring
│
├── NestJS Logger
│
├── Structured Logging
│
├── Log Levels
│
├── Request ID
│
├── Correlation ID
│
├── Error Tracking
│
├── Health Checks
│
├── Metrics
│
└── Monitoring
```

পরবর্তীতে tools:

```text id="k5m8p2"
Pino
   ↓
Fast structured logging

Winston
   ↓
Flexible logging

Sentry
   ↓
Error tracking

Prometheus
   ↓
Metrics

Grafana
   ↓
Metrics dashboards
```

সবশেষে mental model:

```text id="r3v7n1"
                  Production Backend
                         │
       ┌─────────────────┼─────────────────┐
       │                 │                 │
     Logging           Metrics          Errors
       │                 │                 │
       ▼                 ▼                 ▼
     Pino           Prometheus          Sentry
       │                 │
       │                 ▼
       │              Grafana
       │
       ▼
  "কি ঘটেছে?"

                    + Health Checks
                          │
                          ▼
                   "System healthy?"
```

**একজন experienced backend engineer-এর লক্ষ্য শুধু error হলে সেটা দেখা নয়; বরং production system-এর behavior এমনভাবে observable করা, যাতে কোনো সমস্যা হলে request ID দিয়ে পুরো request trace করা, error identify করা, metrics দিয়ে impact বোঝা এবং health check দিয়ে system-এর বর্তমান অবস্থা জানা যায়।**
