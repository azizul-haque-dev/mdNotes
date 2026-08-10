# Fix: User Profile Local Cache (dailyGoalMinutes wiring)

একটা নতুন Zustand/AsyncStorage store না বানিয়ে, এখন পর্যন্ত established pattern-ই অনুসরণ করছি: **SQLite = local read source, API = background sync**। এতে consistency বজায় থাকলো এবং profile-ও offline-এ পড়া যাবে।

---

## 1. SQLite Migration: User Profile টেবিল

`src/db/migrations/006_add_user_profile.ts`

```ts
import { SQLiteDatabase } from "expo-sqlite";
import { Migration } from "./index";

export const migration_006_add_user_profile: Migration = {
  version: 6,
  name: "add_user_profile",
  up: async (db: SQLiteDatabase) => {
    await db.execAsync(`
      CREATE TABLE IF NOT EXISTS user_profile (
        id TEXT PRIMARY KEY DEFAULT 'me',
        display_name TEXT NOT NULL,
        avatar_url TEXT,
        preferred_language TEXT NOT NULL DEFAULT 'bn',
        daily_goal_minutes INTEGER NOT NULL DEFAULT 10,
        updated_at TEXT NOT NULL
      );
    `);
  },
};
```

`src/db/migrations/index.ts`-এ যোগ করো:

```ts
import { migration_006_add_user_profile } from "./006_add_user_profile";

export const migrations: Migration[] = [
  migration_001_init,
  migration_002_add_sentence_transliteration,
  migration_003_add_conversations,
  migration_004_add_srs,
  migration_005_add_daily_activity,
  migration_006_add_user_profile, // যোগ করো
];
```

---

## 2. Mobile API + Repository

`src/api/userApi.ts`

```ts
import { apiClient } from "./apiClient";

export interface UserProfileDto {
  displayName: string;
  avatarUrl?: string;
  preferredLanguage: "bn" | "en";
  dailyGoalMinutes: number;
}

export const userApi = {
  getProfile: async (): Promise<UserProfileDto> => {
    const { data } = await apiClient.get("/users/me");
    return data.data;
  },

  updatePreferences: async (updates: {
    preferredLanguage?: "bn" | "en";
    dailyGoalMinutes?: number;
  }): Promise<UserProfileDto> => {
    const { data } = await apiClient.patch("/users/preferences", updates);
    return data.data;
  },
};
```

`src/db/repositories/UserProfileRepository.ts`

```ts
import { getDatabase } from "../connection";

export interface UserProfileRow {
  id: string;
  display_name: string;
  avatar_url: string | null;
  preferred_language: "bn" | "en";
  daily_goal_minutes: number;
  updated_at: string;
}

export const UserProfileRepository = {
  async get(): Promise<UserProfileRow | null> {
    const db = await getDatabase();
    const row = await db.getFirstAsync<UserProfileRow>(
      "SELECT * FROM user_profile WHERE id = 'me'",
    );
    return row ?? null;
  },

  async upsert(profile: {
    display_name: string;
    avatar_url: string | null;
    preferred_language: "bn" | "en";
    daily_goal_minutes: number;
  }): Promise<void> {
    const db = await getDatabase();
    await db.runAsync(
      `INSERT INTO user_profile (id, display_name, avatar_url, preferred_language, daily_goal_minutes, updated_at)
       VALUES ('me', ?, ?, ?, ?, ?)
       ON CONFLICT(id) DO UPDATE SET
         display_name=excluded.display_name, avatar_url=excluded.avatar_url,
         preferred_language=excluded.preferred_language, daily_goal_minutes=excluded.daily_goal_minutes,
         updated_at=excluded.updated_at`,
      [
        profile.display_name,
        profile.avatar_url,
        profile.preferred_language,
        profile.daily_goal_minutes,
        new Date().toISOString(),
      ],
    );
  },
};
```

---

## 3. Sync + Hook

`src/features/profile/profileSync.ts`

```ts
import { userApi } from "../../api/userApi";
import { UserProfileRepository } from "../../db/repositories/UserProfileRepository";

export async function syncUserProfile(): Promise<void> {
  const remote = await userApi.getProfile();
  await UserProfileRepository.upsert({
    display_name: remote.displayName,
    avatar_url: remote.avatarUrl ?? null,
    preferred_language: remote.preferredLanguage,
    daily_goal_minutes: remote.dailyGoalMinutes,
  });
}
```

`src/features/profile/useUserProfile.ts`

```ts
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { useEffect } from "react";
import { UserProfileRepository } from "../../db/repositories/UserProfileRepository";
import { syncUserProfile } from "./profileSync";
import { useNetworkStore } from "../../store/networkStore";

export function useUserProfile() {
  const queryClient = useQueryClient();
  const isOnline = useNetworkStore((s) => s.isOnline);

  const query = useQuery({
    queryKey: ["userProfile", "local"],
    queryFn: () => UserProfileRepository.get(),
  });

  useEffect(() => {
    if (!isOnline) return;
    syncUserProfile()
      .then(() =>
        queryClient.invalidateQueries({ queryKey: ["userProfile", "local"] }),
      )
      .catch((err) => console.warn("Profile sync failed:", err));
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [isOnline]);

  return query;
}
```

---

## 4. Login-এর পরে সরাসরি Sync (Gap বন্ধ করা)

Step 2-এর auth flow-তে `sync-profile` কল হওয়ার পরপরই local cache-ও ভরে ফেলা উচিত, যাতে প্রথমবার Home screen খোলার আগেই ডাটা তৈরি থাকে। Auth success handler-এ (যেখানে `sync-profile` কল হয়) যোগ করো:

```ts
import { syncUserProfile } from "../features/profile/profileSync";

// sync-profile API success হওয়ার পরে:
await syncUserProfile().catch(() => {
  // silently fail — Home screen-এর background sync পরে আবার চেষ্টা করবে
});
```

---

## 5. Home Screen আপডেট (Hardcoded বাদ)

Step 9.10-এর `app/(app)/home/index.tsx`-এ পরিবর্তন করো:

```tsx
import { useUserProfile } from '../../../src/features/profile/useUserProfile';

// আগে ছিল: const DAILY_GOAL_MINUTES = 10; — এই লাইন সরিয়ে ফেলো

export default function HomeScreen() {
  const { data: stats, isLoading } = useStatistics();
  const { data: profile } = useUserProfile();
  // ...

  const dailyGoalMinutes = profile?.daily_goal_minutes ?? 10; // fallback শুধু profile load না হওয়া পর্যন্ত
  const goalProgress = Math.min(1, (stats?.todayMinutes ?? 0) / dailyGoalMinutes);

  // "আজকের লক্ষ্য" text-এও dailyGoalMinutes ব্যবহার করো hardcoded সংখ্যার বদলে
```

---

**টেস্ট করো:**

1. লগইন করো — `user_profile` SQLite টেবিলে সাথে সাথে row তৈরি হওয়া উচিত
2. Backend-এ MongoDB Compass দিয়ে সেই user-এর `dailyGoalMinutes` ম্যানুয়ালি বদলে দাও (যেমন ২০)
3. App-এ pull বা restart করো — Home screen-এর "আজকের লক্ষ্য" bar নতুন মান দেখানো উচিত (background sync কাজ করছে প্রমাণ)
4. Airplane mode-এ app খোলো — আগের cached `dailyGoalMinutes`-ই দেখা উচিত, crash না করে

---

**Gap বন্ধ হলো।** এখন Step 10 (Polish & Performance)-এ কোনো বাধা ছাড়াই যাওয়া যায়।
