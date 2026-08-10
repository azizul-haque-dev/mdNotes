# Step 3: Backend Foundation

Winston logger + layered architecture (Routes → Controllers → Services → Repositories → Middleware) + centralized error handling + validation + security middleware।

প্রতিটা ধাপ আলাদা, একটা শেষ করে টেস্ট করে পরেরটায় যাও।

---

## Step 3.1 — Dependencies install করো

```bash
npm install winston helmet express-rate-limit cors zod dotenv
npm install -D @types/cors
```

- `winston` → logging (console + file)
- `helmet` → security headers
- `express-rate-limit` → rate limiting
- `cors` → cross-origin config
- `zod` → env + request validation
- `dotenv` → .env load

---

## Step 3.2 — Environment Config Layer (validated)

`.env` ভুল থাকলে সাথে সাথে ধরার জন্য Zod দিয়ে validate করা হচ্ছে — server ভুল env নিয়ে চালু হবে না।

`src/config/env.ts`

```ts
import { z } from "zod";
import dotenv from "dotenv";

dotenv.config();

const envSchema = z.object({
  NODE_ENV: z
    .enum(["development", "production", "test"])
    .default("development"),
  PORT: z.coerce.number().default(4000),
  MONGO_URI: z.string().min(1, "MONGO_URI is required"),
  SUPABASE_URL: z.string().min(1, "SUPABASE_URL is required"),
  SUPABASE_JWT_SECRET: z.string().min(1, "SUPABASE_JWT_SECRET is required"),
  CORS_ORIGIN: z.string().default("*"),
  LOG_LEVEL: z.enum(["error", "warn", "info", "debug"]).default("info"),
});

const parsed = envSchema.safeParse(process.env);

if (!parsed.success) {
  console.error("❌ Invalid environment variables:");
  console.error(parsed.error.flatten().fieldErrors);
  process.exit(1);
}

export const env = parsed.data;
```

**টেস্ট করো:** `.env` থেকে কোনো একটা required key বাদ দিয়ে রান করো — server সাথে সাথে বন্ধ হয়ে error দেখাবে কিনা দেখো।

---

## Step 3.3 — Winston Logger (console + file, error আলাদা ফাইলে)

`src/config/logger.ts`

```ts
import winston from "winston";
import path from "path";
import { env } from "./env";

const logsDir = path.join(process.cwd(), "logs");

const logFormat = winston.format.combine(
  winston.format.timestamp({ format: "YYYY-MM-DD HH:mm:ss" }),
  winston.format.errors({ stack: true }),
  winston.format.json(),
);

export const logger = winston.createLogger({
  level: env.LOG_LEVEL,
  format: logFormat,
  transports: [
    // সব error লগ আলাদা ফাইলে
    new winston.transports.File({
      filename: path.join(logsDir, "error.log"),
      level: "error",
    }),
    // সব লগ (info এবং তার উপরে) একসাথে
    new winston.transports.File({
      filename: path.join(logsDir, "combined.log"),
    }),
  ],
});

// Development-এ console-এও দেখাও (readable format)
if (env.NODE_ENV !== "production") {
  logger.add(
    new winston.transports.Console({
      format: winston.format.combine(
        winston.format.colorize(),
        winston.format.simple(),
      ),
    }),
  );
}
```

`.gitignore`-এ যোগ করো:

```
logs/
```

**টেস্ট করো:**

```ts
import { logger } from "./config/logger";
logger.info("Server starting...");
logger.error("Test error log");
```

রান করার পর `logs/combined.log` আর `logs/error.log` ফাইল তৈরি হয়েছে কিনা চেক করো।

---

## Step 3.4 — Custom Error Classes

`src/utils/AppError.ts`

```ts
export class AppError extends Error {
  public readonly statusCode: number;
  public readonly isOperational: boolean;

  constructor(message: string, statusCode: number) {
    super(message);
    this.statusCode = statusCode;
    this.isOperational = true; // predictable/handled error vs bug

    Error.captureStackTrace(this, this.constructor);
  }
}

export class NotFoundError extends AppError {
  constructor(resource = "Resource") {
    super(`${resource} not found`, 404);
  }
}

export class ValidationError extends AppError {
  constructor(message = "Validation failed") {
    super(message, 400);
  }
}

export class UnauthorizedError extends AppError {
  constructor(message = "Unauthorized") {
    super(message, 401);
  }
}
```

---

## Step 3.5 — Global Error Handler Middleware

`src/middleware/errorHandler.ts`

```ts
import { Request, Response, NextFunction } from "express";
import { AppError } from "../utils/AppError";
import { logger } from "../config/logger";
import { env } from "../config/env";

export function errorHandler(
  err: Error,
  req: Request,
  res: Response,
  _next: NextFunction,
) {
  const isAppError = err instanceof AppError;
  const statusCode = isAppError ? err.statusCode : 500;
  const message = isAppError ? err.message : "Internal Server Error";

  logger.error({
    message: err.message,
    stack: err.stack,
    path: req.path,
    method: req.method,
  });

  res.status(statusCode).json({
    success: false,
    message,
    ...(env.NODE_ENV === "development" && { stack: err.stack }),
  });
}

// 404 handler — কোনো route match না হলে
export function notFoundHandler(req: Request, res: Response) {
  res.status(404).json({
    success: false,
    message: `Route ${req.originalUrl} not found`,
  });
}
```

**Async route-এর error catch করার জন্য wrapper:**

`src/utils/catchAsync.ts`

```ts
import { Request, Response, NextFunction, RequestHandler } from "express";

export const catchAsync =
  (fn: RequestHandler) => (req: Request, res: Response, next: NextFunction) => {
    Promise.resolve(fn(req, res, next)).catch(next);
  };
```

---

## Step 3.6 — Request Validation Middleware (Zod)

`src/middleware/validate.ts`

```ts
import { Request, Response, NextFunction } from "express";
import { AnyZodObject, ZodError } from "zod";
import { ValidationError } from "../utils/AppError";

export const validate =
  (schema: AnyZodObject) =>
  (req: Request, _res: Response, next: NextFunction) => {
    try {
      schema.parse({
        body: req.body,
        query: req.query,
        params: req.params,
      });
      next();
    } catch (err) {
      if (err instanceof ZodError) {
        const message = err.errors
          .map((e) => `${e.path.join(".")}: ${e.message}`)
          .join(", ");
        return next(new ValidationError(message));
      }
      next(err);
    }
  };
```

---

## Step 3.7 — Security + App Wiring

`src/app.ts`

```ts
import express from "express";
import helmet from "helmet";
import cors from "cors";
import rateLimit from "express-rate-limit";
import { env } from "./config/env";
import { logger } from "./config/logger";
import { errorHandler, notFoundHandler } from "./middleware/errorHandler";

export const app = express();

// Security headers
app.use(helmet());

// CORS
app.use(cors({ origin: env.CORS_ORIGIN }));

// Body parsing
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// Rate limiting (সব API-তে global limit)
app.use(
  rateLimit({
    windowMs: 15 * 60 * 1000, // 15 min
    max: 100,
    message: "Too many requests, please try again later.",
    standardHeaders: true,
    legacyHeaders: false,
  }),
);

// Request logging
app.use((req, _res, next) => {
  logger.info(`${req.method} ${req.path}`);
  next();
});

// Health check
app.get("/health", (_req, res) => {
  res.json({ status: "ok", timestamp: new Date().toISOString() });
});

// TODO: routes will be mounted here in Step 5+ (categories, lessons, words...)

// 404 + global error handler (সবসময় সবার শেষে)
app.use(notFoundHandler);
app.use(errorHandler);
```

`src/server.ts`

```ts
import mongoose from "mongoose";
import { app } from "./app";
import { env } from "./config/env";
import { logger } from "./config/logger";

async function startServer() {
  try {
    await mongoose.connect(env.MONGO_URI);
    logger.info("✅ MongoDB connected");

    app.listen(env.PORT, () => {
      logger.info(`🚀 Server running on port ${env.PORT} [${env.NODE_ENV}]`);
    });
  } catch (err) {
    logger.error("❌ Failed to start server", err);
    process.exit(1);
  }
}

startServer();

// Unhandled rejections/exceptions ধরার জন্য
process.on("unhandledRejection", (reason) => {
  logger.error("Unhandled Rejection:", reason);
  process.exit(1);
});
```

**টেস্ট করো:**

```bash
npm run dev
```

তারপর ব্রাউজার/Postman-এ `GET http://localhost:4000/health` কল করো — `{ status: "ok" }` আসা উচিত।
একটা ভুল route হিট করো (`/xyz`) — 404 JSON response আর `logs/combined.log`-এ entry দেখা উচিত।

---

## এই Step-এ যা হলো (সারসংক্ষেপ)

- Winston দিয়ে logging — error আলাদা ফাইলে, সব লগ combined ফাইলে
- Zod দিয়ে .env validate — ভুল config নিয়ে server চালু হবে না
- Custom Error classes (`AppError`, `NotFoundError`, ইত্যাদি) — predictable error handling
- Global error handler — সব error এক জায়গা থেকে handle, response format consistent
- `catchAsync` wrapper — প্রতিটা async controller-এ try/catch লেখা লাগবে না
- Helmet + rate limit + CORS — basic security
- `app.ts` আর `server.ts` আলাদা — testing-এর সময় app export করে সহজে test করা যাবে (Step 11-এ কাজে লাগবে)

**ব্রুটাল ট্রুথ:** এখানে এখনো MongoDB models/schemas (User, Categories, Lessons...) বানানো হয়নি — সেটা ইচ্ছাকৃতভাবে বাদ দেওয়া হয়েছে কারণ content-specific schema গুলো Step 5-এ realistic data নিয়ে একসাথে ডিজাইন করাই ভালো প্র্যাকটিস। এখন শুধু User schema-টা লাগবে কিনা (Supabase Auth থাকা সত্ত্বেও MongoDB-তে profile রাখার জন্য) — এটা পরের ছোট ধাপ হিসেবে করতে পারি, চাইলে বলো।

**পরবর্তী যৌক্তিক ধাপ:** User Profile Schema + Repository (MongoDB) — কারণ Step 2-এর Supabase Auth-এর সাথে এটা সরাসরি যুক্ত, আর Step 4 (Local SQLite) শুরুর আগে backend-এর data layer-টা basic থাকা দরকার।
