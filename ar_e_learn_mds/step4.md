# Step 4: Local Database (SQLite Offline-First)

expo-sqlite (async API, SDK 56 কম্প্যাটিবল, বর্তমান latest: `56.0.5`) + Repository pattern + Outbox sync queue skeleton।

---

## Step 4.1 — Dependencies install

```bash
npx expo install expo-sqlite @react-native-community/netinfo
```

- `expo-sqlite` → local database (native, async API)
- `@react-native-community/netinfo` → online/offline detection (Expo-official recommended community lib)

---

## Step 4.2 — Database Connection + Migration Runner

Versioned migration system — `PRAGMA user_version` দিয়ে ট্র্যাক করা হয়, যাতে schema change করলে পুরনো ইউজারদের ডাটা নষ্ট না হয়।

`src/db/connection.ts`

```ts
import * as SQLite from "expo-sqlite";

const DATABASE_NAME = "arabic_learning.db";

let dbInstance: SQLite.SQLiteDatabase | null = null;

export async function getDatabase(): Promise<SQLite.SQLiteDatabase> {
  if (dbInstance) return dbInstance;

  dbInstance = await SQLite.openDatabaseAsync(DATABASE_NAME);

  // Performance + integrity pragmas
  await dbInstance.execAsync("PRAGMA journal_mode = WAL;");
  await dbInstance.execAsync("PRAGMA foreign_keys = ON;");

  return dbInstance;
}
```

`src/db/migrations/index.ts`

```ts
import { SQLiteDatabase } from "expo-sqlite";
import { migration_001_init } from "./001_init";

export interface Migration {
  version: number;
  name: string;
  up: (db: SQLiteDatabase) => Promise<void>;
}

export const migrations: Migration[] = [migration_001_init];

export async function runMigrations(db: SQLiteDatabase): Promise<void> {
  const result = await db.getFirstAsync<{ user_version: number }>(
    "PRAGMA user_version",
  );
  const currentVersion = result?.user_version ?? 0;

  const pending = migrations.filter((m) => m.version > currentVersion);

  for (const migration of pending) {
    await db.withTransactionAsync(async () => {
      await migration.up(db);
      await db.execAsync(`PRAGMA user_version = ${migration.version}`);
    });
  }
}
```

---

## Step 4.3 — Schema (First Migration)

`src/db/migrations/001_init.ts`

```ts
import { SQLiteDatabase } from "expo-sqlite";
import { Migration } from "./index";

export const migration_001_init: Migration = {
  version: 1,
  name: "init_schema",
  up: async (db: SQLiteDatabase) => {
    await db.execAsync(`
      -- Categories (mirrors backend Categories collection)
      CREATE TABLE IF NOT EXISTS categories (
        id TEXT PRIMARY KEY,
        name_ar TEXT NOT NULL,
        name_bn TEXT NOT NULL,
        name_en TEXT NOT NULL,
        icon TEXT,
        sort_order INTEGER DEFAULT 0,
        updated_at TEXT NOT NULL
      );

      -- Lessons
      CREATE TABLE IF NOT EXISTS lessons (
        id TEXT PRIMARY KEY,
        category_id TEXT NOT NULL,
        title_ar TEXT NOT NULL,
        title_bn TEXT NOT NULL,
        title_en TEXT NOT NULL,
        sort_order INTEGER DEFAULT 0,
        updated_at TEXT NOT NULL,
        FOREIGN KEY (category_id) REFERENCES categories(id)
      );

      -- Words
      CREATE TABLE IF NOT EXISTS words (
        id TEXT PRIMARY KEY,
        lesson_id TEXT NOT NULL,
        arabic TEXT NOT NULL,
        bangla_meaning TEXT NOT NULL,
        english_meaning TEXT NOT NULL,
        pronunciation TEXT,
        audio_url TEXT,
        updated_at TEXT NOT NULL,
        FOREIGN KEY (lesson_id) REFERENCES lessons(id)
      );

      -- Sentences
      CREATE TABLE IF NOT EXISTS sentences (
        id TEXT PRIMARY KEY,
        word_id TEXT NOT NULL,
        arabic TEXT NOT NULL,
        bangla_meaning TEXT NOT NULL,
        english_meaning TEXT NOT NULL,
        audio_url TEXT,
        updated_at TEXT NOT NULL,
        FOREIGN KEY (word_id) REFERENCES words(id)
      );

      -- User progress (local-first, synced to backend)
      CREATE TABLE IF NOT EXISTS progress (
        id TEXT PRIMARY KEY,
        lesson_id TEXT NOT NULL,
        status TEXT NOT NULL CHECK (status IN ('not_started', 'in_progress', 'completed')),
        score INTEGER DEFAULT 0,
        last_practiced_at TEXT,
        is_synced INTEGER DEFAULT 0,
        FOREIGN KEY (lesson_id) REFERENCES lessons(id)
      );

      -- Bookmarks / Favorites
      CREATE TABLE IF NOT EXISTS bookmarks (
        id TEXT PRIMARY KEY,
        item_type TEXT NOT NULL CHECK (item_type IN ('word', 'sentence', 'lesson')),
        item_id TEXT NOT NULL,
        created_at TEXT NOT NULL,
        is_synced INTEGER DEFAULT 0
      );

      -- Downloaded content tracking (Step 6 audio system-এ ব্যবহার হবে)
      CREATE TABLE IF NOT EXISTS downloads (
        id TEXT PRIMARY KEY,
        item_type TEXT NOT NULL CHECK (item_type IN ('lesson_audio', 'word_audio')),
        item_id TEXT NOT NULL,
        local_uri TEXT,
        status TEXT NOT NULL CHECK (status IN ('pending', 'downloading', 'completed', 'failed')) DEFAULT 'pending',
        downloaded_at TEXT
      );

      -- Outbox: offline actions queue (sync engine এর মূল অংশ)
      CREATE TABLE IF NOT EXISTS sync_queue (
        id TEXT PRIMARY KEY,
        entity_type TEXT NOT NULL,
        entity_id TEXT NOT NULL,
        operation TEXT NOT NULL CHECK (operation IN ('create', 'update', 'delete')),
        payload TEXT NOT NULL,
        status TEXT NOT NULL CHECK (status IN ('pending', 'syncing', 'failed')) DEFAULT 'pending',
        retry_count INTEGER DEFAULT 0,
        created_at TEXT NOT NULL,
        last_attempt_at TEXT
      );

      -- Sync metadata (প্রতিটা টেবিল সর্বশেষ কবে backend থেকে sync হয়েছে)
      CREATE TABLE IF NOT EXISTS sync_meta (
        table_name TEXT PRIMARY KEY,
        last_synced_at TEXT
      );

      CREATE INDEX IF NOT EXISTS idx_lessons_category ON lessons(category_id);
      CREATE INDEX IF NOT EXISTS idx_words_lesson ON words(lesson_id);
      CREATE INDEX IF NOT EXISTS idx_sentences_word ON sentences(word_id);
      CREATE INDEX IF NOT EXISTS idx_sync_queue_status ON sync_queue(status);
    `);
  },
};
```

**টেস্ট করো:** App চালু করে migration রান করো (পরের ধাপে wiring করবো), তারপর Drizzle Studio বা `db-inspector` দিয়ে টেবিলগুলো তৈরি হয়েছে কিনা চেক করো। অথবা সহজে: shift+m চেপে Expo dev menu থেকে "Open expo-sqlite" ইন্সপেক্টর ওপেন করো।

---

## Step 4.4 — Base Repository Pattern

Backend-এর মতোই mobile-এও Repository layer — যাতে business logic কখনো raw SQL নিয়ে কাজ না করে।

`src/db/repositories/BaseRepository.ts`

```ts
import { SQLiteDatabase } from "expo-sqlite";
import { getDatabase } from "../connection";

export abstract class BaseRepository<T> {
  protected abstract tableName: string;

  protected async db(): Promise<SQLiteDatabase> {
    return getDatabase();
  }

  async findAll(): Promise<T[]> {
    const db = await this.db();
    return db.getAllAsync<T>(`SELECT * FROM ${this.tableName}`);
  }

  async findById(id: string): Promise<T | null> {
    const db = await this.db();
    const row = await db.getFirstAsync<T>(
      `SELECT * FROM ${this.tableName} WHERE id = ?`,
      [id],
    );
    return row ?? null;
  }

  async deleteById(id: string): Promise<void> {
    const db = await this.db();
    await db.runAsync(`DELETE FROM ${this.tableName} WHERE id = ?`, [id]);
  }
}
```

**উদাহরণ — LessonRepository:**

`src/db/repositories/LessonRepository.ts`

```ts
import { BaseRepository } from "./BaseRepository";

export interface LessonRow {
  id: string;
  category_id: string;
  title_ar: string;
  title_bn: string;
  title_en: string;
  sort_order: number;
  updated_at: string;
}

class LessonRepositoryClass extends BaseRepository<LessonRow> {
  protected tableName = "lessons";

  async findByCategory(categoryId: string): Promise<LessonRow[]> {
    const db = await this.db();
    return db.getAllAsync<LessonRow>(
      "SELECT * FROM lessons WHERE category_id = ? ORDER BY sort_order ASC",
      [categoryId],
    );
  }

  async upsertMany(lessons: LessonRow[]): Promise<void> {
    const db = await this.db();
    await db.withTransactionAsync(async () => {
      for (const lesson of lessons) {
        await db.runAsync(
          `INSERT INTO lessons (id, category_id, title_ar, title_bn, title_en, sort_order, updated_at)
           VALUES (?, ?, ?, ?, ?, ?, ?)
           ON CONFLICT(id) DO UPDATE SET
             title_ar=excluded.title_ar,
             title_bn=excluded.title_bn,
             title_en=excluded.title_en,
             sort_order=excluded.sort_order,
             updated_at=excluded.updated_at`,
          [
            lesson.id,
            lesson.category_id,
            lesson.title_ar,
            lesson.title_bn,
            lesson.title_en,
            lesson.sort_order,
            lesson.updated_at,
          ],
        );
      }
    });
  }
}

export const LessonRepository = new LessonRepositoryClass();
```

এই একই প্যাটার্নে Step 5-এ `CategoryRepository`, `WordRepository`, `SentenceRepository` বানানো হবে — এখন শুধু ভিত্তিটা তৈরি করছি।

---

## Step 4.5 — Network Status (Zustand store)

`src/store/networkStore.ts`

```ts
import { create } from "zustand";
import NetInfo from "@react-native-community/netinfo";

interface NetworkState {
  isOnline: boolean;
  setOnline: (status: boolean) => void;
  init: () => () => void; // returns unsubscribe function
}

export const useNetworkStore = create<NetworkState>((set) => ({
  isOnline: true,
  setOnline: (status) => set({ isOnline: status }),
  init: () => {
    const unsubscribe = NetInfo.addEventListener((state) => {
      set({
        isOnline: Boolean(
          state.isConnected && state.isInternetReachable !== false,
        ),
      });
    });
    return unsubscribe;
  },
}));
```

App root-এ (Step 4.7-এ wiring করবো) এই `init()` একবার কল করলেই পুরো app-এ `useNetworkStore((s) => s.isOnline)` দিয়ে যেকোনো কম্পোনেন্ট থেকে online/offline status পাওয়া যাবে।

---

## Step 4.6 — Sync Queue Repository (Outbox)

`src/db/repositories/SyncQueueRepository.ts`

```ts
import { BaseRepository } from "./BaseRepository";
import { randomUUID } from "expo-crypto";

export interface SyncQueueRow {
  id: string;
  entity_type: string;
  entity_id: string;
  operation: "create" | "update" | "delete";
  payload: string;
  status: "pending" | "syncing" | "failed";
  retry_count: number;
  created_at: string;
  last_attempt_at: string | null;
}

class SyncQueueRepositoryClass extends BaseRepository<SyncQueueRow> {
  protected tableName = "sync_queue";

  async enqueue(
    entityType: string,
    entityId: string,
    operation: SyncQueueRow["operation"],
    payload: Record<string, unknown>,
  ): Promise<void> {
    const db = await this.db();
    await db.runAsync(
      `INSERT INTO sync_queue (id, entity_type, entity_id, operation, payload, status, retry_count, created_at)
       VALUES (?, ?, ?, ?, ?, 'pending', 0, ?)`,
      [
        randomUUID(),
        entityType,
        entityId,
        operation,
        JSON.stringify(payload),
        new Date().toISOString(),
      ],
    );
  }

  async getPending(): Promise<SyncQueueRow[]> {
    const db = await this.db();
    return db.getAllAsync<SyncQueueRow>(
      `SELECT * FROM sync_queue WHERE status != 'syncing' ORDER BY created_at ASC`,
    );
  }

  async markSynced(id: string): Promise<void> {
    await this.deleteById(id);
  }

  async markFailed(id: string): Promise<void> {
    const db = await this.db();
    await db.runAsync(
      `UPDATE sync_queue SET status = 'failed', retry_count = retry_count + 1, last_attempt_at = ? WHERE id = ?`,
      [new Date().toISOString(), id],
    );
  }
}

export const SyncQueueRepository = new SyncQueueRepositoryClass();
```

```bash
npx expo install expo-crypto
```

---

## Step 4.7 — Sync Engine Skeleton

এখন শুধু **skeleton** — actual per-feature sync logic (progress sync, bookmark sync) Step 9-এ পূর্ণাঙ্গভাবে হবে। এখানে শুধু কাঠামো, যাতে পরে প্রতিটা feature এটা reuse করতে পারে।

`src/sync/syncEngine.ts`

```ts
import {
  SyncQueueRepository,
  SyncQueueRow,
} from "../db/repositories/SyncQueueRepository";

const MAX_RETRIES = 3;

// প্রতিটা entity_type-এর জন্য backend-এ কীভাবে পাঠাতে হবে, সেটা feature module রেজিস্টার করবে
type SyncHandler = (item: SyncQueueRow) => Promise<void>;
const handlers = new Map<string, SyncHandler>();

export function registerSyncHandler(entityType: string, handler: SyncHandler) {
  handlers.set(entityType, handler);
}

let isSyncing = false;

export async function processSyncQueue(): Promise<void> {
  if (isSyncing) return; // একসাথে দুইবার চলবে না
  isSyncing = true;

  try {
    const pending = await SyncQueueRepository.getPending();

    for (const item of pending) {
      if (item.retry_count >= MAX_RETRIES) continue; // permanently failed, skip

      const handler = handlers.get(item.entity_type);
      if (!handler) continue; // এই entity_type এর জন্য এখনো handler রেজিস্টার হয়নি

      try {
        await handler(item);
        await SyncQueueRepository.markSynced(item.id);
      } catch {
        await SyncQueueRepository.markFailed(item.id);
      }
    }
  } finally {
    isSyncing = false;
  }
}
```

**ব্যবহার (উদাহরণ, এখন implement করার দরকার নেই — Step 9-এ হবে):**

```ts
registerSyncHandler("progress", async (item) => {
  const payload = JSON.parse(item.payload);
  await apiClient.post("/progress/sync", payload);
});
```

---

## Step 4.8 — App Root Wiring

`app/_layout.tsx`-এ (existing layout-এর ভেতরে যোগ করো):

```tsx
import { SQLiteProvider } from "expo-sqlite";
import { useEffect } from "react";
import { getDatabase } from "../src/db/connection";
import { runMigrations } from "../src/db/migrations";
import { useNetworkStore } from "../src/store/networkStore";
import { processSyncQueue } from "../src/sync/syncEngine";

async function initDatabase() {
  const db = await getDatabase();
  await runMigrations(db);
}

export default function RootLayout() {
  useEffect(() => {
    const unsubscribeNetwork = useNetworkStore.getState().init();

    // Online হওয়ার সাথে সাথে queue process করো
    const unsubscribeSync = useNetworkStore.subscribe((state) => {
      if (state.isOnline) {
        processSyncQueue();
      }
    });

    return () => {
      unsubscribeNetwork();
      unsubscribeSync();
    };
  }, []);

  return (
    <SQLiteProvider
      databaseName="arabic_learning.db"
      onInit={initDatabase}
      useSuspense
    >
      {/* বাকি app layout / Stack এখানে থাকবে */}
    </SQLiteProvider>
  );
}
```

**টেস্ট করো:**

1. App চালু করো — crash ছাড়া লোড হওয়া উচিত
2. Airplane mode অন করে `useNetworkStore.getState().isOnline` চেক করো (`false` হওয়া উচিত)
3. Airplane mode বন্ধ করলে `processSyncQueue()` অটো ট্রিগার হচ্ছে কিনা console log দিয়ে verify করো

---

## এই Step-এ যা হলো (সারসংক্ষেপ)

- **Versioned migration system** — schema change করলে পুরনো user-দের ডাটা নষ্ট হবে না
- **Full offline schema** — categories, lessons, words, sentences, progress, bookmarks, downloads সব local-এ mirror হচ্ছে
- **Repository pattern** — backend-এর মতোই architecture, business logic raw SQL থেকে আলাদা
- **Outbox pattern (`sync_queue`)** — অফলাইন action গুলো queue হয়ে থাকে, internet ফিরলে sync হয়
- **NetInfo + Zustand** — যেকোনো জায়গা থেকে reactive online/offline status
- **Sync engine skeleton** — handler registration pattern, যাতে প্রতিটা feature (progress, bookmarks) নিজের sync logic প্লাগ-ইন করতে পারে duplicate কোড ছাড়াই

**ব্রুটাল ট্রুথ:** `processSyncQueue`-এ এখনো conflict resolution নেই (যেমন — একই lesson দুই ডিভাইস থেকে আলাদাভাবে আপডেট হলে কী হবে)। এই app-এর scope-এ (single-user learning progress) এটা এখন দরকার নেই — "last write wins" যথেষ্ট। যদি ভবিষ্যতে multi-device conflict একটা সমস্যা হয়, তখন `updated_at` timestamp compare করে resolve করা যাবে।

**পরবর্তী যৌক্তিক ধাপ:** Step 5 — Arabic Content (Categories, Lessons, Words, Sentences) real data দিয়ে, backend API + MongoDB schema + এই SQLite repositories-এর সাথে wiring। এটাই প্রথম end-to-end vertical slice হবে যেটা তুমি আগে চিহ্নিত করেছিলে।
