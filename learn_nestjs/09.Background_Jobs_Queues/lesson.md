# 10. Background Jobs / Queues ⚙️

Production application-এ একটা খুব important rule হলো:

> **যে কাজ user-এর HTTP response দেওয়ার জন্য সঙ্গে সঙ্গে করা দরকার নেই, সেটা background job হিসেবে করা যেতে পারে।**

ধরুন user registration করল। Registration-এর পরে আপনাকে:

* database-এ user তৈরি করতে হবে
* verification email পাঠাতে হবে
* welcome notification পাঠাতে হবে
* analytics event পাঠাতে হবে

সবকিছু যদি একই HTTP request-এর মধ্যে করেন:

```text id="v5j1q7"
User
 ↓
POST /register
 ↓
Create User
 ↓
Send Email
 ↓
Send Notification
 ↓
Analytics
 ↓
Response
```

তাহলে user-কে সব কাজ শেষ হওয়া পর্যন্ত wait করতে হবে।

Background queue ব্যবহার করলে:

```text id="c7n4m2"
User
 ↓
POST /register
 ↓
Create User
 ↓
Add Job to Queue
 ↓
Response immediately
```

তারপর:

```text id="e2k8p5"
Queue
 ↓
Worker
 ↓
Send Email
```

এটাই মূল concept।

---

# 1. Queue কী?

Queue মানে হলো **কাজের waiting line**।

বাস্তব জীবনে ধরুন:

```text id="4j6x8m"
Customer
   ↓
Counter
   ↓
Token
   ↓
Waiting Queue
   ↓
Service
```

Backend-এ:

```text id="r8p2k5"
Request
   ↓
Create Job
   ↓
Queue
   ↓
Worker
   ↓
Process Job
```

ধরুন 1000টা email পাঠাতে হবে:

```text id="m4v7q1"
Queue
│
├── Email Job 1
├── Email Job 2
├── Email Job 3
├── Email Job 4
├── ...
└── Email Job 1000
```

Worker এক এক করে বা parallel-এ এগুলো process করবে।

---

# 2. কেন Queue দরকার?

ধরুন:

```http id="k8s3d1"
POST /register
```

আপনার কাজ:

```text id="q2f7x9"
Create user       → 50ms
Send email        → 800ms
Send notification → 300ms
Analytics         → 200ms
```

সব synchronous করলে:

```text id="6q5n1r"
50 + 800 + 300 + 200
= 1350ms
```

User প্রায় 1.35 seconds wait করতে পারে।

Queue ব্যবহার করলে:

```text id="c4m8v2"
Create user
   ↓
Add email job
   ↓
Response
```

User response পেয়ে যাবে, আর email পরে worker process করবে।

---

# 3. BullMQ কী?

**BullMQ** হলো Node.js-এর জন্য একটি queue/job processing library।

এটি Redis-এর উপর কাজ করে।

Architecture:

```text id="j3f8w6"
              NestJS API
                  │
                  ▼
                BullMQ
                  │
                  ▼
                Redis
                  │
                  ▼
                Worker
                  │
                  ▼
              Process Job
```

এখানে:

```text id="z9r2k4"
BullMQ
↓
Queue manage করে

Redis
↓
Job data/state store করতে সাহায্য করে

Worker
↓
Job execute করে
```

---

# 4. Queue + Worker

দুটো concept আলাদা।

### Queue

কাজ জমা রাখে।

```text id="s7m1q3"
Queue
├── Job 1
├── Job 2
├── Job 3
└── Job 4
```

### Worker

Queue থেকে job নিয়ে কাজ করে।

```text id="f2k8p5"
Queue
  ↓
Worker
  ↓
Process Job
```

সহজভাবে:

> **Queue = কাজের waiting line**
> **Worker = যে waiting line থেকে কাজ নিয়ে করে**

---

# 5. NestJS-এ BullMQ Install

প্রথমে:

```bash id="x8n2m5"
npm install @nestjs/bullmq bullmq
```

Redis server লাগবে।

ধরুন:

```env id="h5q9v1"
REDIS_HOST=localhost
REDIS_PORT=6379
```

---

# 6. Queue Register করা

ধরুন আমরা email queue বানাব:

```ts id="p3k7w2"
import { BullModule } from '@nestjs/bullmq';

@Module({
  imports: [
    BullModule.forRoot({
      connection: {
        host: process.env.REDIS_HOST,
        port: Number(process.env.REDIS_PORT),
      },
    }),

    BullModule.registerQueue({
      name: 'email',
    }),
  ],
})
export class AppModule {}
```

এখন আমাদের queue:

```text id="7qv3k1"
email
```

---

# 7. Job Add করা

ধরুন user register করেছে।

আমরা email job queue-তে পাঠাব:

```ts id="n4y8c2"
import { InjectQueue } from '@nestjs/bullmq';
import { Queue } from 'bullmq';

@Injectable()
export class AuthService {
  constructor(
    @InjectQueue('email')
    private readonly emailQueue: Queue,
  ) {}

  async register(email: string) {
    // Create user...

    await this.emailQueue.add(
      'send-verification-email',
      {
        email,
      },
    );

    return {
      message: 'Registration successful',
    };
  }
}
```

এখানে:

```ts id="h7q1m4"
emailQueue.add(...)
```

মানে:

> "এই কাজটা পরে করতে হবে। Queue-তে রেখে দাও।"

---

# 8. Job-এর Data

আমরা job-এর সাথে data পাঠাতে পারি:

```ts id="v6k2p8"
await this.emailQueue.add(
  'send-verification-email',
  {
    userId: user.id,
    email: user.email,
    verificationToken,
  },
);
```

Queue-তে job:

```text id="w3m9c5"
Job
├── name
│   └── send-verification-email
│
└── data
    ├── userId
    ├── email
    └── verificationToken
```

---

# 9. Worker / Processor

এখন worker queue থেকে job নেবে।

```ts id="d8q2k6"
import {
  Processor,
  WorkerHost,
} from '@nestjs/bullmq';

@Processor('email')
export class EmailProcessor
  extends WorkerHost
{
  async process(job: Job) {
    if (
      job.name ===
      'send-verification-email'
    ) {
      console.log(
        `Sending email to ${job.data.email}`,
      );

      // Send email...
    }
  }
}
```

Flow:

```text id="q7v3m1"
AuthService
    ↓
emailQueue.add()
    ↓
Redis
    ↓
EmailProcessor
    ↓
Send Email
```

---

# 10. API এবং Worker আলাদা চিন্তা করুন

এটা খুব important।

আপনার API server:

```text id="n8k2p4"
API
 ↓
Request গ্রহণ
 ↓
Job তৈরি
 ↓
Response
```

Worker:

```text id="b5m7x1"
Worker
 ↓
Queue থেকে Job
 ↓
Heavy operation
 ↓
Complete
```

Production system-এ worker আলাদা process/container হিসেবেও run করা যায়।

তখন:

```text id="z4q8m2"
                Redis
                  │
        ┌─────────┴─────────┐
        ▼                   ▼
     API Server          Worker
        │                   │
     Add Job            Process Job
```

এতে API server-এর উপর background processing-এর load কমে।

---

# 11. Retry

ধরুন email server temporarily unavailable:

```text id="t2w8k5"
Worker
 ↓
Send Email
 ↓
❌ Failed
```

এখন job পুরোপুরি হারিয়ে না গিয়ে আবার চেষ্টা করানো যায়।

এটাই:

# Retry

BullMQ-তে:

```ts id="m7p3q9"
await this.emailQueue.add(
  'send-verification-email',
  {
    email,
  },
  {
    attempts: 3,
  },
);
```

মানে সর্বোচ্চ 3 বার চেষ্টা করা হবে।

```text id="v8k2m4"
Attempt 1 → ❌
Attempt 2 → ❌
Attempt 3 → ✅
```

---

# 12. Retry কেন দরকার?

সব failure permanent না।

ধরুন external email service:

```text id="p5n7x2"
Request
 ↓
Email Provider
 ↓
Temporary Network Error
```

কিছুক্ষণ পরে আবার চেষ্টা করলে কাজ করতে পারে।

তাই:

```text id="d4q8m1"
Temporary failure
       ↓
Retry
       ↓
Success
```

খুব useful।

---

# 13. Backoff

Retry করলে সঙ্গে সঙ্গে আবার request পাঠানো সবসময় ভালো না।

ধরুন:

```text id="s3j7p5"
Attempt 1
   ↓
Failed

Attempt 2
   ↓
Immediately

Attempt 3
   ↓
Immediately
```

External service এখনও down থাকলে unnecessary pressure তৈরি হবে।

তাই retry-এর মাঝে delay দিতে পারি।

এটাই:

# Backoff

---

# 14. Exponential Backoff

একটি common strategy:

```text id="y8q2m6"
Attempt 1 → fail
       ↓ 1 sec

Attempt 2 → fail
       ↓ 2 sec

Attempt 3 → fail
       ↓ 4 sec

Attempt 4 → fail
       ↓ 8 sec
```

অর্থাৎ delay বাড়তে থাকে।

BullMQ:

```ts id="k4v9p2"
await this.emailQueue.add(
  'send-verification-email',
  {
    email,
  },
  {
    attempts: 5,

    backoff: {
      type: 'exponential',
      delay: 1000,
    },
  },
);
```

এখানে initial delay 1 second।

---

# 15. Fixed Backoff

সবসময় exponential দরকার নেই।

Fixed delay:

```ts id="p1x7m3"
{
  attempts: 3,

  backoff: {
    type: 'fixed',
    delay: 5000,
  },
}
```

মানে:

```text id="e8q4n2"
Attempt 1 → fail
   ↓ 5 sec

Attempt 2 → fail
   ↓ 5 sec

Attempt 3
```

---

# 16. Failed Jobs

ধরুন retry করেও কাজ হলো না:

```text id="z5k2m8"
Attempt 1 ❌
Attempt 2 ❌
Attempt 3 ❌
```

তাহলে job:

```text id="w7p4q1"
FAILED
```

state-এ যেতে পারে।

এগুলোকে **Failed Jobs** বলা হয়।

আপনি পরে:

```text id="f9m3x6"
Failed Jobs
    ↓
Inspect
    ↓
Fix problem
    ↓
Retry
```

করতে পারবেন।

---

# 17. Failed Job কেন রাখা দরকার?

ধরুন 1000 email-এর মধ্যে 10টা fail করল।

আপনি চাইবেন না:

```text id="b2v8k5"
10 failed jobs
      ↓
Lost forever ❌
```

বরং:

```text id="c6m1q9"
1000 jobs
   │
   ├── 990 completed
   │
   └── 10 failed
          ↓
       Inspect
```

তাহলে কোন job কেন fail হয়েছে বুঝতে পারবেন।

---

# 18. Delayed Jobs

সব job immediately execute করতে হবে এমন না।

ধরুন:

> User registration-এর 24 ঘণ্টা পরে reminder email পাঠাতে চাই।

তাহলে delayed job:

```ts id="q8v3m7"
await this.emailQueue.add(
  'send-reminder',
  {
    userId: user.id,
  },
  {
    delay: 24 * 60 * 60 * 1000,
  },
);
```

মানে:

```text id="d5n2x8"
Job created
     ↓
Waiting
     ↓
24 hours
     ↓
Worker
     ↓
Send reminder
```

---

# 19. Delayed Job-এর অন্য Example

ধরুন:

```text id="r3k7p2"
Order created
     ↓
Payment pending
     ↓
30 minutes
     ↓
Check payment
```

অথবা:

```text id="v6m1q8"
Password reset requested
     ↓
Reset token cleanup
     ↓
1 hour
```

---

# 20. আপনার দেওয়া Registration Example

আপনার example:

```text id="y5q8m3"
User registers
      ↓
API returns immediately
      ↓
Queue
      ↓
Email Worker
      ↓
Send verification email
```

পুরো flow:

```text id="g7n2k4"
                 User
                   │
                   ▼
            POST /register
                   │
                   ▼
              AuthService
                   │
          ┌────────┴────────┐
          │                 │
          ▼                 ▼
       Database          Email Queue
          │                 │
          │                 ▼
          │               Redis
          │                 │
          │                 ▼
          │             Email Worker
          │                 │
          │                 ▼
          │            Send Email
          │
          ▼
       Response
```

User-এর response email পাঠানোর জন্য আটকে থাকে না।

---

# 21. Order Example

আপনার দ্বিতীয় example:

```text id="c8m4p1"
Order created
     ↓
Queue
 ┌───┼──────────┐
 ↓   ↓          ↓
Email Payment Notification
```

এখানে তিনটি আলাদা job হতে পারে:

```ts id="x7q2m9"
await emailQueue.add(
  'order-confirmation',
  {
    orderId,
  },
);

await paymentQueue.add(
  'process-payment',
  {
    orderId,
  },
);

await notificationQueue.add(
  'order-notification',
  {
    orderId,
  },
);
```

Architecture:

```text id="z3v8k5"
                  Order Created
                       │
                       ▼
                     Queue
              ┌────────┼────────┐
              ▼        ▼        ▼
            Email    Payment  Notification
              │        │        │
              ▼        ▼        ▼
           Worker    Worker    Worker
```

এতে প্রত্যেকটা কাজ independently process করা যায়।

---

# 22. Job State

একটা job-এর lifecycle বুঝুন:

```text id="m8q3v1"
Waiting
   ↓
Active
   ↓
Completed
```

Failure হলে:

```text id="x5k7p2"
Waiting
   ↓
Active
   ↓
Failed
   ↓
Retry
   ↓
Active
   ↓
Completed
```

Delayed হলে:

```text id="n4w9c6"
Delayed
   ↓
Waiting
   ↓
Active
   ↓
Completed
```

---

# 23. Queue Concurrency

ধরুন queue-তে 1000 jobs আছে।

একজন worker যদি একবারে 1টা process করে:

```text id="s2p8m4"
Worker
 ↓
Job 1
 ↓
Job 2
 ↓
Job 3
```

Slow হতে পারে।

আপনি multiple jobs concurrently process করতে পারেন।

Concept:

```text id="j7k3q9"
Queue
 │
 ├── Worker → Job 1
 ├── Worker → Job 2
 ├── Worker → Job 3
 └── Worker → Job 4
```

তবে concurrency বাড়ালে external service এবং database-এর উপর load-ও বাড়ে।

---

# 24. Idempotency

Queue system-এ এটা খুব important concept।

ধরুন:

```text id="w4m8k2"
Send payment
```

Job accidentally দুইবার execute হলো:

```text id="z7q3p1"
Payment
 ↓
$100 charged

Payment
 ↓
$100 charged again ❌
```

তাই critical jobs এমনভাবে design করতে হয় যাতে একই job multiple times process হলেও duplicate side effect না হয়।

এটাকে **idempotency** বলা হয়।

যেমন payment-এর ক্ষেত্রে:

```text id="h8n2m5"
idempotencyKey = order_123_payment
```

একই key দিয়ে payment provider-কে duplicate charge আটকানোর mechanism দেওয়া যায়, যেখানে provider সেটা support করে।

---

# 25. Queue-তে কোন কাজ করবেন?

ভালো candidates:

```text id="r6p2v8"
Email
SMS
Push notification
Image processing
PDF generation
Report generation
Analytics events
Webhook processing
Data synchronization
Scheduled cleanup
Heavy computation
```

যে কাজগুলো:

```text id="x9m4q1"
slow
retryable
resource-intensive
not immediately required
```

সেগুলো queue-এর জন্য ভালো candidate হতে পারে।

---

# 26. কোন কাজ HTTP Request-এ থাকবে?

যেটার result user-এর response-এর জন্য immediately দরকার:

```text id="v3k7m2"
POST /login
   ↓
Verify credentials
   ↓
Generate authentication result
   ↓
Response
```

এটা সাধারণত request-এর মধ্যেই করতে হবে।

কিন্তু:

```text id="n8q4p6"
Send welcome email
Generate analytics event
Send notification
```

এসব background job হতে পারে।

---

# 27. Queue Failure Architecture

Production-এ এমন হতে পারে:

```text id="b5m8q2"
API
 ↓
Queue
 ↓
Worker
 ↓
External Email Service
       │
       ├── Success → Completed
       │
       └── Failure
              ↓
           Retry
              ↓
           Backoff
              ↓
         Retry exhausted
              ↓
          Failed Job
```

এটাই production-grade চিন্তা।

---

# 28. Redis-এর সাথে সম্পর্ক

আগের chapter-এ Redis শিখেছেন।

এখন connection দেখুন:

```text id="q2m7v4"
                 Redis
                   │
          ┌────────┴────────┐
          │                 │
        Cache             BullMQ
          │                 │
          │               Queue
          │                 │
          │               Worker
          │                 │
          ▼                 ▼
       Fast Data         Background Jobs
```

অর্থাৎ Redis শুধু cache-এর জন্য না।

BullMQ-এর মতো queue system-এর infrastructure হিসেবেও Redis ব্যবহার হয়।

---

# 29. Real Production Architecture

একটা বড় application:

```text id="y8q3m5"
                         Client
                           │
                           ▼
                      Load Balancer
                           │
                           ▼
                       NestJS API
                           │
             ┌─────────────┴─────────────┐
             │                           │
             ▼                           ▼
        PostgreSQL                     Redis
             │                           │
             │                    ┌──────┴──────┐
             │                    │             │
             │                  Cache        BullMQ
             │                                  │
             │                                  ▼
             │                               Workers
             │                                  │
             │                     ┌────────────┼───────────┐
             │                     ▼            ▼           ▼
             │                   Email       Payment    Notification
```

এখানে API-এর কাজ:

> Request গ্রহণ করা এবং প্রয়োজনীয় কাজ queue-তে পাঠানো।

Worker-এর কাজ:

> Queue থেকে job নিয়ে background processing করা।

---

# 30. পুরো Chapter-এর Mental Model

এটা সবচেয়ে ভালোভাবে মনে রাখুন:

```text id="e7m2q9"
                    HTTP Request
                         │
                         ▼
                      NestJS
                         │
                    Add Job
                         │
                         ▼
                       Queue
                         │
                         ▼
                       Redis
                         │
                         ▼
                      Worker
                         │
                    Process Job
                         │
              ┌──────────┼──────────┐
              │          │          │
              ▼          ▼          ▼
           Success     Retry      Failure
              │          │          │
              ▼          ▼          ▼
          Completed    Backoff    Failed Job
```

আর আপনার roadmap-এর প্রতিটি বিষয়:

```text id="f3k8m1"
Background Jobs / Queues
│
├── BullMQ
│   └── Queue/job management
│
├── Redis
│   └── Queue infrastructure
│
├── Queues
│   └── Jobs waiting for processing
│
├── Workers
│   └── Jobs execute করে
│
├── Retry
│   └── Failed job আবার চেষ্টা
│
├── Backoff
│   └── Retry-এর আগে delay
│
├── Failed Jobs
│   └── Retry শেষেও failed হওয়া jobs
│
└── Delayed Jobs
    └── নির্দিষ্ট সময় পরে execute
```

### সবচেয়ে গুরুত্বপূর্ণ flow:

```text id="k9q4v2"
User Request
     ↓
API
     ↓
Queue
     ↓
Redis
     ↓
Worker
     ↓
Background Job
```

**Core idea:** API-কে অপ্রয়োজনীয় heavy কাজ করিয়ে user-কে অপেক্ষা করানোর বদলে, job queue-তে কাজ জমা দিয়ে API দ্রুত response দেয়; worker পরে সেই কাজ process করে। Failure হলে retry/backoff এবং শেষ পর্যন্ত failed-job handling থাকে।
