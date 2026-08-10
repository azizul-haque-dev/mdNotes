# Step 9: User Progress (Bookmarks, Daily Goal, Streak, Statistics)

**Architecture decision:** Requirement-এ "Favorites" আর "Bookmarks" আলাদা লেখা আছে, কিন্তু কার্যকরীভাবে দুটোই একই জিনিস — "এই item-টা পরে দেখার জন্য সেভ করে রাখা"। আলাদা দুটো collection/table বানালে duplicate logic হতো ("Never duplicate business logic" rule ভাঙতো)। তাই **একটাই Bookmark সিস্টেম** — `item_type` ('word' | 'sentence' | 'lesson') দিয়ে আলাদা করা হচ্ছে, Favorites শুধু UI label হিসেবে ব্যবহার হবে।

---

# BACKEND

## Step 9.1 — Bookmark Schema + API

`src/models/Bookmark.ts`

```ts
import { Schema, model, Types } from "mongoose";

export interface IBookmark {
  _id: Types.ObjectId;
  supabaseUserId: string;
  itemType: "word" | "sentence" | "lesson";
  itemId: string;
  createdAt: Date;
}

const bookmarkSchema = new Schema<IBookmark>(
  {
    supabaseUserId: { type: String, required: true, index: true },
    itemType: {
      type: String,
      enum: ["word", "sentence", "lesson"],
      required: true,
    },
    itemId: { type: String, required: true },
  },
  { timestamps: { createdAt: true, updatedAt: false } },
);

bookmarkSchema.index(
  { supabaseUserId: 1, itemType: 1, itemId: 1 },
  { unique: true },
);

export const BookmarkModel = model<IBookmark>("Bookmark", bookmarkSchema);
```

`src/repositories/BookmarkRepository.ts`

```ts
import { BookmarkModel } from "../models/Bookmark";

export const BookmarkRepository = {
  add: (supabaseUserId: string, itemType: string, itemId: string) =>
    BookmarkModel.findOneAndUpdate(
      { supabaseUserId, itemType, itemId },
      { $setOnInsert: { supabaseUserId, itemType, itemId } },
      { upsert: true, new: true },
    ).lean(),

  remove: (supabaseUserId: string, itemType: string, itemId: string) =>
    BookmarkModel.deleteOne({ supabaseUserId, itemType, itemId }),

  findAllByUser: (supabaseUserId: string) =>
    BookmarkModel.find({ supabaseUserId }).sort({ createdAt: -1 }).lean(),
};
```

`src/validations/bookmarkValidation.ts`

```ts
import { z } from "zod";

export const syncBookmarkSchema = z.object({
  body: z.object({
    itemType: z.enum(["word", "sentence", "lesson"]),
    itemId: z.string().min(1),
    action: z.enum(["add", "remove"]),
  }),
  params: z.object({}).optional(),
  query: z.object({}).optional(),
});
```

`src/services/bookmarkService.ts`

```ts
import { BookmarkRepository } from "../repositories/BookmarkRepository";

export const bookmarkService = {
  async sync(
    supabaseUserId: string,
    itemType: "word" | "sentence" | "lesson",
    itemId: string,
    action: "add" | "remove",
  ) {
    if (action === "add") {
      return BookmarkRepository.add(supabaseUserId, itemType, itemId);
    }
    await BookmarkRepository.remove(supabaseUserId, itemType, itemId);
    return null;
  },

  async getAllForUser(supabaseUserId: string) {
    return BookmarkRepository.findAllByUser(supabaseUserId);
  },
};
```

`src/controllers/bookmarkController.ts`

```ts
import { Request, Response } from "express";
import { catchAsync } from "../utils/catchAsync";
import { bookmarkService } from "../services/bookmarkService";

export const syncBookmark = catchAsync(async (req: Request, res: Response) => {
  const { supabaseUserId } = req.user!;
  const { itemType, itemId, action } = req.body;
  const result = await bookmarkService.sync(
    supabaseUserId,
    itemType,
    itemId,
    action,
  );
  res.json({ success: true, data: result });
});

export const getMyBookmarks = catchAsync(
  async (req: Request, res: Response) => {
    const { supabaseUserId } = req.user!;
    const bookmarks = await bookmarkService.getAllForUser(supabaseUserId);
    res.json({ success: true, data: bookmarks });
  },
);
```

`src/routes/bookmarkRoutes.ts`

```ts
import { Router } from "express";
import { requireAuth } from "../middleware/requireAuth";
import { validate } from "../middleware/validate";
import { syncBookmarkSchema } from "../validations/bookmarkValidation";
import {
  syncBookmark,
  getMyBookmarks,
} from "../controllers/bookmarkController";

const router = Router();
router.use(requireAuth);
router.post("/sync", validate(syncBookmarkSchema), syncBookmark);
router.get("/", getMyBookmarks);

export default router;
```

`src/app.ts`-এ:

```ts
import bookmarkRoutes from "./routes/bookmarkRoutes";
app.use("/api/v1/bookmarks", bookmarkRoutes);
```

---

## Step 9.2 — Daily Activity Schema (Streak/Statistics-এর ভিত্তি)

প্রতিটা sync-এ পুরো দিনের total minutes **set** করা হয় (increment না) — এতে outbox retry হলে double-count হবে না, idempotent থাকবে।

`src/models/DailyActivity.ts`

```ts
import { Schema, model, Types } from "mongoose";

export interface IDailyActivity {
  _id: Types.ObjectId;
  supabaseUserId: string;
  date: string; // 'YYYY-MM-DD'
  minutes: number;
}

const dailyActivitySchema = new Schema<IDailyActivity>({
  supabaseUserId: { type: String, required: true, index: true },
  date: { type: String, required: true },
  minutes: { type: Number, default: 0 },
});

dailyActivitySchema.index({ supabaseUserId: 1, date: 1 }, { unique: true });

export const DailyActivityModel = model<IDailyActivity>(
  "DailyActivity",
  dailyActivitySchema,
);
```

`src/repositories/DailyActivityRepository.ts`

```ts
import { DailyActivityModel } from "../models/DailyActivity";

export const DailyActivityRepository = {
  setMinutes: (supabaseUserId: string, date: string, minutes: number) =>
    DailyActivityModel.findOneAndUpdate(
      { supabaseUserId, date },
      { $set: { minutes } },
      { upsert: true, new: true },
    ).lean(),
};
```

`src/validations/activityValidation.ts`

```ts
import { z } from "zod";

export const syncActivitySchema = z.object({
  body: z.object({
    date: z.string().regex(/^\d{4}-\d{2}-\d{2}$/),
    minutes: z.coerce.number().min(0),
  }),
  params: z.object({}).optional(),
  query: z.object({}).optional(),
});
```

`src/controllers/activityController.ts`

```ts
import { Request, Response } from "express";
import { catchAsync } from "../utils/catchAsync";
import { DailyActivityRepository } from "../repositories/DailyActivityRepository";

export const syncActivity = catchAsync(async (req: Request, res: Response) => {
  const { supabaseUserId } = req.user!;
  const { date, minutes } = req.body;
  const result = await DailyActivityRepository.setMinutes(
    supabaseUserId,
    date,
    minutes,
  );
  res.json({ success: true, data: result });
});
```

`src/routes/activityRoutes.ts`

```ts
import { Router } from "express";
import { requireAuth } from "../middleware/requireAuth";
import { validate } from "../middleware/validate";
import { syncActivitySchema } from "../validations/activityValidation";
import { syncActivity } from "../controllers/activityController";

const router = Router();
router.use(requireAuth);
router.post("/sync", validate(syncActivitySchema), syncActivity);

export default router;
```

`src/app.ts`-এ:

```ts
import activityRoutes from "./routes/activityRoutes";
app.use("/api/v1/activity", activityRoutes);
```

---

# MOBILE

## Step 9.3 — SQLite Migration: Daily Activity টেবিল

`bookmarks` টেবিল Step 4.3-এই বানানো হয়েছিল, নতুন শুধু `daily_activity`।

`src/db/migrations/005_add_daily_activity.ts`

```ts
import { SQLiteDatabase } from "expo-sqlite";
import { Migration } from "./index";

export const migration_005_add_daily_activity: Migration = {
  version: 5,
  name: "add_daily_activity",
  up: async (db: SQLiteDatabase) => {
    await db.execAsync(`
      CREATE TABLE IF NOT EXISTS daily_activity (
        date TEXT PRIMARY KEY,
        minutes INTEGER NOT NULL DEFAULT 0,
        is_synced INTEGER DEFAULT 0
      );
    `);
  },
};
```

`src/db/migrations/index.ts`-এ যোগ করো:

```ts
import { migration_005_add_daily_activity } from "./005_add_daily_activity";

export const migrations: Migration[] = [
  migration_001_init,
  migration_002_add_sentence_transliteration,
  migration_003_add_conversations,
  migration_004_add_srs,
  migration_005_add_daily_activity, // যোগ করো
];
```

---

## Step 9.4 — Bookmark Repository + Sync

`src/db/repositories/BookmarkRepository.ts`

```ts
import { BaseRepository } from "./BaseRepository";
import { randomUUID } from "expo-crypto";

export interface BookmarkRow {
  id: string;
  item_type: "word" | "sentence" | "lesson";
  item_id: string;
  created_at: string;
  is_synced: number;
}

class BookmarkRepositoryClass extends BaseRepository<BookmarkRow> {
  protected tableName = "bookmarks";

  async isBookmarked(itemType: string, itemId: string): Promise<boolean> {
    const db = await this.db();
    const row = await db.getFirstAsync(
      "SELECT id FROM bookmarks WHERE item_type = ? AND item_id = ?",
      [itemType, itemId],
    );
    return !!row;
  }

  async findByType(itemType: string): Promise<BookmarkRow[]> {
    const db = await this.db();
    return db.getAllAsync<BookmarkRow>(
      "SELECT * FROM bookmarks WHERE item_type = ? ORDER BY created_at DESC",
      [itemType],
    );
  }

  async add(itemType: BookmarkRow["item_type"], itemId: string): Promise<void> {
    const db = await this.db();
    await db.runAsync(
      `INSERT INTO bookmarks (id, item_type, item_id, created_at, is_synced) VALUES (?, ?, ?, ?, 0)`,
      [randomUUID(), itemType, itemId, new Date().toISOString()],
    );
  }

  async removeByItem(itemType: string, itemId: string): Promise<void> {
    const db = await this.db();
    await db.runAsync(
      "DELETE FROM bookmarks WHERE item_type = ? AND item_id = ?",
      [itemType, itemId],
    );
  }
}

export const BookmarkRepository = new BookmarkRepositoryClass();
```

`src/features/bookmarks/bookmarkActions.ts`

```ts
import { BookmarkRepository } from "../../db/repositories/BookmarkRepository";
import { SyncQueueRepository } from "../../db/repositories/SyncQueueRepository";

export async function toggleBookmark(
  itemType: "word" | "sentence" | "lesson",
  itemId: string,
): Promise<boolean> {
  const alreadyBookmarked = await BookmarkRepository.isBookmarked(
    itemType,
    itemId,
  );

  if (alreadyBookmarked) {
    await BookmarkRepository.removeByItem(itemType, itemId);
    await SyncQueueRepository.enqueue("bookmark", itemId, "delete", {
      itemType,
      itemId,
      action: "remove",
    });
    return false;
  } else {
    await BookmarkRepository.add(itemType, itemId);
    await SyncQueueRepository.enqueue("bookmark", itemId, "create", {
      itemType,
      itemId,
      action: "add",
    });
    return true;
  }
}
```

`src/features/bookmarks/registerBookmarkSync.ts`

```ts
import { registerSyncHandler } from "../../sync/syncEngine";
import { apiClient } from "../../api/apiClient";

export function registerBookmarkSyncHandler(): void {
  registerSyncHandler("bookmark", async (item) => {
    const payload = JSON.parse(item.payload);
    await apiClient.post("/bookmarks/sync", payload);
  });
}
```

`app/_layout.tsx`-এ যোগ করো:

```ts
import { registerBookmarkSyncHandler } from "../src/features/bookmarks/registerBookmarkSync";
registerBookmarkSyncHandler();
```

---

## Step 9.5 — Daily Activity Repository + Streak Calculation

`src/db/repositories/DailyActivityRepository.ts`

```ts
import { BaseRepository } from "./BaseRepository";

export interface DailyActivityRow {
  date: string;
  minutes: number;
  is_synced: number;
}

function todayDateString(): string {
  return new Date().toISOString().split("T")[0]; // 'YYYY-MM-DD'
}

class DailyActivityRepositoryClass extends BaseRepository<DailyActivityRow> {
  protected tableName = "daily_activity";

  async addMinutesToday(minutesToAdd: number): Promise<DailyActivityRow> {
    const db = await this.db();
    const date = todayDateString();

    const existing = await db.getFirstAsync<DailyActivityRow>(
      "SELECT * FROM daily_activity WHERE date = ?",
      [date],
    );

    const newMinutes = (existing?.minutes ?? 0) + minutesToAdd;

    if (existing) {
      await db.runAsync(
        "UPDATE daily_activity SET minutes = ?, is_synced = 0 WHERE date = ?",
        [newMinutes, date],
      );
    } else {
      await db.runAsync(
        "INSERT INTO daily_activity (date, minutes, is_synced) VALUES (?, ?, 0)",
        [date, newMinutes],
      );
    }

    return { date, minutes: newMinutes, is_synced: 0 };
  }

  async markSynced(date: string): Promise<void> {
    const db = await this.db();
    await db.runAsync(
      "UPDATE daily_activity SET is_synced = 1 WHERE date = ?",
      [date],
    );
  }

  async getTodayMinutes(): Promise<number> {
    const db = await this.db();
    const row = await db.getFirstAsync<DailyActivityRow>(
      "SELECT * FROM daily_activity WHERE date = ?",
      [todayDateString()],
    );
    return row?.minutes ?? 0;
  }

  async getTotalMinutes(): Promise<number> {
    const db = await this.db();
    const row = await db.getFirstAsync<{ total: number }>(
      "SELECT COALESCE(SUM(minutes), 0) as total FROM daily_activity",
    );
    return row?.total ?? 0;
  }

  // consecutive দিন হিসাব — আজ অথবা গতকাল থেকে শুরু করে পেছনের দিকে গোনা হয়
  async computeStreak(): Promise<number> {
    const db = await this.db();
    const rows = await db.getAllAsync<{ date: string }>(
      "SELECT date FROM daily_activity WHERE minutes > 0 ORDER BY date DESC",
    );
    const activeDates = new Set(rows.map((r) => r.date));

    let streak = 0;
    const cursor = new Date();

    // আজ practice না করলেও streak গতকাল পর্যন্ত ধরে রাখা হয় (দিন শেষ হওয়ার আগ পর্যন্ত সুযোগ)
    if (!activeDates.has(todayDateString())) {
      cursor.setDate(cursor.getDate() - 1);
    }

    while (activeDates.has(cursor.toISOString().split("T")[0])) {
      streak += 1;
      cursor.setDate(cursor.getDate() - 1);
    }

    return streak;
  }
}

export const DailyActivityRepository = new DailyActivityRepositoryClass();
```

`src/features/activity/registerActivitySync.ts`

```ts
import { registerSyncHandler } from "../../sync/syncEngine";
import { apiClient } from "../../api/apiClient";
import { DailyActivityRepository } from "../../db/repositories/DailyActivityRepository";

export function registerActivitySyncHandler(): void {
  registerSyncHandler("daily_activity", async (item) => {
    const payload = JSON.parse(item.payload);
    await apiClient.post("/activity/sync", payload);
    await DailyActivityRepository.markSynced(item.entity_id);
  });
}
```

`app/_layout.tsx`-এ যোগ করো:

```ts
import { registerActivitySyncHandler } from "../src/features/activity/registerActivitySync";
registerActivitySyncHandler();
```

---

## Step 9.6 — Session Timer Hook (Practice সময় ট্র্যাক করা)

`src/features/activity/useSessionTimer.ts`

```ts
import { useEffect, useRef } from "react";
import { DailyActivityRepository } from "../../db/repositories/DailyActivityRepository";
import { SyncQueueRepository } from "../../db/repositories/SyncQueueRepository";

// কোনো screen mount হলে টাইমার শুরু, unmount হলে elapsed minutes সেভ হয়
export function useSessionTimer() {
  const startRef = useRef<number>(Date.now());

  useEffect(() => {
    startRef.current = Date.now();

    return () => {
      const elapsedMs = Date.now() - startRef.current;
      const elapsedMinutes = Math.max(1, Math.round(elapsedMs / 60000)); // ন্যূনতম ১ মিনিট গণনা

      DailyActivityRepository.addMinutesToday(elapsedMinutes).then((row) => {
        SyncQueueRepository.enqueue("daily_activity", row.date, "update", {
          date: row.date,
          minutes: row.minutes,
        });
      });
    };
  }, []);
}
```

**ব্যবহার:** Learn flow (`learn.tsx`), Quiz (`quiz.tsx`), Flashcards (`flashcards.tsx`), Daily Practice (`practice/index.tsx`) — প্রতিটার শুরুতে একলাইন যোগ করো:

```tsx
useSessionTimer();
```

---

## Step 9.7 — Statistics Aggregation

`src/db/repositories/ProgressRepository.ts`-এ যোগ করো:

```ts
async countCompletedLessons(): Promise<number> {
  const db = await this.db();
  const row = await db.getFirstAsync<{ count: number }>(
    `SELECT COUNT(*) as count FROM progress WHERE status = 'completed'`
  );
  return row?.count ?? 0;
}
```

`src/db/repositories/SrsRepository.ts`-এ যোগ করো:

```ts
// level >= 3 মানে "মোটামুটি শেখা হয়ে গেছে" ধরা হচ্ছে
async countLearnedWords(): Promise<number> {
  const db = await this.db();
  const row = await db.getFirstAsync<{ count: number }>(
    'SELECT COUNT(*) as count FROM word_srs WHERE level >= 3'
  );
  return row?.count ?? 0;
}
```

`src/features/stats/useStatistics.ts`

```ts
import { useQuery } from "@tanstack/react-query";
import { ProgressRepository } from "../../db/repositories/ProgressRepository";
import { SrsRepository } from "../../db/repositories/SrsRepository";
import { DailyActivityRepository } from "../../db/repositories/DailyActivityRepository";
import { BookmarkRepository } from "../../db/repositories/BookmarkRepository";

export interface Statistics {
  completedLessons: number;
  learnedWords: number;
  totalMinutes: number;
  todayMinutes: number;
  streak: number;
  bookmarkedCount: number;
}

export function useStatistics() {
  return useQuery<Statistics>({
    queryKey: ["statistics", "local"],
    queryFn: async () => {
      const [
        completedLessons,
        learnedWords,
        totalMinutes,
        todayMinutes,
        streak,
        bookmarks,
      ] = await Promise.all([
        ProgressRepository.countCompletedLessons(),
        SrsRepository.countLearnedWords(),
        DailyActivityRepository.getTotalMinutes(),
        DailyActivityRepository.getTodayMinutes(),
        DailyActivityRepository.computeStreak(),
        BookmarkRepository.findAll(),
      ]);

      return {
        completedLessons,
        learnedWords,
        totalMinutes,
        todayMinutes,
        streak,
        bookmarkedCount: bookmarks.length,
      };
    },
  });
}
```

---

## Step 9.8 — Bookmark Button Component

`src/components/BookmarkButton.tsx`

```tsx
import { Pressable, Text } from "react-native";
import { useState, useEffect } from "react";
import { BookmarkRepository } from "../db/repositories/BookmarkRepository";
import { toggleBookmark } from "../features/bookmarks/bookmarkActions";

interface Props {
  itemType: "word" | "sentence" | "lesson";
  itemId: string;
}

export function BookmarkButton({ itemType, itemId }: Props) {
  const [bookmarked, setBookmarked] = useState(false);

  useEffect(() => {
    BookmarkRepository.isBookmarked(itemType, itemId).then(setBookmarked);
  }, [itemType, itemId]);

  const handleToggle = async () => {
    const newState = await toggleBookmark(itemType, itemId);
    setBookmarked(newState);
  };

  return (
    <Pressable
      onPress={handleToggle}
      className="w-10 h-10 items-center justify-center"
    >
      <Text className="text-2xl">{bookmarked ? "★" : "☆"}</Text>
    </Pressable>
  );
}
```

Word Detail screen-এ (Step 5.18) যোগ করো:

```tsx
import { BookmarkButton } from "../../../src/components/BookmarkButton";

// audio buttons-এর পাশে:
<BookmarkButton itemType="word" itemId={word.id} />;
```

---

## Step 9.9 — Bookmarks Screen

`app/(app)/bookmarks/index.tsx`

```tsx
import { View, Text, FlatList, Pressable } from "react-native";
import { useEffect, useState } from "react";
import { router } from "expo-router";
import {
  BookmarkRepository,
  BookmarkRow,
} from "../../../src/db/repositories/BookmarkRepository";
import {
  WordRepository,
  WordRow,
} from "../../../src/db/repositories/WordRepository";

export default function BookmarksScreen() {
  const [bookmarks, setBookmarks] = useState<BookmarkRow[]>([]);
  const [words, setWords] = useState<Record<string, WordRow>>({});

  useEffect(() => {
    BookmarkRepository.findByType("word").then(async (rows) => {
      setBookmarks(rows);
      const wordMap: Record<string, WordRow> = {};
      for (const b of rows) {
        const word = await WordRepository.findById(b.item_id);
        if (word) wordMap[b.item_id] = word;
      }
      setWords(wordMap);
    });
  }, []);

  if (bookmarks.length === 0) {
    return (
      <View className="flex-1 items-center justify-center bg-white dark:bg-gray-900 px-6">
        <Text className="text-gray-400 text-center">
          এখনো কোনো শব্দ বুকমার্ক করা হয়নি।
        </Text>
      </View>
    );
  }

  return (
    <FlatList
      className="flex-1 bg-white dark:bg-gray-900"
      data={bookmarks}
      keyExtractor={(item) => item.id}
      contentContainerStyle={{ padding: 16 }}
      renderItem={({ item }) => {
        const word = words[item.item_id];
        if (!word) return null;
        return (
          <Pressable
            onPress={() => router.push(`/words/${word.id}`)}
            className="bg-gray-100 dark:bg-gray-800 rounded-xl p-4 mb-3 flex-row justify-between items-center"
          >
            <View>
              <Text className="text-lg font-semibold text-gray-900 dark:text-white">
                {word.bangla_meaning}
              </Text>
              <Text className="text-gray-500 dark:text-gray-400">
                {word.pronunciation}
              </Text>
            </View>
            <Text className="text-2xl">{word.arabic}</Text>
          </Pressable>
        );
      }}
    />
  );
}
```

---

## Step 9.10 — Home / Statistics Screen

`app/(app)/home/index.tsx`

```tsx
import { View, Text, ScrollView } from "react-native";
import { useStatistics } from "../../../src/features/stats/useStatistics";
import { ProgressRepository } from "../../../src/db/repositories/ProgressRepository";
import { useEffect, useState } from "react";

const DAILY_GOAL_MINUTES = 10; // Step 9.11-এ User profile থেকে dynamic হবে

export default function HomeScreen() {
  const { data: stats, isLoading } = useStatistics();
  const [recent, setRecent] = useState<
    Awaited<ReturnType<typeof ProgressRepository.getRecentlyPracticed>>
  >([]);

  useEffect(() => {
    ProgressRepository.getRecentlyPracticed(5).then(setRecent);
  }, []);

  if (isLoading || !stats) {
    return (
      <View className="flex-1 items-center justify-center bg-white dark:bg-gray-900">
        <Text className="text-gray-400">লোড হচ্ছে...</Text>
      </View>
    );
  }

  const goalProgress = Math.min(1, stats.todayMinutes / DAILY_GOAL_MINUTES);

  return (
    <ScrollView className="flex-1 bg-white dark:bg-gray-900 px-6 py-8">
      {/* Streak */}
      <View className="bg-orange-100 dark:bg-orange-900 rounded-2xl p-5 mb-4 flex-row items-center justify-between">
        <View>
          <Text className="text-orange-600 dark:text-orange-300 text-sm">
            দৈনিক স্ট্রিক
          </Text>
          <Text className="text-3xl font-bold text-orange-700 dark:text-orange-200">
            🔥 {stats.streak} দিন
          </Text>
        </View>
      </View>

      {/* Daily Goal */}
      <View className="bg-gray-100 dark:bg-gray-800 rounded-2xl p-5 mb-4">
        <Text className="text-gray-500 dark:text-gray-400 text-sm mb-2">
          আজকের লক্ষ্য: {stats.todayMinutes}/{DAILY_GOAL_MINUTES} মিনিট
        </Text>
        <View className="h-2 bg-gray-200 dark:bg-gray-700 rounded-full overflow-hidden">
          <View
            className="h-2 bg-emerald-500"
            style={{ width: `${goalProgress * 100}%` }}
          />
        </View>
      </View>

      {/* Stats Grid */}
      <View className="flex-row flex-wrap gap-3 mb-6">
        <View className="flex-1 min-w-[45%] bg-blue-50 dark:bg-blue-950 rounded-xl p-4">
          <Text className="text-2xl font-bold text-blue-600 dark:text-blue-300">
            {stats.completedLessons}
          </Text>
          <Text className="text-gray-500 dark:text-gray-400 text-sm">
            সম্পন্ন লেসন
          </Text>
        </View>
        <View className="flex-1 min-w-[45%] bg-purple-50 dark:bg-purple-950 rounded-xl p-4">
          <Text className="text-2xl font-bold text-purple-600 dark:text-purple-300">
            {stats.learnedWords}
          </Text>
          <Text className="text-gray-500 dark:text-gray-400 text-sm">
            শেখা শব্দ
          </Text>
        </View>
        <View className="flex-1 min-w-[45%] bg-emerald-50 dark:bg-emerald-950 rounded-xl p-4">
          <Text className="text-2xl font-bold text-emerald-600 dark:text-emerald-300">
            {stats.totalMinutes}
          </Text>
          <Text className="text-gray-500 dark:text-gray-400 text-sm">
            মোট মিনিট
          </Text>
        </View>
        <View className="flex-1 min-w-[45%] bg-yellow-50 dark:bg-yellow-950 rounded-xl p-4">
          <Text className="text-2xl font-bold text-yellow-600 dark:text-yellow-300">
            {stats.bookmarkedCount}
          </Text>
          <Text className="text-gray-500 dark:text-gray-400 text-sm">
            বুকমার্ক
          </Text>
        </View>
      </View>

      {/* Recently Learned */}
      <Text className="text-lg font-semibold text-gray-900 dark:text-white mb-3">
        সম্প্রতি চর্চা করা
      </Text>
      {recent.length === 0 ? (
        <Text className="text-gray-400">এখনো কিছু চর্চা করা হয়নি।</Text>
      ) : (
        recent.map((r) => (
          <View
            key={r.id}
            className="bg-gray-100 dark:bg-gray-800 rounded-xl p-4 mb-2"
          >
            <Text className="text-gray-900 dark:text-white">{r.title_bn}</Text>
            <Text className="text-gray-500 dark:text-gray-400 text-xs">
              {r.status === "completed" ? "সম্পন্ন ✓" : "চলমান"}
            </Text>
          </View>
        ))
      )}
    </ScrollView>
  );
}
```

---

**টেস্ট করো:**

1. একটা lesson সম্পূর্ণ করো → Home screen-এ streak `১ দিন`, completed lesson count বাড়া উচিত
2. Word Detail-এ star বাটনে চাপো → Bookmarks screen-এ সেই word দেখা উচিত
3. App বন্ধ করে পরদিন খোলো (বা device date এক দিন এগিয়ে দাও) — আজকে কিছু practice না করলে streak না বাড়া উচিত, কিন্তু গতকাল practice করা থাকলে streak ভাঙা উচিত না যতক্ষণ না আজকের দিনও চলে যায়
4. Airplane mode-এ সব করে Internet অন করো — bookmark ও daily_activity sync_queue প্রসেস হয়ে backend-এ যাওয়া উচিত (Compass-এ verify করো)

---

## এই Step-এ যা হলো (সারসংক্ষেপ)

- Bookmarks/Favorites — একটাই unified system, duplicate logic এড়ানো হয়েছে
- Daily Activity — idempotent "set minutes for date" pattern দিয়ে streak/stats-এর ভিত্তি
- Session timer hook — যেকোনো learning screen-এ এক লাইনে time tracking যোগ করা যায়
- Streak calculation — আজ practice না করলেও গতকাল পর্যন্ত grace period আছে (হঠাৎ ভেঙে যাওয়ার হতাশা কমায়)
- Statistics screen — completed lessons, learned words, total minutes, bookmarks সব এক জায়গায়, সম্পূর্ণ local aggregation (offline কাজ করে)

**ব্রুটাল ট্রুথ:** `DAILY_GOAL_MINUTES` এখন hardcoded (১০)। Step 3-এ User profile-এ `dailyGoalMinutes` field আছে, কিন্তু সেটা এখনো mobile-এ local cache/fetch করে Home screen-এ ব্যবহার করা হয়নি — এটা ছোট কিন্তু গুরুত্বপূর্ণ gap, পরের ছোট ধাপ হিসেবে ঠিক করা উচিত (auth store-এ profile fetch করে dailyGoalMinutes বসানো)।

**পরবর্তী যৌক্তিক ধাপ:** Step 10 — Polish & Performance (Dark Mode, Animations, Loading states, Accessibility, Optimization), অথবা আগে ছোট `dailyGoalMinutes` gap-টা ঠিক করে নেওয়া।
