# Step 7: Learning Flow (Lesson → Word → Sentence → Conversation → Progress)

এখন পর্যন্ত বানানো সব piece (SQLite, sync engine, audio) এখানে একটা guided learning experience-এ যুক্ত হবে। দুইটা নতুন জিনিস দরকার: **Conversation** content (নতুন schema) আর **Progress tracking**-এর আসল implementation (Step 4.7-এ sync engine skeleton বানানো হয়েছিল, এখন সেটা কাজে লাগানো হবে)।

---

# BACKEND

## Step 7.1 — Conversation MongoDB Schema

Conversation-এর প্রতিটা লাইন আলাদা document না করে একটা lesson-এর পুরো dialogue একটাই document-এ রাখা হচ্ছে — কারণ conversation lines সবসময় একসাথে পড়া/দেখানো হয়, আলাদা করে query করার দরকার নেই। এটা "avoid overengineering" নীতির সাথে সামঞ্জস্যপূর্ণ।

`src/models/Conversation.ts`

```ts
import { Schema, model, Types } from "mongoose";

export interface IConversationLine {
  speaker: string; // যেমন "A" / "B" অথবা "সাকিব" / "রাফি"
  arabic: string;
  transliteration: string;
  banglaMeaning: string;
  englishMeaning: string;
  audioUrl?: string;
}

export interface IConversation {
  _id: Types.ObjectId;
  lessonId: Types.ObjectId;
  titleAr: string;
  titleBn: string;
  titleEn: string;
  lines: IConversationLine[];
  createdAt: Date;
  updatedAt: Date;
}

const conversationLineSchema = new Schema<IConversationLine>(
  {
    speaker: { type: String, required: true },
    arabic: { type: String, required: true },
    transliteration: { type: String, required: true },
    banglaMeaning: { type: String, required: true },
    englishMeaning: { type: String, required: true },
    audioUrl: { type: String },
  },
  { _id: false },
);

const conversationSchema = new Schema<IConversation>(
  {
    lessonId: {
      type: Schema.Types.ObjectId,
      ref: "Lesson",
      required: true,
      index: true,
    },
    titleAr: { type: String, required: true },
    titleBn: { type: String, required: true },
    titleEn: { type: String, required: true },
    lines: { type: [conversationLineSchema], required: true },
  },
  { timestamps: true },
);

export const ConversationModel = model<IConversation>(
  "Conversation",
  conversationSchema,
);
```

---

## Step 7.2 — Repository / Service / Controller / Routes

`src/repositories/ConversationRepository.ts`

```ts
import { ConversationModel, IConversation } from "../models/Conversation";

export const ConversationRepository = {
  findByLesson: (lessonId: string) =>
    ConversationModel.find({ lessonId }).lean(),
  create: (data: Partial<IConversation>) => ConversationModel.create(data),
};
```

`src/services/conversationService.ts`

```ts
import { ConversationRepository } from "../repositories/ConversationRepository";

export const conversationService = {
  async getByLesson(lessonId: string) {
    return ConversationRepository.findByLesson(lessonId);
  },
};
```

`src/controllers/contentController.ts`-এ যোগ করো:

```ts
import { conversationService } from "../services/conversationService";

export const getConversationsByLesson = catchAsync(
  async (req: Request, res: Response) => {
    const { lessonId } = req.params;
    const conversations = await conversationService.getByLesson(lessonId);
    res.json({ success: true, data: conversations });
  },
);
```

`src/routes/contentRoutes.ts`-এ যোগ করো (validation একই `lessonIdParamSchema` reuse করছি):

```ts
import { getConversationsByLesson } from "../controllers/contentController";

router.get(
  "/lessons/:lessonId/conversations",
  validate(lessonIdParamSchema),
  getConversationsByLesson,
);
```

---

## Step 7.3 — Progress MongoDB Schema

`src/models/Progress.ts`

```ts
import { Schema, model, Types } from "mongoose";

export interface IProgress {
  _id: Types.ObjectId;
  supabaseUserId: string;
  lessonId: string;
  status: "not_started" | "in_progress" | "completed";
  score: number;
  lastPracticedAt: Date;
  createdAt: Date;
  updatedAt: Date;
}

const progressSchema = new Schema<IProgress>(
  {
    supabaseUserId: { type: String, required: true, index: true },
    lessonId: { type: String, required: true },
    status: {
      type: String,
      enum: ["not_started", "in_progress", "completed"],
      default: "not_started",
    },
    score: { type: Number, default: 0 },
    lastPracticedAt: { type: Date, default: Date.now },
  },
  { timestamps: true },
);

// একজন user-এর একটা lesson-এ একটাই progress document থাকবে
progressSchema.index({ supabaseUserId: 1, lessonId: 1 }, { unique: true });

export const ProgressModel = model<IProgress>("Progress", progressSchema);
```

`src/repositories/ProgressRepository.ts`

```ts
import { ProgressModel, IProgress } from "../models/Progress";

export const ProgressRepository = {
  upsert: (
    supabaseUserId: string,
    lessonId: string,
    data: Partial<IProgress>,
  ) =>
    ProgressModel.findOneAndUpdate(
      { supabaseUserId, lessonId },
      { $set: data },
      { new: true, upsert: true, setDefaultsOnInsert: true },
    ).lean(),

  findAllByUser: (supabaseUserId: string) =>
    ProgressModel.find({ supabaseUserId }).sort({ lastPracticedAt: -1 }).lean(),
};
```

`src/validations/progressValidation.ts`

```ts
import { z } from "zod";

export const syncProgressSchema = z.object({
  body: z.object({
    lessonId: z.string().min(1),
    status: z.enum(["not_started", "in_progress", "completed"]),
    score: z.coerce.number().min(0).default(0),
    lastPracticedAt: z.string().datetime(),
  }),
  params: z.object({}).optional(),
  query: z.object({}).optional(),
});
```

`src/services/progressService.ts`

```ts
import { ProgressRepository } from "../repositories/ProgressRepository";

interface SyncProgressInput {
  lessonId: string;
  status: "not_started" | "in_progress" | "completed";
  score: number;
  lastPracticedAt: string;
}

export const progressService = {
  // Idempotent upsert — sync queue বারবার retry করলেও সমস্যা নেই
  async syncProgress(supabaseUserId: string, input: SyncProgressInput) {
    return ProgressRepository.upsert(supabaseUserId, input.lessonId, {
      status: input.status,
      score: input.score,
      lastPracticedAt: new Date(input.lastPracticedAt),
    });
  },

  async getAllForUser(supabaseUserId: string) {
    return ProgressRepository.findAllByUser(supabaseUserId);
  },
};
```

`src/controllers/progressController.ts`

```ts
import { Request, Response } from "express";
import { catchAsync } from "../utils/catchAsync";
import { progressService } from "../services/progressService";

export const syncProgress = catchAsync(async (req: Request, res: Response) => {
  const { supabaseUserId } = req.user!;
  const progress = await progressService.syncProgress(supabaseUserId, req.body);
  res.json({ success: true, data: progress });
});

export const getMyProgress = catchAsync(async (req: Request, res: Response) => {
  const { supabaseUserId } = req.user!;
  const progress = await progressService.getAllForUser(supabaseUserId);
  res.json({ success: true, data: progress });
});
```

`src/routes/progressRoutes.ts`

```ts
import { Router } from "express";
import { requireAuth } from "../middleware/requireAuth";
import { validate } from "../middleware/validate";
import { syncProgressSchema } from "../validations/progressValidation";
import { syncProgress, getMyProgress } from "../controllers/progressController";

const router = Router();
router.use(requireAuth);

router.post("/sync", validate(syncProgressSchema), syncProgress);
router.get("/", getMyProgress);

export default router;
```

`src/app.ts`-এ:

```ts
import progressRoutes from "./routes/progressRoutes";
app.use("/api/v1/progress", progressRoutes);
```

---

## Step 7.4 — Conversation Seed Data (বাস্তব উদাহরণ)

Step 5.7-এর `seedContent.ts`-এ Greetings lesson-এর পরে যোগ করো:

```ts
import { ConversationModel } from "../models/Conversation";

// ... basicGreetings lesson তৈরি হওয়ার পরে:
await ConversationModel.create({
  lessonId: basicGreetings._id,
  titleAr: "محادثة عند اللقاء",
  titleBn: "দেখা হলে কথোপকথন",
  titleEn: "Meeting Conversation",
  lines: [
    {
      speaker: "A",
      arabic: "السلام عليكم",
      transliteration: "As-salamu alaykum",
      banglaMeaning: "আপনার উপর শান্তি বর্ষিত হোক",
      englishMeaning: "Peace be upon you",
    },
    {
      speaker: "B",
      arabic: "وعليكم السلام، كيف حالك؟",
      transliteration: "Wa alaykum as-salam, kayfa haluk?",
      banglaMeaning: "আপনার উপরও শান্তি, আপনি কেমন আছেন?",
      englishMeaning: "And peace be upon you too, how are you?",
    },
    {
      speaker: "A",
      arabic: "بخير، شكرا. وأنت؟",
      transliteration: "Bikhayr, shukran. Wa anta?",
      banglaMeaning: "ভালো আছি, ধন্যবাদ। আর আপনি?",
      englishMeaning: "I'm fine, thank you. And you?",
    },
    {
      speaker: "B",
      arabic: "الحمد لله، بخير",
      transliteration: "Alhamdulillah, bikhayr",
      banglaMeaning: "আলহামদুলিল্লাহ, ভালো আছি",
      englishMeaning: "Praise be to God, I'm fine",
    },
  ],
});
```

`npm run seed` আবার চালাও।

---

# MOBILE

## Step 7.5 — SQLite Migration: Conversations টেবিল

Lines array আলাদা টেবিলে normalize না করে JSON string হিসেবে রাখা হচ্ছে — এটা display-only, নেস্টেড রিলেশনাল query দরকার নেই।

`src/db/migrations/003_add_conversations.ts`

```ts
import { SQLiteDatabase } from "expo-sqlite";
import { Migration } from "./index";

export const migration_003_add_conversations: Migration = {
  version: 3,
  name: "add_conversations",
  up: async (db: SQLiteDatabase) => {
    await db.execAsync(`
      CREATE TABLE IF NOT EXISTS conversations (
        id TEXT PRIMARY KEY,
        lesson_id TEXT NOT NULL,
        title_ar TEXT NOT NULL,
        title_bn TEXT NOT NULL,
        title_en TEXT NOT NULL,
        lines_json TEXT NOT NULL,
        updated_at TEXT NOT NULL
      );
      CREATE INDEX IF NOT EXISTS idx_conversations_lesson ON conversations(lesson_id);
    `);
  },
};
```

`src/db/migrations/index.ts`-এ যোগ করো:

```ts
import { migration_003_add_conversations } from "./003_add_conversations";

export const migrations: Migration[] = [
  migration_001_init,
  migration_002_add_sentence_transliteration,
  migration_003_add_conversations, // যোগ করো
];
```

---

## Step 7.6 — Conversation: API + Repository + Sync + Hook

`src/api/contentApi.ts`-এ যোগ করো:

```ts
export interface ConversationLineDto {
  speaker: string;
  arabic: string;
  transliteration: string;
  banglaMeaning: string;
  englishMeaning: string;
  audioUrl?: string;
}

export interface ConversationDto {
  _id: string;
  lessonId: string;
  titleAr: string;
  titleBn: string;
  titleEn: string;
  lines: ConversationLineDto[];
}

// contentApi object-এর ভেতরে:
getConversationsByLesson: async (lessonId: string): Promise<ConversationDto[]> => {
  const { data } = await apiClient.get(`/content/lessons/${lessonId}/conversations`);
  return data.data;
},
```

`src/db/repositories/ConversationRepository.ts`

```ts
import { BaseRepository } from "./BaseRepository";
import { ConversationLineDto } from "../../api/contentApi";

export interface ConversationRow {
  id: string;
  lesson_id: string;
  title_ar: string;
  title_bn: string;
  title_en: string;
  lines_json: string;
  updated_at: string;
}

export interface ParsedConversation extends Omit<
  ConversationRow,
  "lines_json"
> {
  lines: ConversationLineDto[];
}

class ConversationRepositoryClass extends BaseRepository<ConversationRow> {
  protected tableName = "conversations";

  async findByLesson(lessonId: string): Promise<ParsedConversation[]> {
    const db = await this.db();
    const rows = await db.getAllAsync<ConversationRow>(
      "SELECT * FROM conversations WHERE lesson_id = ?",
      [lessonId],
    );
    return rows.map((r) => ({ ...r, lines: JSON.parse(r.lines_json) }));
  }

  async upsertMany(conversations: ConversationRow[]): Promise<void> {
    const db = await this.db();
    await db.withTransactionAsync(async () => {
      for (const c of conversations) {
        await db.runAsync(
          `INSERT INTO conversations (id, lesson_id, title_ar, title_bn, title_en, lines_json, updated_at)
           VALUES (?, ?, ?, ?, ?, ?, ?)
           ON CONFLICT(id) DO UPDATE SET
             title_ar=excluded.title_ar, title_bn=excluded.title_bn, title_en=excluded.title_en,
             lines_json=excluded.lines_json, updated_at=excluded.updated_at`,
          [
            c.id,
            c.lesson_id,
            c.title_ar,
            c.title_bn,
            c.title_en,
            c.lines_json,
            c.updated_at,
          ],
        );
      }
    });
  }
}

export const ConversationRepository = new ConversationRepositoryClass();
```

`src/features/conversations/conversationSync.ts`

```ts
import { contentApi } from "../../api/contentApi";
import { ConversationRepository } from "../../db/repositories/ConversationRepository";

export async function syncConversations(lessonId: string): Promise<void> {
  const remote = await contentApi.getConversationsByLesson(lessonId);
  const rows = remote.map((c) => ({
    id: c._id,
    lesson_id: c.lessonId,
    title_ar: c.titleAr,
    title_bn: c.titleBn,
    title_en: c.titleEn,
    lines_json: JSON.stringify(c.lines),
    updated_at: new Date().toISOString(),
  }));
  await ConversationRepository.upsertMany(rows);
}
```

`src/features/conversations/useConversations.ts`

```ts
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { useEffect } from "react";
import { ConversationRepository } from "../../db/repositories/ConversationRepository";
import { syncConversations } from "./conversationSync";
import { useNetworkStore } from "../../store/networkStore";

export function useConversations(lessonId: string) {
  const queryClient = useQueryClient();
  const isOnline = useNetworkStore((s) => s.isOnline);

  const query = useQuery({
    queryKey: ["conversations", "local", lessonId],
    queryFn: () => ConversationRepository.findByLesson(lessonId),
    enabled: !!lessonId,
  });

  useEffect(() => {
    if (!isOnline || !lessonId) return;
    syncConversations(lessonId)
      .then(() =>
        queryClient.invalidateQueries({
          queryKey: ["conversations", "local", lessonId],
        }),
      )
      .catch((err) => console.warn("Conversation sync failed:", err));
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [isOnline, lessonId]);

  return query;
}
```

---

## Step 7.7 — Progress: Repository + Outbox Wiring

`src/db/repositories/ProgressRepository.ts`

```ts
import { BaseRepository } from "./BaseRepository";
import { randomUUID } from "expo-crypto";

export interface ProgressRow {
  id: string;
  lesson_id: string;
  status: "not_started" | "in_progress" | "completed";
  score: number;
  last_practiced_at: string | null;
  is_synced: number;
}

class ProgressRepositoryClass extends BaseRepository<ProgressRow> {
  protected tableName = "progress";

  async findByLesson(lessonId: string): Promise<ProgressRow | null> {
    const db = await this.db();
    const row = await db.getFirstAsync<ProgressRow>(
      "SELECT * FROM progress WHERE lesson_id = ?",
      [lessonId],
    );
    return row ?? null;
  }

  async upsertLocal(
    lessonId: string,
    status: ProgressRow["status"],
    score: number,
  ): Promise<ProgressRow> {
    const db = await this.db();
    const now = new Date().toISOString();
    const existing = await this.findByLesson(lessonId);

    const id = existing?.id ?? randomUUID();

    if (existing) {
      await db.runAsync(
        `UPDATE progress SET status = ?, score = ?, last_practiced_at = ?, is_synced = 0 WHERE id = ?`,
        [status, score, now, id],
      );
    } else {
      await db.runAsync(
        `INSERT INTO progress (id, lesson_id, status, score, last_practiced_at, is_synced)
         VALUES (?, ?, ?, ?, ?, 0)`,
        [id, lessonId, status, score, now],
      );
    }

    return {
      id,
      lesson_id: lessonId,
      status,
      score,
      last_practiced_at: now,
      is_synced: 0,
    };
  }

  async markSynced(id: string): Promise<void> {
    const db = await this.db();
    await db.runAsync("UPDATE progress SET is_synced = 1 WHERE id = ?", [id]);
  }

  // Recently Learned — Step 9-এর Home screen-এ কাজে লাগবে
  async getRecentlyPracticed(
    limit = 5,
  ): Promise<(ProgressRow & { title_bn: string })[]> {
    const db = await this.db();
    return db.getAllAsync<ProgressRow & { title_bn: string }>(
      `SELECT p.*, l.title_bn FROM progress p
       JOIN lessons l ON l.id = p.lesson_id
       WHERE p.last_practiced_at IS NOT NULL
       ORDER BY p.last_practiced_at DESC
       LIMIT ?`,
      [limit],
    );
  }
}

export const ProgressRepository = new ProgressRepositoryClass();
```

`src/features/progress/progressActions.ts`

```ts
import { ProgressRepository } from "../../db/repositories/ProgressRepository";
import { SyncQueueRepository } from "../../db/repositories/SyncQueueRepository";

export async function markLessonStarted(lessonId: string): Promise<void> {
  const existing = await ProgressRepository.findByLesson(lessonId);
  if (existing?.status === "completed") return; // completed-কে আবার in_progress করবো না

  const row = await ProgressRepository.upsertLocal(
    lessonId,
    "in_progress",
    existing?.score ?? 0,
  );
  await enqueueProgressSync(row);
}

export async function markLessonCompleted(
  lessonId: string,
  score: number,
): Promise<void> {
  const row = await ProgressRepository.upsertLocal(
    lessonId,
    "completed",
    score,
  );
  await enqueueProgressSync(row);
}

async function enqueueProgressSync(row: {
  id: string;
  lesson_id: string;
  status: string;
  score: number;
  last_practiced_at: string | null;
}): Promise<void> {
  await SyncQueueRepository.enqueue("progress", row.id, "update", {
    lessonId: row.lesson_id,
    status: row.status,
    score: row.score,
    lastPracticedAt: row.last_practiced_at,
  });
}
```

## Step 7.8 — Sync Handler রেজিস্টার করা (Step 4.7 skeleton-এর আসল ব্যবহার)

`src/features/progress/registerProgressSync.ts`

```ts
import { registerSyncHandler } from "../../sync/syncEngine";
import { apiClient } from "../../api/apiClient";
import { ProgressRepository } from "../../db/repositories/ProgressRepository";

export function registerProgressSyncHandler(): void {
  registerSyncHandler("progress", async (item) => {
    const payload = JSON.parse(item.payload);
    await apiClient.post("/progress/sync", payload);
    await ProgressRepository.markSynced(item.entity_id);
  });
}
```

`app/_layout.tsx`-এ একবার কল করো (component-এর বাইরে, module load সময়ে):

```ts
import { registerProgressSyncHandler } from "../src/features/progress/registerProgressSync";
registerProgressSyncHandler();
```

---

## Step 7.9 — Lesson Player (মূল Learning Flow)

Word → Word → ... → Conversation → সম্পন্ন, ধাপে ধাপে।

`app/(app)/lessons/[lessonId]/learn.tsx`

```tsx
import { View, Text, Pressable, ScrollView } from "react-native";
import { useState, useEffect, useMemo } from "react";
import { useLocalSearchParams, router } from "expo-router";
import { useWords } from "../../../../src/features/words/useWords";
import { useConversations } from "../../../../src/features/conversations/useConversations";
import {
  markLessonStarted,
  markLessonCompleted,
} from "../../../../src/features/progress/progressActions";
import { AudioPlayButton } from "../../../../src/components/AudioPlayButton";

type Step =
  | { type: "word"; index: number }
  | { type: "conversation" }
  | { type: "done" };

export default function LessonPlayerScreen() {
  const { lessonId } = useLocalSearchParams<{ lessonId: string }>();
  const { data: words, isLoading: wordsLoading } = useWords(lessonId);
  const { data: conversations } = useConversations(lessonId);

  const [stepIndex, setStepIndex] = useState(0);

  useEffect(() => {
    markLessonStarted(lessonId);
  }, [lessonId]);

  const steps: Step[] = useMemo(() => {
    const wordSteps: Step[] = (words ?? []).map((_, i) => ({
      type: "word",
      index: i,
    }));
    const convoSteps: Step[] =
      (conversations ?? []).length > 0 ? [{ type: "conversation" }] : [];
    return [...wordSteps, ...convoSteps, { type: "done" }];
  }, [words, conversations]);

  const currentStep = steps[stepIndex];

  const goNext = async () => {
    if (stepIndex >= steps.length - 1) return;

    if (currentStep?.type === "done" || steps[stepIndex + 1]?.type === "done") {
      await markLessonCompleted(lessonId, 100);
    }
    setStepIndex((i) => i + 1);
  };

  const goPrev = () => setStepIndex((i) => Math.max(0, i - 1));

  if (wordsLoading || !currentStep) {
    return (
      <View className="flex-1 items-center justify-center bg-white dark:bg-gray-900">
        <Text className="text-gray-400">লোড হচ্ছে...</Text>
      </View>
    );
  }

  return (
    <View className="flex-1 bg-white dark:bg-gray-900">
      {/* Progress bar */}
      <View className="h-1 bg-gray-100 dark:bg-gray-800">
        <View
          className="h-1 bg-emerald-500"
          style={{ width: `${((stepIndex + 1) / steps.length) * 100}%` }}
        />
      </View>

      <ScrollView contentContainerStyle={{ flexGrow: 1, padding: 24 }}>
        {currentStep.type === "word" && words && (
          <View className="flex-1 items-center justify-center">
            <Text className="text-5xl text-gray-900 dark:text-white mb-4">
              {words[currentStep.index].arabic}
            </Text>
            <Text className="text-lg text-gray-500 dark:text-gray-400 mb-2">
              {words[currentStep.index].pronunciation}
            </Text>
            <Text className="text-xl text-gray-900 dark:text-white mb-1">
              {words[currentStep.index].bangla_meaning}
            </Text>
            <Text className="text-base text-gray-500 dark:text-gray-400 mb-6">
              {words[currentStep.index].english_meaning}
            </Text>
            <AudioPlayButton
              itemId={words[currentStep.index].id}
              audioUrl={words[currentStep.index].audio_url ?? undefined}
            />
          </View>
        )}

        {currentStep.type === "conversation" &&
          conversations &&
          conversations[0] && (
            <View>
              <Text className="text-xl font-semibold text-gray-900 dark:text-white mb-4">
                {conversations[0].title_bn}
              </Text>
              {conversations[0].lines.map((line, i) => (
                <View
                  key={i}
                  className={`mb-3 p-3 rounded-xl max-w-[85%] ${
                    line.speaker === "A"
                      ? "bg-emerald-100 dark:bg-emerald-900 self-start"
                      : "bg-gray-100 dark:bg-gray-800 self-end"
                  }`}
                >
                  <Text className="text-lg text-right text-gray-900 dark:text-white">
                    {line.arabic}
                  </Text>
                  <Text className="text-gray-500 dark:text-gray-400">
                    {line.transliteration}
                  </Text>
                  <Text className="text-gray-900 dark:text-white">
                    {line.banglaMeaning}
                  </Text>
                </View>
              ))}
            </View>
          )}

        {currentStep.type === "done" && (
          <View className="flex-1 items-center justify-center">
            <Text className="text-2xl font-semibold text-emerald-600 mb-2">
              🎉 সম্পন্ন হয়েছে!
            </Text>
            <Text className="text-gray-500 dark:text-gray-400 mb-6">
              এই লেসনটা তুমি সফলভাবে শেষ করেছো।
            </Text>
            <Pressable
              onPress={() => router.back()}
              className="bg-emerald-500 rounded-xl px-6 py-3"
            >
              <Text className="text-white font-semibold">
                লেসন লিস্টে ফিরে যাও
              </Text>
            </Pressable>
          </View>
        )}
      </ScrollView>

      {currentStep.type !== "done" && (
        <View className="flex-row justify-between px-6 py-4 border-t border-gray-100 dark:border-gray-800">
          <Pressable
            onPress={goPrev}
            disabled={stepIndex === 0}
            className="px-6 py-3"
          >
            <Text
              className={
                stepIndex === 0
                  ? "text-gray-300"
                  : "text-gray-700 dark:text-gray-300"
              }
            >
              পেছনে
            </Text>
          </Pressable>
          <Pressable
            onPress={goNext}
            className="bg-emerald-500 rounded-xl px-6 py-3"
          >
            <Text className="text-white font-semibold">পরবর্তী</Text>
          </Pressable>
        </View>
      )}
    </View>
  );
}
```

---

## Step 7.10 — Lessons Screen থেকে Learn Flow-এ ঢোকা

Step 5.16-এর `app/(app)/categories/[categoryId]/lessons.tsx`-এ প্রতিটা row-তে দুইটা action দাও:

```tsx
<Pressable
  onPress={() => router.push(`/lessons/${item.id}/learn`)}
  className="bg-gray-100 dark:bg-gray-800 rounded-xl p-4 mb-3"
>
  <Text className="text-lg font-semibold text-gray-900 dark:text-white">
    {item.title_bn}
  </Text>
  <Text className="text-gray-500 dark:text-gray-400">{item.title_en}</Text>
  <Pressable
    onPress={() => router.push(`/lessons/${item.id}/words`)}
    className="mt-2"
  >
    <Text className="text-sm text-emerald-600">শব্দ তালিকা দেখো →</Text>
  </Pressable>
</Pressable>
```

Primary action (পুরো row ট্যাপ) → Learn flow শুরু হয়। Secondary link → Step 5-এর browse screen (reference lুকআপের জন্য এখনো দরকারি)।

---

**টেস্ট করো:**

1. একটা lesson-এ "Learn" flow শুরু করো — word-by-word এগোয়, শেষে conversation দেখায়, তারপর "সম্পন্ন হয়েছে" স্ক্রিন
2. MongoDB Compass-এ `progress` collection চেক করো — status `completed` আর score `100` সহ document তৈরি হয়েছে কিনা
3. Airplane mode অন করে পুরো flow আবার করো — progress local-এ সেভ হওয়া উচিত, sync_queue-তে entry যোগ হওয়া উচিত। তারপর internet অন করো — queue প্রসেস হয়ে backend-এ sync হওয়া উচিত (Compass-এ ভেরিফাই করো)

---

## এই Step-এ যা হলো (সারসংক্ষেপ)

- Conversation content — নতুন schema, seed data, sync pattern (আগের প্যাটার্নই অনুসরণ করা হয়েছে)
- Progress tracking backend — idempotent upsert, unique (user, lesson) constraint
- **Step 4.7-এর sync engine skeleton আসলেই কাজে লাগানো হলো** — `registerSyncHandler('progress', ...)` দিয়ে outbox pattern প্রথমবার বাস্তবে ব্যবহৃত হলো
- Lesson Player — word-by-word guided flow, conversation দেখানো, সম্পূর্ণ হলে progress mark করা — সব অফলাইনেও কাজ করে
- `getRecentlyPracticed()` — Step 9-এর Home/Dashboard screen-এর জন্য আগে থেকেই প্রস্তুত করা হলো

**ব্রুটাল ট্রুথ:** Score system এখন খুবই সরল — সম্পূর্ণ করলেই ফিক্সড ১০০ দেওয়া হচ্ছে, actual comprehension measure করা হচ্ছে না। Step 8 (Quiz & Revision)-এ MCQ/flashcard performance থেকে প্রকৃত score আসবে — তখন এই ফিক্সড ১০০ logic-টা replace করতে হবে real quiz score দিয়ে।

**পরবর্তী যৌক্তিক ধাপ:** Step 8 — Quiz & Revision (Flashcards, MCQ, Daily Practice, Spaced Repetition basics), যেটা এখনকার ফিক্সড completion score-কে real, measurable score-এ পরিণত করবে।
