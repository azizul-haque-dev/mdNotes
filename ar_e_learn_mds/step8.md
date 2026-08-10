# Step 8: Quiz & Revision (Flashcards, MCQ, Spaced Repetition, Daily Practice)

**Architecture decision:** Full SM-2 spaced repetition algorithm (ease factor, variable intervals) হলো overengineering এই app-এর scope-এর জন্য। তার বদলে সহজ **Leitner-style level system** ব্যবহার করছি — সঠিক উত্তর দিলে level বাড়ে (interval বড় হয়), ভুল দিলে level কমে। এটাই "spaced repetition basics" requirement যথেষ্টভাবে পূরণ করে।

MCQ/Flashcard সম্পূর্ণ **client-side generate** হবে already-synced local SQLite word data থেকে — নতুন কোনো backend endpoint লাগবে না কুইজের জন্য, শুধু SRS state sync-এর জন্য লাগবে (Progress-এর মতোই outbox pattern)।

---

# BACKEND

## Step 8.1 — SRS (Spaced Repetition State) Schema

`src/models/WordSrs.ts`

```ts
import { Schema, model, Types } from "mongoose";

export interface IWordSrs {
  _id: Types.ObjectId;
  supabaseUserId: string;
  wordId: string;
  level: number; // 0-5
  nextReviewAt: Date;
  lastReviewedAt: Date;
  createdAt: Date;
  updatedAt: Date;
}

const wordSrsSchema = new Schema<IWordSrs>(
  {
    supabaseUserId: { type: String, required: true, index: true },
    wordId: { type: String, required: true },
    level: { type: Number, default: 0, min: 0, max: 5 },
    nextReviewAt: { type: Date, required: true },
    lastReviewedAt: { type: Date, default: Date.now },
  },
  { timestamps: true },
);

wordSrsSchema.index({ supabaseUserId: 1, wordId: 1 }, { unique: true });

export const WordSrsModel = model<IWordSrs>("WordSrs", wordSrsSchema);
```

`src/repositories/SrsRepository.ts`

```ts
import { WordSrsModel, IWordSrs } from "../models/WordSrs";

export const SrsRepository = {
  upsert: (supabaseUserId: string, wordId: string, data: Partial<IWordSrs>) =>
    WordSrsModel.findOneAndUpdate(
      { supabaseUserId, wordId },
      { $set: data },
      { new: true, upsert: true, setDefaultsOnInsert: true },
    ).lean(),
};
```

`src/validations/srsValidation.ts`

```ts
import { z } from "zod";

export const syncSrsSchema = z.object({
  body: z.object({
    wordId: z.string().min(1),
    level: z.coerce.number().int().min(0).max(5),
    nextReviewAt: z.string().datetime(),
    lastReviewedAt: z.string().datetime(),
  }),
  params: z.object({}).optional(),
  query: z.object({}).optional(),
});
```

`src/services/srsService.ts`

```ts
import { SrsRepository } from "../repositories/SrsRepository";

interface SyncSrsInput {
  wordId: string;
  level: number;
  nextReviewAt: string;
  lastReviewedAt: string;
}

export const srsService = {
  async syncSrs(supabaseUserId: string, input: SyncSrsInput) {
    return SrsRepository.upsert(supabaseUserId, input.wordId, {
      level: input.level,
      nextReviewAt: new Date(input.nextReviewAt),
      lastReviewedAt: new Date(input.lastReviewedAt),
    });
  },
};
```

`src/controllers/srsController.ts`

```ts
import { Request, Response } from "express";
import { catchAsync } from "../utils/catchAsync";
import { srsService } from "../services/srsService";

export const syncSrs = catchAsync(async (req: Request, res: Response) => {
  const { supabaseUserId } = req.user!;
  const result = await srsService.syncSrs(supabaseUserId, req.body);
  res.json({ success: true, data: result });
});
```

`src/routes/srsRoutes.ts`

```ts
import { Router } from "express";
import { requireAuth } from "../middleware/requireAuth";
import { validate } from "../middleware/validate";
import { syncSrsSchema } from "../validations/srsValidation";
import { syncSrs } from "../controllers/srsController";

const router = Router();
router.use(requireAuth);
router.post("/sync", validate(syncSrsSchema), syncSrs);

export default router;
```

`src/app.ts`-এ:

```ts
import srsRoutes from "./routes/srsRoutes";
app.use("/api/v1/srs", srsRoutes);
```

---

# MOBILE

## Step 8.2 — SQLite Migration: SRS টেবিল

`src/db/migrations/004_add_srs.ts`

```ts
import { SQLiteDatabase } from "expo-sqlite";
import { Migration } from "./index";

export const migration_004_add_srs: Migration = {
  version: 4,
  name: "add_srs",
  up: async (db: SQLiteDatabase) => {
    await db.execAsync(`
      CREATE TABLE IF NOT EXISTS word_srs (
        word_id TEXT PRIMARY KEY,
        level INTEGER NOT NULL DEFAULT 0,
        next_review_at TEXT NOT NULL,
        last_reviewed_at TEXT NOT NULL,
        is_synced INTEGER DEFAULT 0
      );
      CREATE INDEX IF NOT EXISTS idx_srs_next_review ON word_srs(next_review_at);
    `);
  },
};
```

`src/db/migrations/index.ts`-এ যোগ করো:

```ts
import { migration_004_add_srs } from "./004_add_srs";

export const migrations: Migration[] = [
  migration_001_init,
  migration_002_add_sentence_transliteration,
  migration_003_add_conversations,
  migration_004_add_srs, // যোগ করো
];
```

---

## Step 8.3 — Spaced Repetition Algorithm (Leitner-style)

`src/srs/spacedRepetition.ts`

```ts
// Level অনুযায়ী পরবর্তী review কতদিন পরে হবে (দিনে)
const INTERVAL_DAYS = [0, 1, 3, 7, 14, 30]; // index = level (0-5)

export function computeNextReview(currentLevel: number, isCorrect: boolean) {
  const newLevel = isCorrect
    ? Math.min(currentLevel + 1, INTERVAL_DAYS.length - 1)
    : Math.max(currentLevel - 1, 0);

  const now = new Date();
  const nextReviewAt = new Date(now);
  nextReviewAt.setDate(now.getDate() + INTERVAL_DAYS[newLevel]);

  return {
    newLevel,
    nextReviewAt: nextReviewAt.toISOString(),
    lastReviewedAt: now.toISOString(),
  };
}
```

**যুক্তি:** সঠিক উত্তর দিলে level ১ বাড়ে (পরের review দূরে সরে যায়), ভুল দিলে level ১ কমে (আবার তাড়াতাড়ি দেখানো হবে)। Level 0 = নতুন/কঠিন শব্দ, level 5 = ভালোভাবে শেখা।

---

## Step 8.4 — SRS Repository + Sync

`src/db/repositories/SrsRepository.ts`

```ts
import { BaseRepository } from "./BaseRepository";
import { computeNextReview } from "../../srs/spacedRepetition";

export interface SrsRow {
  word_id: string;
  level: number;
  next_review_at: string;
  last_reviewed_at: string;
  is_synced: number;
}

class SrsRepositoryClass extends BaseRepository<SrsRow> {
  protected tableName = "word_srs";

  async findByWord(wordId: string): Promise<SrsRow | null> {
    const db = await this.db();
    const row = await db.getFirstAsync<SrsRow>(
      "SELECT * FROM word_srs WHERE word_id = ?",
      [wordId],
    );
    return row ?? null;
  }

  async recordAnswer(wordId: string, isCorrect: boolean): Promise<SrsRow> {
    const existing = await this.findByWord(wordId);
    const { newLevel, nextReviewAt, lastReviewedAt } = computeNextReview(
      existing?.level ?? 0,
      isCorrect,
    );

    const db = await this.db();
    if (existing) {
      await db.runAsync(
        `UPDATE word_srs SET level = ?, next_review_at = ?, last_reviewed_at = ?, is_synced = 0 WHERE word_id = ?`,
        [newLevel, nextReviewAt, lastReviewedAt, wordId],
      );
    } else {
      await db.runAsync(
        `INSERT INTO word_srs (word_id, level, next_review_at, last_reviewed_at, is_synced)
         VALUES (?, ?, ?, ?, 0)`,
        [wordId, newLevel, nextReviewAt, lastReviewedAt],
      );
    }

    return {
      word_id: wordId,
      level: newLevel,
      next_review_at: nextReviewAt,
      last_reviewed_at: lastReviewedAt,
      is_synced: 0,
    };
  }

  async markSynced(wordId: string): Promise<void> {
    const db = await this.db();
    await db.runAsync("UPDATE word_srs SET is_synced = 1 WHERE word_id = ?", [
      wordId,
    ]);
  }

  // Daily Practice-এর জন্য — যেসব শব্দ আজ review করার সময় হয়েছে
  async getDueWords(limit = 20) {
    const db = await this.db();
    return db.getAllAsync<{
      word_id: string;
      arabic: string;
      pronunciation: string | null;
      bangla_meaning: string;
      english_meaning: string;
      audio_url: string | null;
      level: number;
    }>(
      `SELECT w.id as word_id, w.arabic, w.pronunciation, w.bangla_meaning, w.english_meaning, w.audio_url, s.level
       FROM word_srs s
       JOIN words w ON w.id = s.word_id
       WHERE s.next_review_at <= ?
       ORDER BY s.next_review_at ASC
       LIMIT ?`,
      [new Date().toISOString(), limit],
    );
  }
}

export const SrsRepository = new SrsRepositoryClass();
```

`src/features/srs/registerSrsSync.ts`

```ts
import { registerSyncHandler } from "../../sync/syncEngine";
import { apiClient } from "../../api/apiClient";
import { SrsRepository } from "../../db/repositories/SrsRepository";

export function registerSrsSyncHandler(): void {
  registerSyncHandler("srs", async (item) => {
    const payload = JSON.parse(item.payload);
    await apiClient.post("/srs/sync", payload);
    await SrsRepository.markSynced(item.entity_id);
  });
}
```

`src/features/srs/srsActions.ts`

```ts
import { SrsRepository } from "../../db/repositories/SrsRepository";
import { SyncQueueRepository } from "../../db/repositories/SyncQueueRepository";

export async function answerWord(
  wordId: string,
  isCorrect: boolean,
): Promise<void> {
  const row = await SrsRepository.recordAnswer(wordId, isCorrect);
  await SyncQueueRepository.enqueue("srs", wordId, "update", {
    wordId: row.word_id,
    level: row.level,
    nextReviewAt: row.next_review_at,
    lastReviewedAt: row.last_reviewed_at,
  });
}
```

`app/_layout.tsx`-এ যোগ করো (Progress handler-এর পাশে):

```ts
import { registerSrsSyncHandler } from "../src/features/srs/registerSrsSync";
registerSrsSyncHandler();
```

---

## Step 8.5 — MCQ Generator (Local, Offline)

`src/quiz/mcqGenerator.ts`

```ts
import { WordRow } from "../db/repositories/WordRepository";

export interface McqQuestion {
  wordId: string;
  arabic: string;
  audioUrl: string | null;
  correctAnswer: string;
  options: string[]; // shuffled, correctAnswer সহ ৪টা
}

function shuffle<T>(arr: T[]): T[] {
  return [...arr].sort(() => Math.random() - 0.5);
}

export function generateMcqQuestions(words: WordRow[]): McqQuestion[] {
  if (words.length < 4) {
    // যথেষ্ট distractor না থাকলে যা আছে তা দিয়েই কাজ চালাও
    return words.map((w) => ({
      wordId: w.id,
      arabic: w.arabic,
      audioUrl: w.audio_url,
      correctAnswer: w.bangla_meaning,
      options: shuffle(words.map((x) => x.bangla_meaning)),
    }));
  }

  return words.map((word) => {
    const distractors = shuffle(words.filter((w) => w.id !== word.id))
      .slice(0, 3)
      .map((w) => w.bangla_meaning);

    return {
      wordId: word.id,
      arabic: word.arabic,
      audioUrl: word.audio_url,
      correctAnswer: word.bangla_meaning,
      options: shuffle([word.bangla_meaning, ...distractors]),
    };
  });
}
```

---

## Step 8.6 — MCQ Quiz Screen (Lesson শেষে ব্যবহৃত হবে)

`app/(app)/lessons/[lessonId]/quiz.tsx`

```tsx
import { View, Text, Pressable } from "react-native";
import { useState, useMemo, useEffect } from "react";
import { useLocalSearchParams, router } from "expo-router";
import { useWords } from "../../../../src/features/words/useWords";
import {
  generateMcqQuestions,
  McqQuestion,
} from "../../../../src/quiz/mcqGenerator";
import { answerWord } from "../../../../src/features/srs/srsActions";
import { markLessonCompleted } from "../../../../src/features/progress/progressActions";

export default function QuizScreen() {
  const { lessonId } = useLocalSearchParams<{ lessonId: string }>();
  const { data: words, isLoading } = useWords(lessonId);

  const [questions, setQuestions] = useState<McqQuestion[]>([]);
  const [current, setCurrent] = useState(0);
  const [selected, setSelected] = useState<string | null>(null);
  const [correctCount, setCorrectCount] = useState(0);
  const [finished, setFinished] = useState(false);

  useEffect(() => {
    if (words && words.length > 0) {
      setQuestions(generateMcqQuestions(words));
    }
  }, [words]);

  if (isLoading || questions.length === 0) {
    return (
      <View className="flex-1 items-center justify-center bg-white dark:bg-gray-900">
        <Text className="text-gray-400">প্রশ্ন তৈরি হচ্ছে...</Text>
      </View>
    );
  }

  const question = questions[current];

  const selectAnswer = async (option: string) => {
    if (selected) return; // একবার সিলেক্ট করলে আর বদলানো যাবে না
    setSelected(option);

    const isCorrect = option === question.correctAnswer;
    if (isCorrect) setCorrectCount((c) => c + 1);
    await answerWord(question.wordId, isCorrect);
  };

  const next = async () => {
    if (current + 1 < questions.length) {
      setCurrent((c) => c + 1);
      setSelected(null);
    } else {
      const scorePercent = Math.round((correctCount / questions.length) * 100);
      await markLessonCompleted(lessonId, scorePercent);
      setFinished(true);
    }
  };

  if (finished) {
    const scorePercent = Math.round((correctCount / questions.length) * 100);
    return (
      <View className="flex-1 items-center justify-center bg-white dark:bg-gray-900 px-6">
        <Text className="text-2xl font-semibold text-emerald-600 mb-2">
          কুইজ শেষ!
        </Text>
        <Text className="text-lg text-gray-700 dark:text-gray-300 mb-6">
          তোমার স্কোর: {correctCount}/{questions.length} ({scorePercent}%)
        </Text>
        <Pressable
          onPress={() => router.back()}
          className="bg-emerald-500 rounded-xl px-6 py-3"
        >
          <Text className="text-white font-semibold">লেসনে ফিরে যাও</Text>
        </Pressable>
      </View>
    );
  }

  return (
    <View className="flex-1 bg-white dark:bg-gray-900 px-6 py-8">
      <Text className="text-gray-400 mb-4">
        প্রশ্ন {current + 1} / {questions.length}
      </Text>
      <Text className="text-4xl text-center text-gray-900 dark:text-white mb-8">
        {question.arabic}
      </Text>

      {question.options.map((option) => {
        const isSelected = selected === option;
        const isCorrectOption = option === question.correctAnswer;
        let bgClass = "bg-gray-100 dark:bg-gray-800";

        if (selected) {
          if (isCorrectOption) bgClass = "bg-emerald-200 dark:bg-emerald-800";
          else if (isSelected) bgClass = "bg-red-200 dark:bg-red-900";
        }

        return (
          <Pressable
            key={option}
            onPress={() => selectAnswer(option)}
            className={`${bgClass} rounded-xl p-4 mb-3`}
          >
            <Text className="text-gray-900 dark:text-white text-center">
              {option}
            </Text>
          </Pressable>
        );
      })}

      {selected && (
        <Pressable
          onPress={next}
          className="bg-emerald-500 rounded-xl p-4 mt-4"
        >
          <Text className="text-white text-center font-semibold">
            {current + 1 < questions.length ? "পরবর্তী প্রশ্ন" : "ফলাফল দেখো"}
          </Text>
        </Pressable>
      )}
    </View>
  );
}
```

---

## Step 8.7 — Lesson Player Flow আপডেট (Fixed Score → Real Quiz Score)

Step 7.9-এ "done" step-এ ফিক্সড ১০০ score দেওয়া হতো। এখন সেটা বদলে quiz-এ পাঠাচ্ছি — real score সেখান থেকে আসবে।

`app/(app)/lessons/[lessonId]/learn.tsx`-এ পরিবর্তন করো:

```tsx
// আগে ছিল: markLessonCompleted(lessonId, 100) — এই লাইন সরিয়ে ফেলো
// "done" step-এর বদলে conversation শেষে সরাসরি quiz-এ redirect করো:

const goNext = async () => {
  if (stepIndex >= steps.length - 1) return;

  const isLastStep = steps[stepIndex + 1]?.type === "done";
  if (isLastStep) {
    router.replace(`/lessons/${lessonId}/quiz`); // markLessonCompleted এখন quiz.tsx-এ হবে
    return;
  }
  setStepIndex((i) => i + 1);
};
```

`steps` array থেকে `{ type: 'done' }` বাদ দিয়ে conversation/last-word-ই শেষ ধাপ, তারপর quiz স্বয়ংক্রিয়ভাবে খুলবে।

---

## Step 8.8 — Flashcard Screen (স্বাধীন Revision Mode)

Quiz থেকে আলাদা — এখানে শুধু SRS আপডেট হয়, lesson সম্পূর্ণ হওয়ার সাথে সম্পর্ক নেই। Free revision-এর জন্য।

`app/(app)/lessons/[lessonId]/flashcards.tsx`

```tsx
import { View, Text, Pressable } from "react-native";
import { useState } from "react";
import { useLocalSearchParams, router } from "expo-router";
import { useWords } from "../../../../src/features/words/useWords";
import { answerWord } from "../../../../src/features/srs/srsActions";

export default function FlashcardsScreen() {
  const { lessonId } = useLocalSearchParams<{ lessonId: string }>();
  const { data: words, isLoading } = useWords(lessonId);
  const [index, setIndex] = useState(0);
  const [flipped, setFlipped] = useState(false);

  if (isLoading || !words || words.length === 0) {
    return (
      <View className="flex-1 items-center justify-center bg-white dark:bg-gray-900">
        <Text className="text-gray-400">লোড হচ্ছে...</Text>
      </View>
    );
  }

  if (index >= words.length) {
    return (
      <View className="flex-1 items-center justify-center bg-white dark:bg-gray-900 px-6">
        <Text className="text-xl font-semibold text-emerald-600 mb-4">
          রিভিশন শেষ!
        </Text>
        <Pressable
          onPress={() => router.back()}
          className="bg-emerald-500 rounded-xl px-6 py-3"
        >
          <Text className="text-white font-semibold">ফিরে যাও</Text>
        </Pressable>
      </View>
    );
  }

  const word = words[index];

  const respond = async (knewIt: boolean) => {
    await answerWord(word.id, knewIt);
    setFlipped(false);
    setIndex((i) => i + 1);
  };

  return (
    <View className="flex-1 bg-white dark:bg-gray-900 items-center justify-center px-6">
      <Text className="text-gray-400 mb-4">
        {index + 1} / {words.length}
      </Text>

      <Pressable
        onPress={() => setFlipped((f) => !f)}
        className="bg-gray-100 dark:bg-gray-800 rounded-2xl w-full aspect-square items-center justify-center mb-8"
      >
        {!flipped ? (
          <Text className="text-5xl text-gray-900 dark:text-white">
            {word.arabic}
          </Text>
        ) : (
          <View className="items-center">
            <Text className="text-lg text-gray-500 dark:text-gray-400 mb-2">
              {word.pronunciation}
            </Text>
            <Text className="text-2xl text-gray-900 dark:text-white">
              {word.bangla_meaning}
            </Text>
          </View>
        )}
      </Pressable>

      {flipped && (
        <View className="flex-row gap-4 w-full">
          <Pressable
            onPress={() => respond(false)}
            className="flex-1 bg-red-100 dark:bg-red-900 rounded-xl py-4 items-center"
          >
            <Text className="text-red-600 dark:text-red-300 font-semibold">
              জানতাম না
            </Text>
          </Pressable>
          <Pressable
            onPress={() => respond(true)}
            className="flex-1 bg-emerald-100 dark:bg-emerald-900 rounded-xl py-4 items-center"
          >
            <Text className="text-emerald-600 dark:text-emerald-300 font-semibold">
              জানতাম
            </Text>
          </Pressable>
        </View>
      )}
    </View>
  );
}
```

Words screen-এ (Step 5.17) একটা "Flashcard দিয়ে রিভিশন করো" বাটন যোগ করো header-এ।

---

## Step 8.9 — Daily Practice Screen (Due Words, সব Lesson মিলিয়ে)

`app/(app)/practice/index.tsx`

```tsx
import { View, Text, Pressable } from "react-native";
import { useState, useEffect } from "react";
import { router } from "expo-router";
import { SrsRepository } from "../../../src/db/repositories/SrsRepository";
import { answerWord } from "../../../src/features/srs/srsActions";

export default function DailyPracticeScreen() {
  const [dueWords, setDueWords] = useState<
    Awaited<ReturnType<typeof SrsRepository.getDueWords>>
  >([]);
  const [index, setIndex] = useState(0);
  const [flipped, setFlipped] = useState(false);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    SrsRepository.getDueWords(20).then((words) => {
      setDueWords(words);
      setLoading(false);
    });
  }, []);

  if (loading) {
    return (
      <View className="flex-1 items-center justify-center bg-white dark:bg-gray-900">
        <Text className="text-gray-400">লোড হচ্ছে...</Text>
      </View>
    );
  }

  if (dueWords.length === 0) {
    return (
      <View className="flex-1 items-center justify-center bg-white dark:bg-gray-900 px-6">
        <Text className="text-lg text-gray-700 dark:text-gray-300 text-center mb-2">
          🎉 আজকের রিভিশনের জন্য কোনো শব্দ বাকি নেই!
        </Text>
        <Text className="text-gray-400 text-center">
          নতুন লেসন শিখে আসো, নতুন শব্দ যোগ হবে।
        </Text>
      </View>
    );
  }

  if (index >= dueWords.length) {
    return (
      <View className="flex-1 items-center justify-center bg-white dark:bg-gray-900 px-6">
        <Text className="text-xl font-semibold text-emerald-600 mb-4">
          আজকের প্র্যাকটিস শেষ!
        </Text>
        <Pressable
          onPress={() => router.back()}
          className="bg-emerald-500 rounded-xl px-6 py-3"
        >
          <Text className="text-white font-semibold">ফিরে যাও</Text>
        </Pressable>
      </View>
    );
  }

  const word = dueWords[index];

  const respond = async (knewIt: boolean) => {
    await answerWord(word.word_id, knewIt);
    setFlipped(false);
    setIndex((i) => i + 1);
  };

  return (
    <View className="flex-1 bg-white dark:bg-gray-900 items-center justify-center px-6">
      <Text className="text-gray-400 mb-4">
        {index + 1} / {dueWords.length}
      </Text>

      <Pressable
        onPress={() => setFlipped((f) => !f)}
        className="bg-gray-100 dark:bg-gray-800 rounded-2xl w-full aspect-square items-center justify-center mb-8"
      >
        {!flipped ? (
          <Text className="text-5xl text-gray-900 dark:text-white">
            {word.arabic}
          </Text>
        ) : (
          <View className="items-center">
            <Text className="text-lg text-gray-500 dark:text-gray-400 mb-2">
              {word.pronunciation}
            </Text>
            <Text className="text-2xl text-gray-900 dark:text-white">
              {word.bangla_meaning}
            </Text>
          </View>
        )}
      </Pressable>

      {flipped && (
        <View className="flex-row gap-4 w-full">
          <Pressable
            onPress={() => respond(false)}
            className="flex-1 bg-red-100 dark:bg-red-900 rounded-xl py-4 items-center"
          >
            <Text className="text-red-600 dark:text-red-300 font-semibold">
              জানতাম না
            </Text>
          </Pressable>
          <Pressable
            onPress={() => respond(true)}
            className="flex-1 bg-emerald-100 dark:bg-emerald-900 rounded-xl py-4 items-center"
          >
            <Text className="text-emerald-600 dark:text-emerald-300 font-semibold">
              জানতাম
            </Text>
          </Pressable>
        </View>
      )}
    </View>
  );
}
```

---

**টেস্ট করো:**

1. একটা lesson-এ পুরো Learn flow শেষ করো — এবার শেষে fixed 100 না, বরং MCQ quiz খুলবে, real score দিয়ে lesson complete হবে
2. Quiz-এ কিছু ভুল উত্তর দাও — MongoDB-তে `word_srs` collection-এ সেই word-এর level কমেছে কিনা (বা নতুন হলে 0-তেই আছে) চেক করো
3. Flashcard screen থেকে "জানতাম" চাপো কয়েকবার — SQLite-এ `word_srs.level` বাড়ছে আর `next_review_at` ভবিষ্যতের তারিখ হচ্ছে কিনা দেখো
4. `word_srs`-এ `next_review_at` ম্যানুয়ালি past date-এ বদলে দাও (SQLite inspector দিয়ে) — Daily Practice screen-এ সেই word "due" হিসেবে দেখানো উচিত

---

## এই Step-এ যা হলো (সারসংক্ষেপ)

- **Leitner-style spaced repetition** — সহজ, বোধগম্য, ৬টা level, প্রতি ভুল/ঠিকে interval কমে/বাড়ে
- MCQ ও Flashcard দুটোই সম্পূর্ণ **offline** কাজ করে — local SQLite word data থেকে generate হয়
- SRS state Progress-এর মতোই outbox pattern দিয়ে backend-এ sync হয়
- **Step 7-এর brutal truth সমাধান হলো** — ফিক্সড 100 score-এর জায়গায় এখন real quiz performance থেকে score আসে
- Daily Practice — সব lesson মিলিয়ে "due" শব্দ একসাথে রিভিশনের সুযোগ

**ব্রুটাল ট্রুথ:** MCQ distractor selection এখন lesson-level pool থেকে random — যদি lesson-এ শুধু ৪-৫টা শব্দ থাকে, distractor-গুলো predictable হয়ে যেতে পারে (ছোট pool)। বড় lesson-এ সমস্যা না, কিন্তু ছোট lesson-এ quiz একটু সহজ মনে হতে পারে। এখন এটা acceptable trade-off, ভবিষ্যতে চাইলে category-level pool থেকে distractor নেওয়া যায়।

**পরবর্তী যৌক্তিক ধাপ:** Step 9 — User Progress (Bookmarks, Favorites, Daily Goal, Streak, Statistics, Sync)। Progress আর SRS collection ইতিমধ্যে আছে, `getRecentlyPracticed()` ও প্রস্তুত — এখন Home/Stats screen আর streak calculation logic বানানো হবে।
