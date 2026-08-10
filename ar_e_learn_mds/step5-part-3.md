# User Profile Schema (Option A — Thin Profile)

Identity + preference শুধু। Streak/progress/stats এখানে নেই — সেগুলো Step 9-এ আলাদা collection-এ থাকবে।

---

## 1. MongoDB Schema

`src/models/User.ts`

```ts
import { Schema, model, Types } from "mongoose";

export interface IUser {
  _id: Types.ObjectId;
  supabaseUserId: string; // Supabase JWT-এর `sub` claim, unique link
  email: string;
  displayName: string;
  avatarUrl?: string;
  preferredLanguage: "bn" | "en";
  dailyGoalMinutes: number;
  createdAt: Date;
  updatedAt: Date;
}

const userSchema = new Schema<IUser>(
  {
    supabaseUserId: { type: String, required: true, unique: true, index: true },
    email: { type: String, required: true, trim: true, lowercase: true },
    displayName: { type: String, required: true, trim: true },
    avatarUrl: { type: String },
    preferredLanguage: { type: String, enum: ["bn", "en"], default: "bn" },
    dailyGoalMinutes: { type: Number, default: 10 },
  },
  { timestamps: true },
);

export const UserModel = model<IUser>("User", userSchema);
```

---

## 2. Repository

`src/repositories/UserRepository.ts`

```ts
import { UserModel, IUser } from "../models/User";

export const UserRepository = {
  findBySupabaseId: (supabaseUserId: string) =>
    UserModel.findOne({ supabaseUserId }).lean(),

  upsertBySupabaseId: (supabaseUserId: string, data: Partial<IUser>) =>
    UserModel.findOneAndUpdate(
      { supabaseUserId },
      { $set: data, $setOnInsert: { supabaseUserId } },
      { new: true, upsert: true, setDefaultsOnInsert: true },
    ).lean(),
};
```

---

## 3. Validation (Zod)

`src/validations/userValidation.ts`

```ts
import { z } from "zod";

export const syncProfileSchema = z.object({
  body: z.object({
    displayName: z.string().min(1).max(100),
    avatarUrl: z.string().url().optional(),
    preferredLanguage: z.enum(["bn", "en"]).optional(),
  }),
  params: z.object({}).optional(),
  query: z.object({}).optional(),
});

export const updatePreferencesSchema = z.object({
  body: z.object({
    preferredLanguage: z.enum(["bn", "en"]).optional(),
    dailyGoalMinutes: z.coerce.number().int().min(1).max(240).optional(),
  }),
  params: z.object({}).optional(),
  query: z.object({}).optional(),
});
```

---

## 4. Service

`src/services/userService.ts`

```ts
import { UserRepository } from "../repositories/UserRepository";
import { NotFoundError } from "../utils/AppError";

interface SyncProfileInput {
  supabaseUserId: string;
  email: string;
  displayName: string;
  avatarUrl?: string;
  preferredLanguage?: "bn" | "en";
}

export const userService = {
  // Idempotent — বারবার কল করলেও সমস্যা নেই, একই user document আপডেট হয়
  async syncProfile(input: SyncProfileInput) {
    return UserRepository.upsertBySupabaseId(input.supabaseUserId, {
      email: input.email,
      displayName: input.displayName,
      avatarUrl: input.avatarUrl,
      preferredLanguage: input.preferredLanguage,
    });
  },

  async getBySupabaseId(supabaseUserId: string) {
    const user = await UserRepository.findBySupabaseId(supabaseUserId);
    if (!user) throw new NotFoundError("User profile");
    return user;
  },

  async updatePreferences(
    supabaseUserId: string,
    updates: { preferredLanguage?: "bn" | "en"; dailyGoalMinutes?: number },
  ) {
    const user = await UserRepository.upsertBySupabaseId(
      supabaseUserId,
      updates,
    );
    if (!user) throw new NotFoundError("User profile");
    return user;
  },
};
```

---

## 5. Controller

`src/controllers/userController.ts`

```ts
import { Request, Response } from "express";
import { catchAsync } from "../utils/catchAsync";
import { userService } from "../services/userService";

// req.user টা Step 2-এর JWT middleware থেকে আসে (supabaseUserId, email decode করা থাকে)
export const syncProfile = catchAsync(async (req: Request, res: Response) => {
  const { supabaseUserId, email } = req.user!; // JWT middleware থেকে
  const { displayName, avatarUrl, preferredLanguage } = req.body;

  const user = await userService.syncProfile({
    supabaseUserId,
    email,
    displayName,
    avatarUrl,
    preferredLanguage,
  });

  res.json({ success: true, data: user });
});

export const getProfile = catchAsync(async (req: Request, res: Response) => {
  const { supabaseUserId } = req.user!;
  const user = await userService.getBySupabaseId(supabaseUserId);
  res.json({ success: true, data: user });
});

export const updatePreferences = catchAsync(
  async (req: Request, res: Response) => {
    const { supabaseUserId } = req.user!;
    const user = await userService.updatePreferences(supabaseUserId, req.body);
    res.json({ success: true, data: user });
  },
);
```

---

## 6. Routes

`src/routes/userRoutes.ts`

```ts
import { Router } from "express";
import { requireAuth } from "../middleware/requireAuth";
import { validate } from "../middleware/validate";
import {
  syncProfileSchema,
  updatePreferencesSchema,
} from "../validations/userValidation";
import {
  syncProfile,
  getProfile,
  updatePreferences,
} from "../controllers/userController";

const router = Router();

router.use(requireAuth);

router.post("/sync-profile", validate(syncProfileSchema), syncProfile);
router.get("/me", getProfile);
router.patch(
  "/preferences",
  validate(updatePreferencesSchema),
  updatePreferences,
);

export default router;
```

`src/app.ts`-এ (আগে থেকে mount করা না থাকলে):

```ts
import userRoutes from "./routes/userRoutes";
app.use("/api/v1/users", userRoutes);
```

---

**টেস্ট করো:** লগইনের পর mobile app থেকে `POST /api/v1/users/sync-profile` কল হচ্ছে কিনা দেখো (Step 2-এ এটা auth flow-এর অংশ হিসেবে থাকার কথা)। MongoDB Compass-এ `users` collection-এ একটাই document তৈরি হচ্ছে কিনা যাচাই করো — বারবার লগইন করলেও document duplicate না হয়ে update হওয়া উচিত (idempotency test)।

---

**ব্রুটাল ট্রুথ:** এই gap-টা এখন বন্ধ হলো। এখন Step 6 (Audio System)-এ ফেরা যায় নিশ্চিন্তে — কোনো pending architecture decision বাকি নেই।

**পরবর্তী:** Step 6 — Audio System শুরু করবো?
