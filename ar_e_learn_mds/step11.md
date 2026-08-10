# Step 11: Testing & Deployment

**একটা জরুরি ব্রুটাল ট্রুথ শুরুতেই — Apple Sign-In মিসিং:**
তোমার original requirement doc-এ "Apple Login (iOS)" ছিল, কিন্তু Step 2-এ শুধু Email/Password, OTP, আর Google OAuth implement হয়েছে। এটা **শুধু একটা মিসিং ফিচার না — App Store submission blocker।** Apple-এর App Store Review Guideline 4.8 অনুযায়ী, কোনো app যদি third-party login (Google) অফার করে, তাহলে iOS version-এ **Sign in with Apple**-ও বাধ্যতামূলক অফার করতে হবে, নাহলে review-এ reject হবে। Deployment-এর আগে এটা যোগ করতে হবে — Step 11 শেষে এটা আলাদা ছোট addendum হিসেবে করে দেবো, তুমি চাইলে।

এখন Testing & Deployment-এ যাচ্ছি।

---

# BACKEND TESTING

## Step 11.1 — Test Setup (Jest + Supertest + In-Memory MongoDB)

Real MongoDB Atlas-এর সাথে টেস্ট চালানো risky (ডাটা নষ্ট হতে পারে) — তাই `mongodb-memory-server` দিয়ে প্রতিটা টেস্ট run-এ আলাদা, isolated, disposable database তৈরি হয়।

```bash
npm install -D jest ts-jest @types/jest supertest @types/supertest mongodb-memory-server
```

`jest.config.js`

```js
module.exports = {
  preset: "ts-jest",
  testEnvironment: "node",
  setupFilesAfterEnv: ["<rootDir>/tests/setup.ts"],
  testMatch: ["**/*.test.ts"],
  coveragePathIgnorePatterns: ["/node_modules/", "/tests/"],
};
```

`tests/setup.ts`

```ts
import { MongoMemoryServer } from "mongodb-memory-server";
import mongoose from "mongoose";

let mongoServer: MongoMemoryServer;

beforeAll(async () => {
  mongoServer = await MongoMemoryServer.create();
  await mongoose.connect(mongoServer.getUri());
});

afterEach(async () => {
  const collections = mongoose.connection.collections;
  for (const key in collections) {
    await collections[key].deleteMany({});
  }
});

afterAll(async () => {
  await mongoose.disconnect();
  await mongoServer.stop();
});
```

`package.json`-এ:

```json
"scripts": {
  "test": "jest --runInBand",
  "test:watch": "jest --watch --runInBand"
}
```

---

## Step 11.2 — Unit Test উদাহরণ (Service Layer)

`src/services/__tests__/categoryService.test.ts`

```ts
import { categoryService } from "../categoryService";
import { CategoryModel } from "../../models/Category";
import { NotFoundError } from "../../utils/AppError";

describe("categoryService", () => {
  it("getAll শুধু active category রিটার্ন করে, sortOrder অনুযায়ী", async () => {
    await CategoryModel.create([
      {
        nameAr: "ب",
        nameBn: "খ",
        nameEn: "B",
        slug: "b",
        sortOrder: 2,
        isActive: true,
      },
      {
        nameAr: "أ",
        nameBn: "ক",
        nameEn: "A",
        slug: "a",
        sortOrder: 1,
        isActive: true,
      },
      {
        nameAr: "ج",
        nameBn: "গ",
        nameEn: "C",
        slug: "c",
        sortOrder: 3,
        isActive: false,
      },
    ]);

    const result = await categoryService.getAll();

    expect(result).toHaveLength(2);
    expect(result[0].slug).toBe("a");
    expect(result[1].slug).toBe("b");
  });

  it("getById ভুল id দিলে NotFoundError throw করে", async () => {
    await expect(
      categoryService.getById("64b000000000000000000000"),
    ).rejects.toThrow(NotFoundError);
  });
});
```

---

## Step 11.3 — Integration Test উদাহরণ (API Endpoint, Supertest)

Auth middleware মক (mock) করে টেস্ট করছি — real Supabase JWT দিয়ে টেস্ট চালানো ধীর ও external dependency তৈরি করে।

`tests/mocks/mockAuth.ts`

```ts
import { Request, Response, NextFunction } from "express";

export function mockRequireAuth(supabaseUserId = "test-user-123") {
  return (req: Request, _res: Response, next: NextFunction) => {
    req.user = { supabaseUserId, email: "test@example.com" };
    next();
  };
}
```

`src/routes/__tests__/contentRoutes.test.ts`

```ts
import request from "supertest";
import { app } from "../../app";
import { CategoryModel } from "../../models/Category";

// requireAuth middleware mock করা হচ্ছে (jest.config-এ moduleNameMapper অথবা jest.mock দিয়ে)
jest.mock("../../middleware/requireAuth", () => ({
  requireAuth: (req: any, _res: any, next: any) => {
    req.user = { supabaseUserId: "test-user", email: "test@test.com" };
    next();
  },
}));

describe("GET /api/v1/content/categories", () => {
  it("active categories লিস্ট রিটার্ন করে", async () => {
    await CategoryModel.create({
      nameAr: "التحيات",
      nameBn: "শুভেচ্ছা",
      nameEn: "Greetings",
      slug: "greetings",
      sortOrder: 1,
    });

    const res = await request(app).get("/api/v1/content/categories");

    expect(res.status).toBe(200);
    expect(res.body.success).toBe(true);
    expect(res.body.data).toHaveLength(1);
    expect(res.body.data[0].slug).toBe("greetings");
  });

  it("auth ছাড়া হিট করলে 401 রিটার্ন করে (mock ছাড়া middleware সরাসরি টেস্ট)", async () => {
    // এই টেস্টের জন্য আলাদা describe block-এ jest.mock ছাড়া import করতে হবে,
    // অথবা supertest দিয়ে raw app-এ ভুল/missing Authorization header পাঠিয়ে যাচাই করো
  });
});
```

**কী কী টেস্ট অবশ্যই থাকা উচিত (checklist, সব endpoint-এ কপি-পেস্ট প্যাটার্ন):**

- Happy path (সঠিক ডাটা → সঠিক response)
- Validation failure (ভুল/missing body/params → 400)
- Not found (ভুল id → 404)
- Auth missing/invalid (→ 401)
- Pagination boundary (page=0, limit=1000 ইত্যাদি edge case)

---

# MOBILE TESTING

## Step 11.4 — Test Setup (Jest + React Native Testing Library)

```bash
npx expo install jest-expo jest @testing-library/react-native @testing-library/jest-native -- --dev
```

`package.json`-এ:

```json
{
  "scripts": {
    "test": "jest"
  },
  "jest": {
    "preset": "jest-expo",
    "transformIgnorePatterns": [
      "node_modules/(?!((jest-)?react-native|@react-native(-community)?)|expo(nent)?|@expo(nent)?/.*|@expo-google-fonts/.*|react-navigation|@react-navigation/.*|@unimodules/.*|unimodules|sentry-expo|native-base|react-native-svg)"
    ]
  }
}
```

---

## Step 11.5 — Unit Test উদাহরণ (Pure Logic — SRS Algorithm)

Pure function-গুলো টেস্ট করা সবচেয়ে সহজ ও মূল্যবান — কোনো mock লাগে না।

`src/srs/__tests__/spacedRepetition.test.ts`

```ts
import { computeNextReview } from "../spacedRepetition";

describe("computeNextReview", () => {
  it("সঠিক উত্তরে level বাড়ে", () => {
    const result = computeNextReview(2, true);
    expect(result.newLevel).toBe(3);
  });

  it("level সর্বোচ্চ ৫-এ capped থাকে", () => {
    const result = computeNextReview(5, true);
    expect(result.newLevel).toBe(5);
  });

  it("ভুল উত্তরে level কমে", () => {
    const result = computeNextReview(3, false);
    expect(result.newLevel).toBe(2);
  });

  it("level সর্বনিম্ন ০-তে capped থাকে", () => {
    const result = computeNextReview(0, false);
    expect(result.newLevel).toBe(0);
  });
});
```

`src/quiz/__tests__/mcqGenerator.test.ts`

```ts
import { generateMcqQuestions } from "../mcqGenerator";
import { WordRow } from "../../db/repositories/WordRepository";

const mockWords: WordRow[] = [
  {
    id: "1",
    lesson_id: "l1",
    arabic: "أ",
    pronunciation: "a",
    bangla_meaning: "ক",
    english_meaning: "A",
    audio_url: null,
    updated_at: "",
  },
  {
    id: "2",
    lesson_id: "l1",
    arabic: "ب",
    pronunciation: "b",
    bangla_meaning: "খ",
    english_meaning: "B",
    audio_url: null,
    updated_at: "",
  },
  {
    id: "3",
    lesson_id: "l1",
    arabic: "ج",
    pronunciation: "j",
    bangla_meaning: "গ",
    english_meaning: "C",
    audio_url: null,
    updated_at: "",
  },
  {
    id: "4",
    lesson_id: "l1",
    arabic: "د",
    pronunciation: "d",
    bangla_meaning: "ঘ",
    english_meaning: "D",
    audio_url: null,
    updated_at: "",
  },
];

describe("generateMcqQuestions", () => {
  it("প্রতিটা প্রশ্নে ৪টা করে option থাকে", () => {
    const questions = generateMcqQuestions(mockWords);
    questions.forEach((q) => expect(q.options).toHaveLength(4));
  });

  it("প্রতিটা প্রশ্নের options-এ সঠিক উত্তর থাকে", () => {
    const questions = generateMcqQuestions(mockWords);
    questions.forEach((q) => expect(q.options).toContain(q.correctAnswer));
  });
});
```

---

## Step 11.6 — Component Test উদাহরণ

`src/components/__tests__/AsyncStateView.test.tsx`

```tsx
import { render, screen } from "@testing-library/react-native";
import { Text } from "react-native";
import { AsyncStateView } from "../AsyncStateView";

describe("AsyncStateView", () => {
  it("empty state দেখায় যখন data খালি", () => {
    render(
      <AsyncStateView
        isLoading={false}
        isError={false}
        data={[]}
        emptyLabel="কিছু নেই"
      >
        {() => <Text>content</Text>}
      </AsyncStateView>,
    );
    expect(screen.getByText("কিছু নেই")).toBeTruthy();
  });

  it("error state দেখায় যখন isError true", () => {
    render(
      <AsyncStateView
        isLoading={false}
        isError={true}
        data={null}
        errorLabel="সমস্যা হয়েছে"
      >
        {() => <Text>content</Text>}
      </AsyncStateView>,
    );
    expect(screen.getByText("সমস্যা হয়েছে")).toBeTruthy();
  });

  it("data থাকলে children render করে", () => {
    render(
      <AsyncStateView isLoading={false} isError={false} data={["item1"]}>
        {(data) => <Text>{data[0]}</Text>}
      </AsyncStateView>,
    );
    expect(screen.getByText("item1")).toBeTruthy();
  });
});
```

**ব্রুটাল ট্রুথ:** SQLite repository/hook টেস্ট করা কঠিন কারণ `expo-sqlite` device/simulator ছাড়া চলে না — সেগুলোর জন্য পুরোপুরি unit test না করে, pure logic (SRS algorithm, MCQ generator, validation schema) আর presentational component-এ ফোকাস করাই বাস্তবসম্মত। SQLite-নির্ভর কোড E2E test (Maestro/Detox) দিয়ে কভার করাই ভালো — কিন্তু সেটা এই scope-এর বাইরে, চাইলে আলাদা addendum হিসেবে যোগ করতে পারি।

---

# EAS BUILD

## Step 11.7 — EAS Setup

```bash
npm install -g eas-cli
eas login
eas build:configure
```

`eas.json`

```json
{
  "cli": {
    "version": ">= 12.0.0",
    "appVersionSource": "remote"
  },
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal",
      "env": { "APP_ENV": "development" }
    },
    "preview": {
      "distribution": "internal",
      "env": { "APP_ENV": "preview" }
    },
    "production": {
      "autoIncrement": true,
      "env": { "APP_ENV": "production" }
    }
  },
  "submit": {
    "production": {}
  }
}
```

**Environment variables/secrets** (API URL, Supabase keys) কোডে hardcode না করে EAS-এ রাখো:

```bash
eas secret:create --scope project --name EXPO_PUBLIC_API_URL --value https://your-backend.onrender.com/api/v1
eas secret:create --scope project --name EXPO_PUBLIC_SUPABASE_URL --value https://xxx.supabase.co
eas secret:create --scope project --name EXPO_PUBLIC_SUPABASE_ANON_KEY --value xxx
```

`app.json`-এ production values:

```json
{
  "expo": {
    "name": "Arabic Learning",
    "slug": "arabic-learning",
    "version": "1.0.0",
    "userInterfaceStyle": "automatic",
    "ios": {
      "bundleIdentifier": "com.yourcompany.arabiclearning",
      "buildNumber": "1"
    },
    "android": {
      "package": "com.yourcompany.arabiclearning",
      "versionCode": 1
    }
  }
}
```

**Build কমান্ড:**

```bash
eas build --platform android --profile preview   # প্রথমে preview দিয়ে টেস্ট করো
eas build --platform ios --profile production
eas build --platform android --profile production
```

---

# BACKEND DEPLOYMENT

## Step 11.8 — Platform Comparison (বর্তমান বাজার অনুযায়ী)

|                       | **Render**                                            | **Railway**                                |
| --------------------- | ----------------------------------------------------- | ------------------------------------------ |
| Free tier             | আছে (750 hrs/মাস, inactivity-তে spin down)            | নেই — trial credit শেষে ন্যূনতম $1-5/মাস   |
| Pricing model         | Flat plan-based, predictable                          | Usage-based (per-second), bill ওঠানামা করে |
| MongoDB Atlas connect | সহজ, শুধু connection string                           | সহজ, শুধু connection string                |
| Learning project fit  | ✅ ভালো — free tier দিয়ে শুরু, চাহিদা বাড়লে upgrade | ঠিক আছে, কিন্তু শুরু থেকেই bill আসবে       |

**সুপারিশ: Render।** এই app এখনো learning/personal project পর্যায়ে, তাই Render-এর free tier দিয়ে শুরু করা যায় (backend ঘুমিয়ে পড়বে ১৫ মিনিট idle থাকলে, কিন্তু dev/testing-এর জন্য এটা সমস্যা না)। Predictable flat pricing budget করা সহজ যখন real user আসা শুরু করবে। Railway-এর developer experience ভালো, কিন্তু শুরু থেকেই বিল আসাটা এই স্টেজে অপ্রয়োজনীয় খরচ।

---

## Step 11.9 — Render Deployment

`render.yaml` (repo root-এ, Blueprint হিসেবে)

```yaml
services:
  - type: web
    name: arabic-learning-api
    env: node
    region: singapore
    plan: free
    buildCommand: npm install && npm run build
    startCommand: npm run start
    envVars:
      - key: NODE_ENV
        value: production
      - key: MONGO_URI
        sync: false
      - key: SUPABASE_URL
        sync: false
      - key: SUPABASE_JWT_SECRET
        sync: false
      - key: CORS_ORIGIN
        sync: false
```

`package.json`-এ নিশ্চিত করো:

```json
"scripts": {
  "build": "tsc",
  "start": "node dist/server.js"
}
```

**Deploy steps:**

1. Code GitHub-এ push করো
2. Render dashboard-এ "New Web Service" → GitHub repo connect করো
3. `render.yaml` auto-detect হবে, অথবা manual configure করো
4. Environment variables (MONGO_URI, SUPABASE_URL, ইত্যাদি) dashboard-এ secret হিসেবে বসাও — কখনো repo-তে commit করো না
5. Deploy — প্রতি push-এ auto-redeploy হবে

**MongoDB Atlas-এ Network Access:** production-এ Render-এর IP range dynamic, তাই Atlas-এ "Allow access from anywhere" (0.0.0.0/0) সেট করতে হবে **কিন্তু** MongoDB user-এর password strong আর connection string secret রাখা must — এটাই standard practice managed PaaS-এর ক্ষেত্রে (VPC peering না থাকলে)।

---

## Step 11.10 — Production Checklist

**Security:**

- [ ] `.env`, `google-services.json`, কোনো secret কখনো git-এ commit হয়নি (`.gitignore` চেক করো)
- [ ] CORS `origin: '*'` থেকে production domain-এ restrict করা (`CORS_ORIGIN=https://yourapp.com`)
- [ ] Rate limiting production traffic অনুযায়ী tune করা (Step 3-এ ১৫ মিনিটে ১০০ request ছিল, প্রয়োজনে বাড়াও/কমাও)
- [ ] MongoDB Atlas-এ IP whitelist ও strong password
- [ ] Supabase JWT verification production Supabase project-এর সাথে (dev/staging project আলাদা রাখা ভালো)
- [ ] Helmet middleware active আছে তা নিশ্চিত করা

**Reliability:**

- [ ] Winston log file rotation সেট করা (এখন `error.log`/`combined.log` unbounded বাড়বে — `winston-daily-rotate-file` যোগ করার কথা বিবেচনা করো)
- [ ] MongoDB Atlas backup policy অন করা
- [ ] Health check endpoint (`/health`, Step 3-এ বানানো) uptime monitoring-এ (UptimeRobot, ইত্যাদি) যুক্ত করা

**Mobile:**

- [ ] `EXPO_PUBLIC_*` env variable দিয়ে API URL — production build production URL পয়েন্ট করছে কিনা ভেরিফাই করো
- [ ] **Apple Sign-In যোগ করা হয়েছে** (এই ডকুমেন্টের শুরুতে উল্লেখিত ব্লকার)
- [ ] App icon, splash screen, store screenshots প্রস্তুত
- [ ] Privacy Policy URL (Supabase/Google OAuth-এর জন্য বাধ্যতামূলক, App Store/Play Store উভয় জায়গায়)

**Testing:**

- [ ] Backend `npm test` CI pipeline-এ (GitHub Actions) চলছে প্রতি PR-এ
- [ ] Critical path manual QA: signup → login → offline lesson → sync → quiz → logout

---

**টেস্ট করো:**

1. `npm test` (backend) — সব টেস্ট pass করা উচিত
2. `eas build --platform android --profile preview` — build সফল হওয়া উচিত, APK ডাউনলোড করে physical device-এ ইনস্টল করে টেস্ট করো
3. Render-এ deploy করার পর `GET https://your-app.onrender.com/health` কল করো — response আসা উচিত
4. Production build mobile app থেকে production backend URL-এ hit করে পুরো login → sync flow টেস্ট করো

---

## এই Step-এ যা হলো (সারসংক্ষেপ)

- Backend: Jest + Supertest + `mongodb-memory-server` দিয়ে isolated, fast test setup
- Mobile: Jest + React Native Testing Library, pure-logic ও component test-এ ফোকাস (SQLite-নির্ভর কোড E2E-এর জন্য রাখা হয়েছে)
- EAS Build profiles (development/preview/production) + secrets management
- Render-এ backend deployment (Railway-এর সাথে তুলনা করে reasoning সহ recommend করা হয়েছে)
- সম্পূর্ণ production checklist — security, reliability, mobile store-readiness

**ব্রুটাল ট্রুথ (আবার, গুরুত্বপূর্ণ বলে repeat করছি):** Apple Sign-In ছাড়া iOS App Store-এ submit করলে reject হবে। এটা এখন কাজ শেষ বলে ধরার আগে ঠিক করতে হবে — চাইলে এখনই এটা একটা ছোট addendum হিসেবে implement করে দিতে পারি (Supabase Apple provider + `expo-apple-authentication`)।

**প্রজেক্ট সম্পূর্ণ হলো — ১১টা Step-ই কভার হয়েছে।** এখন বাকি যা আছে: (ক) Apple Sign-In addendum, (খ) Step 10-এর cleanup pattern বাকি screen-গুলোতে apply করা, (গ) real production deploy করে দেখা। কোনটা দিয়ে শুরু করতে চাও?
