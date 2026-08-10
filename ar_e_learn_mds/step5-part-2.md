# Step 5 (বাকি অংশ): Lessons → Words → Sentences Screens + Search

Categories vertical slice-এ যে প্যাটার্ন বানানো হয়েছিল (SQLite = read source, API = background sync), এখন সেটাই Lessons, Words, Sentences-এ repeat করছি।

---

## Step 5.12 — Schema Fix (Sentences-এ transliteration কলাম যোগ)

Step 4.3-এর migration-এ `sentences` টেবিলে `transliteration` কলাম রাখা হয়নি, কিন্তু backend model-এ আছে। নতুন migration দিয়ে ঠিক করছি (আগের migration কখনো edit করবে না — নতুন migration যোগ করাই সঠিক নিয়ম)।

`src/db/migrations/002_add_sentence_transliteration.ts`

```ts
import { SQLiteDatabase } from "expo-sqlite";
import { Migration } from "./index";

export const migration_002_add_sentence_transliteration: Migration = {
  version: 2,
  name: "add_sentence_transliteration",
  up: async (db: SQLiteDatabase) => {
    await db.execAsync(`
      ALTER TABLE sentences ADD COLUMN transliteration TEXT;
    `);
  },
};
```

`src/db/migrations/index.ts`-এ যোগ করো:

```ts
import { migration_002_add_sentence_transliteration } from "./002_add_sentence_transliteration";

export const migrations: Migration[] = [
  migration_001_init,
  migration_002_add_sentence_transliteration, // যোগ করো
];
```

**টেস্ট করো:** App restart করো — crash ছাড়া migration রান হওয়া উচিত (কারণ `user_version` আগে 1 ছিল, এখন শুধু version 2-টাই নতুন করে চলবে)।

---

## Step 5.13 — Mobile API Layer সম্পূর্ণ করা

`src/api/contentApi.ts`-এ যোগ করো:

```ts
export interface WordDto {
  _id: string;
  lessonId: string;
  arabic: string;
  transliteration: string;
  banglaMeaning: string;
  englishMeaning: string;
  audioUrl?: string;
  sortOrder: number;
}

export interface SentenceDto {
  _id: string;
  wordId: string;
  arabic: string;
  transliteration: string;
  banglaMeaning: string;
  englishMeaning: string;
  audioUrl?: string;
}

// contentApi object-এর ভেতরে যোগ করো:
getWordsByLesson: async (lessonId: string): Promise<WordDto[]> => {
  const { data } = await apiClient.get(`/content/lessons/${lessonId}/words`);
  return data.items;
},

getSentencesByWord: async (wordId: string): Promise<SentenceDto[]> => {
  const { data } = await apiClient.get(`/content/words/${wordId}/sentences`);
  return data.data;
},
```

---

## Step 5.14 — Mobile SQLite Repositories (Word, Sentence)

`src/db/repositories/WordRepository.ts`

```ts
import { BaseRepository } from "./BaseRepository";

export interface WordRow {
  id: string;
  lesson_id: string;
  arabic: string;
  pronunciation: string | null; // backend-এর transliteration এখানে map হয়
  bangla_meaning: string;
  english_meaning: string;
  audio_url: string | null;
  updated_at: string;
}

class WordRepositoryClass extends BaseRepository<WordRow> {
  protected tableName = "words";

  async findByLesson(lessonId: string): Promise<WordRow[]> {
    const db = await this.db();
    return db.getAllAsync<WordRow>(
      "SELECT * FROM words WHERE lesson_id = ? ORDER BY rowid ASC",
      [lessonId],
    );
  }

  // Offline-first local search — full dataset আগে থেকেই SQLite-এ sync হয়ে থাকে,
  // তাই internet ছাড়াই কাজ করে
  async searchLocal(query: string): Promise<WordRow[]> {
    const db = await this.db();
    const like = `%${query}%`;
    return db.getAllAsync<WordRow>(
      `SELECT * FROM words
       WHERE arabic LIKE ? OR pronunciation LIKE ? OR bangla_meaning LIKE ? OR english_meaning LIKE ?
       LIMIT 50`,
      [like, like, like, like],
    );
  }

  async upsertMany(words: WordRow[]): Promise<void> {
    const db = await this.db();
    await db.withTransactionAsync(async () => {
      for (const w of words) {
        await db.runAsync(
          `INSERT INTO words (id, lesson_id, arabic, pronunciation, bangla_meaning, english_meaning, audio_url, updated_at)
           VALUES (?, ?, ?, ?, ?, ?, ?, ?)
           ON CONFLICT(id) DO UPDATE SET
             arabic=excluded.arabic, pronunciation=excluded.pronunciation,
             bangla_meaning=excluded.bangla_meaning, english_meaning=excluded.english_meaning,
             audio_url=excluded.audio_url, updated_at=excluded.updated_at`,
          [
            w.id,
            w.lesson_id,
            w.arabic,
            w.pronunciation,
            w.bangla_meaning,
            w.english_meaning,
            w.audio_url,
            w.updated_at,
          ],
        );
      }
    });
  }
}

export const WordRepository = new WordRepositoryClass();
```

`src/db/repositories/SentenceRepository.ts`

```ts
import { BaseRepository } from "./BaseRepository";

export interface SentenceRow {
  id: string;
  word_id: string;
  arabic: string;
  transliteration: string | null;
  bangla_meaning: string;
  english_meaning: string;
  audio_url: string | null;
  updated_at: string;
}

class SentenceRepositoryClass extends BaseRepository<SentenceRow> {
  protected tableName = "sentences";

  async findByWord(wordId: string): Promise<SentenceRow[]> {
    const db = await this.db();
    return db.getAllAsync<SentenceRow>(
      "SELECT * FROM sentences WHERE word_id = ?",
      [wordId],
    );
  }

  async upsertMany(sentences: SentenceRow[]): Promise<void> {
    const db = await this.db();
    await db.withTransactionAsync(async () => {
      for (const s of sentences) {
        await db.runAsync(
          `INSERT INTO sentences (id, word_id, arabic, transliteration, bangla_meaning, english_meaning, audio_url, updated_at)
           VALUES (?, ?, ?, ?, ?, ?, ?, ?)
           ON CONFLICT(id) DO UPDATE SET
             arabic=excluded.arabic, transliteration=excluded.transliteration,
             bangla_meaning=excluded.bangla_meaning, english_meaning=excluded.english_meaning,
             audio_url=excluded.audio_url, updated_at=excluded.updated_at`,
          [
            s.id,
            s.word_id,
            s.arabic,
            s.transliteration,
            s.bangla_meaning,
            s.english_meaning,
            s.audio_url,
            s.updated_at,
          ],
        );
      }
    });
  }
}

export const SentenceRepository = new SentenceRepositoryClass();
```

---

## Step 5.15 — Sync Functions + Hooks

`src/features/lessons/lessonSync.ts`

```ts
import { contentApi } from "../../api/contentApi";
import { LessonRepository } from "../../db/repositories/LessonRepository";

export async function syncLessons(categoryId: string): Promise<void> {
  const remote = await contentApi.getLessonsByCategory(categoryId);
  const rows = remote.map((l) => ({
    id: l._id,
    category_id: l.categoryId,
    title_ar: l.titleAr,
    title_bn: l.titleBn,
    title_en: l.titleEn,
    sort_order: l.sortOrder,
    updated_at: new Date().toISOString(),
  }));
  await LessonRepository.upsertMany(rows);
}
```

`src/features/lessons/useLessons.ts`

```ts
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { useEffect } from "react";
import { LessonRepository } from "../../db/repositories/LessonRepository";
import { syncLessons } from "./lessonSync";
import { useNetworkStore } from "../../store/networkStore";

export function useLessons(categoryId: string) {
  const queryClient = useQueryClient();
  const isOnline = useNetworkStore((s) => s.isOnline);

  const query = useQuery({
    queryKey: ["lessons", "local", categoryId],
    queryFn: () => LessonRepository.findByCategory(categoryId),
    enabled: !!categoryId,
  });

  useEffect(() => {
    if (!isOnline || !categoryId) return;
    syncLessons(categoryId)
      .then(() =>
        queryClient.invalidateQueries({
          queryKey: ["lessons", "local", categoryId],
        }),
      )
      .catch((err) => console.warn("Lesson sync failed:", err));
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [isOnline, categoryId]);

  return query;
}
```

`src/features/words/wordSync.ts`

```ts
import { contentApi } from "../../api/contentApi";
import { WordRepository } from "../../db/repositories/WordRepository";

export async function syncWords(lessonId: string): Promise<void> {
  const remote = await contentApi.getWordsByLesson(lessonId);
  const rows = remote.map((w) => ({
    id: w._id,
    lesson_id: w.lessonId,
    arabic: w.arabic,
    pronunciation: w.transliteration,
    bangla_meaning: w.banglaMeaning,
    english_meaning: w.englishMeaning,
    audio_url: w.audioUrl ?? null,
    updated_at: new Date().toISOString(),
  }));
  await WordRepository.upsertMany(rows);
}
```

`src/features/words/useWords.ts`

```ts
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { useEffect } from "react";
import { WordRepository } from "../../db/repositories/WordRepository";
import { syncWords } from "./wordSync";
import { useNetworkStore } from "../../store/networkStore";

export function useWords(lessonId: string) {
  const queryClient = useQueryClient();
  const isOnline = useNetworkStore((s) => s.isOnline);

  const query = useQuery({
    queryKey: ["words", "local", lessonId],
    queryFn: () => WordRepository.findByLesson(lessonId),
    enabled: !!lessonId,
  });

  useEffect(() => {
    if (!isOnline || !lessonId) return;
    syncWords(lessonId)
      .then(() =>
        queryClient.invalidateQueries({
          queryKey: ["words", "local", lessonId],
        }),
      )
      .catch((err) => console.warn("Word sync failed:", err));
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [isOnline, lessonId]);

  return query;
}
```

`src/features/sentences/sentenceSync.ts`

```ts
import { contentApi } from "../../api/contentApi";
import { SentenceRepository } from "../../db/repositories/SentenceRepository";

export async function syncSentences(wordId: string): Promise<void> {
  const remote = await contentApi.getSentencesByWord(wordId);
  const rows = remote.map((s) => ({
    id: s._id,
    word_id: s.wordId,
    arabic: s.arabic,
    transliteration: s.transliteration,
    bangla_meaning: s.banglaMeaning,
    english_meaning: s.englishMeaning,
    audio_url: s.audioUrl ?? null,
    updated_at: new Date().toISOString(),
  }));
  await SentenceRepository.upsertMany(rows);
}
```

`src/features/sentences/useSentences.ts`

```ts
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { useEffect } from "react";
import { SentenceRepository } from "../../db/repositories/SentenceRepository";
import { syncSentences } from "./sentenceSync";
import { useNetworkStore } from "../../store/networkStore";

export function useSentences(wordId: string) {
  const queryClient = useQueryClient();
  const isOnline = useNetworkStore((s) => s.isOnline);

  const query = useQuery({
    queryKey: ["sentences", "local", wordId],
    queryFn: () => SentenceRepository.findByWord(wordId),
    enabled: !!wordId,
  });

  useEffect(() => {
    if (!isOnline || !wordId) return;
    syncSentences(wordId)
      .then(() =>
        queryClient.invalidateQueries({
          queryKey: ["sentences", "local", wordId],
        }),
      )
      .catch((err) => console.warn("Sentence sync failed:", err));
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [isOnline, wordId]);

  return query;
}
```

---

## Step 5.16 — Lessons Screen

`app/(app)/categories/[categoryId]/lessons.tsx`

```tsx
import { View, Text, FlatList, Pressable } from "react-native";
import { useLocalSearchParams, router } from "expo-router";
import { useLessons } from "../../../../src/features/lessons/useLessons";

export default function LessonsScreen() {
  const { categoryId } = useLocalSearchParams<{ categoryId: string }>();
  const { data: lessons, isLoading, isError } = useLessons(categoryId);

  if (isLoading) {
    return (
      <View className="flex-1 items-center justify-center bg-white dark:bg-gray-900">
        <Text className="text-gray-400">লোড হচ্ছে...</Text>
      </View>
    );
  }

  if (isError) {
    return (
      <View className="flex-1 items-center justify-center bg-white dark:bg-gray-900 px-6">
        <Text className="text-red-500 text-center">লেসন লোড করা যায়নি।</Text>
      </View>
    );
  }

  if (!lessons || lessons.length === 0) {
    return (
      <View className="flex-1 items-center justify-center bg-white dark:bg-gray-900 px-6">
        <Text className="text-gray-400 text-center">
          এই ক্যাটাগরিতে এখনো কোনো লেসন নেই।
        </Text>
      </View>
    );
  }

  return (
    <FlatList
      className="flex-1 bg-white dark:bg-gray-900"
      data={lessons}
      keyExtractor={(item) => item.id}
      contentContainerStyle={{ padding: 16 }}
      renderItem={({ item }) => (
        <Pressable
          onPress={() => router.push(`/lessons/${item.id}/words`)}
          className="bg-gray-100 dark:bg-gray-800 rounded-xl p-4 mb-3"
        >
          <Text className="text-lg font-semibold text-gray-900 dark:text-white">
            {item.title_bn}
          </Text>
          <Text className="text-gray-500 dark:text-gray-400">
            {item.title_en}
          </Text>
        </Pressable>
      )}
    />
  );
}
```

---

## Step 5.17 — Words Screen

`app/(app)/lessons/[lessonId]/words.tsx`

```tsx
import { View, Text, FlatList, Pressable } from "react-native";
import { useLocalSearchParams, router } from "expo-router";
import { useWords } from "../../../../src/features/words/useWords";

export default function WordsScreen() {
  const { lessonId } = useLocalSearchParams<{ lessonId: string }>();
  const { data: words, isLoading, isError } = useWords(lessonId);

  if (isLoading) {
    return (
      <View className="flex-1 items-center justify-center bg-white dark:bg-gray-900">
        <Text className="text-gray-400">লোড হচ্ছে...</Text>
      </View>
    );
  }

  if (isError) {
    return (
      <View className="flex-1 items-center justify-center bg-white dark:bg-gray-900 px-6">
        <Text className="text-red-500 text-center">শব্দ লোড করা যায়নি।</Text>
      </View>
    );
  }

  if (!words || words.length === 0) {
    return (
      <View className="flex-1 items-center justify-center bg-white dark:bg-gray-900 px-6">
        <Text className="text-gray-400 text-center">
          এই লেসনে এখনো কোনো শব্দ নেই।
        </Text>
      </View>
    );
  }

  return (
    <FlatList
      className="flex-1 bg-white dark:bg-gray-900"
      data={words}
      keyExtractor={(item) => item.id}
      contentContainerStyle={{ padding: 16 }}
      renderItem={({ item }) => (
        <Pressable
          onPress={() => router.push(`/words/${item.id}`)}
          className="bg-gray-100 dark:bg-gray-800 rounded-xl p-4 mb-3 flex-row justify-between items-center"
        >
          <View>
            <Text className="text-lg font-semibold text-gray-900 dark:text-white">
              {item.bangla_meaning}
            </Text>
            <Text className="text-gray-500 dark:text-gray-400">
              {item.pronunciation}
            </Text>
          </View>
          <Text className="text-2xl">{item.arabic}</Text>
        </Pressable>
      )}
    />
  );
}
```

---

## Step 5.18 — Word Detail Screen (Sentences সহ)

`app/(app)/words/[wordId]/index.tsx`

```tsx
import { View, Text, ScrollView } from "react-native";
import { useLocalSearchParams } from "expo-router";
import { useSentences } from "../../../src/features/sentences/useSentences";
import {
  WordRepository,
  WordRow,
} from "../../../src/db/repositories/WordRepository";
import { useEffect, useState } from "react";

export default function WordDetailScreen() {
  const { wordId } = useLocalSearchParams<{ wordId: string }>();
  const [word, setWord] = useState<WordRow | null>(null);
  const { data: sentences, isLoading: sentencesLoading } = useSentences(wordId);

  useEffect(() => {
    WordRepository.findById(wordId).then(setWord);
  }, [wordId]);

  if (!word) {
    return (
      <View className="flex-1 items-center justify-center bg-white dark:bg-gray-900">
        <Text className="text-gray-400">লোড হচ্ছে...</Text>
      </View>
    );
  }

  return (
    <ScrollView className="flex-1 bg-white dark:bg-gray-900 px-6 py-8">
      <Text className="text-4xl text-right mb-2 text-gray-900 dark:text-white">
        {word.arabic}
      </Text>
      <Text className="text-lg text-gray-500 dark:text-gray-400 mb-4">
        {word.pronunciation}
      </Text>
      <Text className="text-xl text-gray-900 dark:text-white">
        {word.bangla_meaning}
      </Text>
      <Text className="text-base text-gray-500 dark:text-gray-400 mb-6">
        {word.english_meaning}
      </Text>

      <Text className="text-sm font-semibold text-gray-400 uppercase mb-2">
        উদাহরণ বাক্য
      </Text>

      {sentencesLoading && <Text className="text-gray-400">লোড হচ্ছে...</Text>}

      {!sentencesLoading && (!sentences || sentences.length === 0) && (
        <Text className="text-gray-400">এই শব্দের কোনো উদাহরণ বাক্য নেই।</Text>
      )}

      {sentences?.map((s) => (
        <View
          key={s.id}
          className="bg-gray-100 dark:bg-gray-800 rounded-xl p-4 mb-3"
        >
          <Text className="text-xl text-right text-gray-900 dark:text-white mb-1">
            {s.arabic}
          </Text>
          <Text className="text-gray-500 dark:text-gray-400 mb-1">
            {s.transliteration}
          </Text>
          <Text className="text-gray-900 dark:text-white">
            {s.bangla_meaning}
          </Text>
          <Text className="text-gray-500 dark:text-gray-400">
            {s.english_meaning}
          </Text>
        </View>
      ))}
    </ScrollView>
  );
}
```

---

## Step 5.19 — Search (Offline-First, Local SQLite)

**Decision:** ব্যাকএন্ডে text-index search endpoint থাকলেও, এই app-এর ডাটা ছোট আকারের এবং ইতিমধ্যে SQLite-এ sync হয়ে থাকে — তাই Search সম্পূর্ণ local SQLite-এ (`LIKE` query) implement করছি। এতে internet ছাড়াও search কাজ করবে, যেটা "Offline First" requirement-এর সাথে সরাসরি সামঞ্জস্যপূর্ণ। Backend-এর `$text` search endpoint অক্ষত থাকছে ভবিষ্যতে admin/analytics বা বড় ডাটাসেটের জন্য দরকার হলে ব্যবহার করার জন্য।

`src/features/search/useWordSearch.ts`

```ts
import { useState, useEffect } from "react";
import { WordRepository, WordRow } from "../../db/repositories/WordRepository";

export function useWordSearch(query: string) {
  const [results, setResults] = useState<WordRow[]>([]);
  const [isSearching, setIsSearching] = useState(false);

  useEffect(() => {
    if (query.trim().length < 2) {
      setResults([]);
      return;
    }

    setIsSearching(true);
    const timeout = setTimeout(() => {
      WordRepository.searchLocal(query.trim())
        .then(setResults)
        .finally(() => setIsSearching(false));
    }, 300); // debounce

    return () => clearTimeout(timeout);
  }, [query]);

  return { results, isSearching };
}
```

`app/(app)/search/index.tsx`

```tsx
import { View, Text, TextInput, FlatList } from "react-native";
import { useState } from "react";
import { router } from "expo-router";
import { Pressable } from "react-native";
import { useWordSearch } from "../../../src/features/search/useWordSearch";

export default function SearchScreen() {
  const [query, setQuery] = useState("");
  const { results, isSearching } = useWordSearch(query);

  return (
    <View className="flex-1 bg-white dark:bg-gray-900 px-4 pt-4">
      <TextInput
        value={query}
        onChangeText={setQuery}
        placeholder="আরবি, বাংলা বা ইংরেজিতে খুঁজুন..."
        placeholderTextColor="#9ca3af"
        className="bg-gray-100 dark:bg-gray-800 rounded-xl px-4 py-3 text-gray-900 dark:text-white mb-4"
      />

      {isSearching && (
        <Text className="text-gray-400 text-center">খোঁজা হচ্ছে...</Text>
      )}

      {!isSearching && query.trim().length >= 2 && results.length === 0 && (
        <Text className="text-gray-400 text-center">
          কোনো ফলাফল পাওয়া যায়নি।
        </Text>
      )}

      <FlatList
        data={results}
        keyExtractor={(item) => item.id}
        renderItem={({ item }) => (
          <Pressable
            onPress={() => router.push(`/words/${item.id}`)}
            className="bg-gray-100 dark:bg-gray-800 rounded-xl p-4 mb-3 flex-row justify-between items-center"
          >
            <View>
              <Text className="text-lg font-semibold text-gray-900 dark:text-white">
                {item.bangla_meaning}
              </Text>
              <Text className="text-gray-500 dark:text-gray-400">
                {item.pronunciation}
              </Text>
            </View>
            <Text className="text-2xl">{item.arabic}</Text>
          </Pressable>
        )}
      />
    </View>
  );
}
```

**টেস্ট করো:**

1. Categories → Lessons → Words → Word Detail (sentences সহ) পুরো flow ট্যাপ করে ঘুরে দেখো
2. Airplane mode অন করে পুরো flow আবার test করো — সব ডাটা (আগে sync হওয়া) দেখা উচিত
3. Search স্ক্রিনে "hello", "মা", "واحد" ইত্যাদি লিখে খুঁজে দেখো — অফলাইনেও কাজ করা উচিত

---

## Step 5 — সম্পূর্ণ (Final Summary)

- Backend: Category/Lesson/Word/Sentence schema + layered API + text-search endpoint + real seed data ✅
- Mobile: প্রতিটা entity-র জন্য SQLite repository + offline-first sync hook (SQLite = read, API = background sync) ✅
- Screens: Categories → Lessons → Words → Word Detail (with Sentences), সব loading/empty/error state সহ ✅
- Search: সম্পূর্ণ local SQLite-based, internet ছাড়াই কাজ করে ✅
- Schema inconsistency (sentences transliteration column) ধরা পড়ে migration দিয়ে ঠিক করা হয়েছে ✅

**ব্রুটাল ট্রুথ:** `WordRow.audio_url` column আছে কিন্তু কোনো actual audio download/playback logic এখনো নেই — সেটা ইচ্ছাকৃতভাবে বাদ, কারণ Step 6 পুরোটাই Audio System নিয়ে। এছাড়া User Profile MongoDB schema প্রশ্নটা এখনো ঝুলে আছে — Step 6 বা 7 শুরুর আগে সেটা সামলানো উচিত, বিশেষত Step 9 (Progress)-এর আগে অবশ্যই লাগবে।

**পরবর্তী যৌক্তিক ধাপ:** Step 6 — Audio System (Streaming, Download, Offline Cache, Playback), যেহেতু Word ও Sentence দুটোতেই `audio_url` ফিল্ড ইতিমধ্যে প্রস্তুত।
