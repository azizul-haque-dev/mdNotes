# Step 6: Audio System

`expo-audio` (playback) + `expo-file-system` নতুন API (download) + `downloads` SQLite টেবিল (Step 4-এ বানানো) দিয়ে সম্পূর্ণ offline-capable audio system।

---

## Step 6.1 — Dependencies

```bash
npx expo install expo-audio expo-file-system
```

---

## Step 6.2 — Audio Directory Setup

`src/audio/audioStorage.ts`

```ts
import { Directory, Paths } from "expo-file-system";

export const AUDIO_DIR = new Directory(Paths.document, "audio");

export function ensureAudioDir(): void {
  if (!AUDIO_DIR.exists) {
    AUDIO_DIR.create({ intermediates: true });
  }
}

// item_id (word/sentence Mongo _id) থেকে predictable local file path
export function getLocalAudioPath(itemId: string): string {
  return `${AUDIO_DIR.uri}/${itemId}.mp3`;
}
```

App শুরুতে একবার কল করো (`app/_layout.tsx`-এর `initDatabase`-এর পাশে):

```ts
import { ensureAudioDir } from "../src/audio/audioStorage";
ensureAudioDir();
```

---

## Step 6.3 — Download Repository (SQLite)

`src/db/repositories/DownloadRepository.ts`

```ts
import { BaseRepository } from "./BaseRepository";

export interface DownloadRow {
  id: string;
  item_type: "lesson_audio" | "word_audio";
  item_id: string;
  local_uri: string | null;
  status: "pending" | "downloading" | "completed" | "failed";
  downloaded_at: string | null;
}

class DownloadRepositoryClass extends BaseRepository<DownloadRow> {
  protected tableName = "downloads";

  async findByItemId(itemId: string): Promise<DownloadRow | null> {
    const db = await this.db();
    const row = await db.getFirstAsync<DownloadRow>(
      "SELECT * FROM downloads WHERE item_id = ?",
      [itemId],
    );
    return row ?? null;
  }

  async upsertStatus(
    itemId: string,
    itemType: DownloadRow["item_type"],
    status: DownloadRow["status"],
    localUri: string | null,
  ): Promise<void> {
    const db = await this.db();
    const existing = await this.findByItemId(itemId);

    if (existing) {
      await db.runAsync(
        `UPDATE downloads SET status = ?, local_uri = ?, downloaded_at = ? WHERE item_id = ?`,
        [
          status,
          localUri,
          status === "completed" ? new Date().toISOString() : null,
          itemId,
        ],
      );
    } else {
      await db.runAsync(
        `INSERT INTO downloads (id, item_type, item_id, local_uri, status, downloaded_at)
         VALUES (?, ?, ?, ?, ?, ?)`,
        [
          `dl_${itemId}`,
          itemType,
          itemId,
          localUri,
          status,
          status === "completed" ? new Date().toISOString() : null,
        ],
      );
    }
  }

  async getAllCompleted(): Promise<DownloadRow[]> {
    const db = await this.db();
    return db.getAllAsync<DownloadRow>(
      `SELECT * FROM downloads WHERE status = 'completed'`,
    );
  }
}

export const DownloadRepository = new DownloadRepositoryClass();
```

---

## Step 6.4 — Download Manager (progress সহ)

`src/audio/downloadManager.ts`

```ts
import { File } from "expo-file-system";
import { AUDIO_DIR, getLocalAudioPath } from "./audioStorage";
import { DownloadRepository } from "../db/repositories/DownloadRepository";

interface DownloadOptions {
  onProgress?: (progress: number) => void; // 0 থেকে 1
}

export async function downloadAudio(
  itemId: string,
  itemType: "lesson_audio" | "word_audio",
  remoteUrl: string,
  options: DownloadOptions = {},
): Promise<string> {
  await DownloadRepository.upsertStatus(itemId, itemType, "downloading", null);

  const destination = new File(AUDIO_DIR, `${itemId}.mp3`);

  try {
    const task = File.createDownloadTask(remoteUrl, destination, {
      onProgress: ({ bytesWritten, totalBytes }) => {
        if (totalBytes > 0 && options.onProgress) {
          options.onProgress(bytesWritten / totalBytes);
        }
      },
    });

    const file = await task.downloadAsync();
    const localUri = file.uri;

    await DownloadRepository.upsertStatus(
      itemId,
      itemType,
      "completed",
      localUri,
    );
    return localUri;
  } catch (err) {
    await DownloadRepository.upsertStatus(itemId, itemType, "failed", null);
    throw err;
  }
}

export async function deleteDownloadedAudio(itemId: string): Promise<void> {
  const record = await DownloadRepository.findByItemId(itemId);
  if (record?.local_uri) {
    const file = new File(record.local_uri);
    if (file.exists) file.delete();
  }
  await DownloadRepository.upsertStatus(itemId, "word_audio", "pending", null);
}

// getLocalAudioPath ব্যবহার না করেও predictable path হওয়ায় existence সরাসরি চেক করা যায়
export function localFileExists(itemId: string): boolean {
  const file = new File(getLocalAudioPath(itemId));
  return file.exists;
}
```

---

## Step 6.5 — Download Hook (UI progress-এর জন্য)

`src/audio/useDownloadAudio.ts`

```ts
import { useState, useCallback, useEffect } from "react";
import { downloadAudio, deleteDownloadedAudio } from "./downloadManager";
import {
  DownloadRepository,
  DownloadRow,
} from "../db/repositories/DownloadRepository";

export function useDownloadAudio(
  itemId: string,
  itemType: "lesson_audio" | "word_audio",
  remoteUrl?: string,
) {
  const [status, setStatus] = useState<DownloadRow["status"]>("pending");
  const [progress, setProgress] = useState(0);

  useEffect(() => {
    DownloadRepository.findByItemId(itemId).then((record) => {
      if (record) setStatus(record.status);
    });
  }, [itemId]);

  const download = useCallback(async () => {
    if (!remoteUrl) return;
    setStatus("downloading");
    setProgress(0);
    try {
      await downloadAudio(itemId, itemType, remoteUrl, {
        onProgress: setProgress,
      });
      setStatus("completed");
    } catch {
      setStatus("failed");
    }
  }, [itemId, itemType, remoteUrl]);

  const remove = useCallback(async () => {
    await deleteDownloadedAudio(itemId);
    setStatus("pending");
    setProgress(0);
  }, [itemId]);

  return { status, progress, download, remove };
}
```

---

## Step 6.6 — Global Audio Player (Context + Provider)

একটাই player instance পুরো app-এ — যাতে একসাথে দুইটা audio না বাজে এবং mini-player future-এ সহজে বানানো যায়।

`src/audio/AudioPlayerContext.tsx`

```tsx
import {
  createContext,
  useContext,
  useState,
  useCallback,
  ReactNode,
} from "react";
import { useAudioPlayer, useAudioPlayerStatus } from "expo-audio";
import { localFileExists, getLocalAudioPath } from "./audioStorage";

interface AudioPlayerContextValue {
  currentItemId: string | null;
  isPlaying: boolean;
  isLoading: boolean;
  currentTime: number;
  duration: number;
  play: (itemId: string, remoteUrl: string) => void;
  pause: () => void;
  toggle: (itemId: string, remoteUrl: string) => void;
}

const AudioPlayerContext = createContext<AudioPlayerContextValue | null>(null);

export function AudioPlayerProvider({ children }: { children: ReactNode }) {
  const player = useAudioPlayer(null);
  const status = useAudioPlayerStatus(player);
  const [currentItemId, setCurrentItemId] = useState<string | null>(null);
  const [isLoading, setIsLoading] = useState(false);

  const play = useCallback(
    (itemId: string, remoteUrl: string) => {
      // Offline-first: local file থাকলে সেটাই ব্যবহার হবে, নাহলে stream
      const source = localFileExists(itemId)
        ? getLocalAudioPath(itemId)
        : remoteUrl;

      setIsLoading(true);
      setCurrentItemId(itemId);
      player.replace(source);
      player.play();
      setIsLoading(false);
    },
    [player],
  );

  const pause = useCallback(() => {
    player.pause();
  }, [player]);

  const toggle = useCallback(
    (itemId: string, remoteUrl: string) => {
      if (currentItemId === itemId && status.playing) {
        pause();
      } else {
        play(itemId, remoteUrl);
      }
    },
    [currentItemId, status.playing, play, pause],
  );

  return (
    <AudioPlayerContext.Provider
      value={{
        currentItemId,
        isPlaying: status.playing,
        isLoading,
        currentTime: status.currentTime,
        duration: status.duration,
        play,
        pause,
        toggle,
      }}
    >
      {children}
    </AudioPlayerContext.Provider>
  );
}

export function useGlobalAudioPlayer(): AudioPlayerContextValue {
  const ctx = useContext(AudioPlayerContext);
  if (!ctx)
    throw new Error(
      "useGlobalAudioPlayer must be used within AudioPlayerProvider",
    );
  return ctx;
}
```

`app/_layout.tsx`-এ wrap করো (SQLiteProvider-এর ভেতরে):

```tsx
import { AudioPlayerProvider } from "../src/audio/AudioPlayerContext";

// ...
<SQLiteProvider
  databaseName="arabic_learning.db"
  onInit={initDatabase}
  useSuspense
>
  <AudioPlayerProvider>{/* বাকি app */}</AudioPlayerProvider>
</SQLiteProvider>;
```

**ব্রুটাল ট্রুথ:** `expo-audio`-এর `player.replace()` মেথডে পুরনো ভার্সনগুলোতে Android-এ bug ছিল (iOS-এ ঠিক থাকলেও)। SDK 56-এ ঠিক হয়ে থাকার কথা, কিন্তু Android physical device-এ track change করে টেস্ট করে নিশ্চিত হয়ে নাও।

---

## Step 6.7 — Reusable Play Button Component

`src/components/AudioPlayButton.tsx`

```tsx
import { Pressable, Text, ActivityIndicator, View } from "react-native";
import { useGlobalAudioPlayer } from "../audio/AudioPlayerContext";

interface Props {
  itemId: string;
  audioUrl?: string;
}

export function AudioPlayButton({ itemId, audioUrl }: Props) {
  const { currentItemId, isPlaying, isLoading, toggle } =
    useGlobalAudioPlayer();

  if (!audioUrl) return null;

  const isActive = currentItemId === itemId;
  const showLoading = isActive && isLoading;
  const showPlaying = isActive && isPlaying;

  return (
    <Pressable
      onPress={() => toggle(itemId, audioUrl)}
      className="w-10 h-10 rounded-full bg-emerald-500 items-center justify-center"
    >
      {showLoading ? (
        <ActivityIndicator color="white" size="small" />
      ) : (
        <Text className="text-white text-lg">{showPlaying ? "⏸" : "▶"}</Text>
      )}
    </Pressable>
  );
}
```

---

## Step 6.8 — Download Button Component (Progress সহ)

`src/components/AudioDownloadButton.tsx`

```tsx
import { Pressable, Text, View } from "react-native";
import { useDownloadAudio } from "../audio/useDownloadAudio";

interface Props {
  itemId: string;
  audioUrl?: string;
}

export function AudioDownloadButton({ itemId, audioUrl }: Props) {
  const { status, progress, download, remove } = useDownloadAudio(
    itemId,
    "word_audio",
    audioUrl,
  );

  if (!audioUrl) return null;

  if (status === "completed") {
    return (
      <Pressable
        onPress={remove}
        className="px-3 py-1 rounded-full bg-gray-200 dark:bg-gray-700"
      >
        <Text className="text-xs text-gray-600 dark:text-gray-300">
          ডাউনলোড হয়েছে ✓
        </Text>
      </Pressable>
    );
  }

  if (status === "downloading") {
    return (
      <View className="px-3 py-1 rounded-full bg-gray-100 dark:bg-gray-800">
        <Text className="text-xs text-gray-500">
          {Math.round(progress * 100)}%
        </Text>
      </View>
    );
  }

  return (
    <Pressable
      onPress={download}
      className="px-3 py-1 rounded-full bg-blue-100 dark:bg-blue-900"
    >
      <Text className="text-xs text-blue-600 dark:text-blue-300">
        {status === "failed" ? "আবার চেষ্টা করো" : "ডাউনলোড করো"}
      </Text>
    </Pressable>
  );
}
```

---

## Step 6.9 — Word Detail Screen-এ Wire করা

Step 5.18-এর `app/(app)/words/[wordId]/index.tsx`-এ যোগ করো:

```tsx
import { AudioPlayButton } from "../../../src/components/AudioPlayButton";
import { AudioDownloadButton } from "../../../src/components/AudioDownloadButton";

// Arabic word-এর নিচে যোগ করো:
<View className="flex-row items-center gap-3 mb-4">
  <AudioPlayButton itemId={word.id} audioUrl={word.audio_url ?? undefined} />
  <AudioDownloadButton
    itemId={word.id}
    audioUrl={word.audio_url ?? undefined}
  />
</View>;
```

---

## Step 6.10 — Cache Management Screen

`app/(app)/settings/audio-cache.tsx`

```tsx
import { View, Text, Pressable, FlatList } from "react-native";
import { useEffect, useState } from "react";
import { File } from "expo-file-system";
import {
  DownloadRepository,
  DownloadRow,
} from "../../../src/db/repositories/DownloadRepository";
import { AUDIO_DIR } from "../../../src/audio/audioStorage";

function formatBytes(bytes: number): string {
  if (bytes < 1024) return `${bytes} B`;
  if (bytes < 1024 * 1024) return `${(bytes / 1024).toFixed(1)} KB`;
  return `${(bytes / (1024 * 1024)).toFixed(1)} MB`;
}

export default function AudioCacheScreen() {
  const [downloads, setDownloads] = useState<DownloadRow[]>([]);
  const [totalSize, setTotalSize] = useState(0);

  const loadDownloads = async () => {
    const completed = await DownloadRepository.getAllCompleted();
    setDownloads(completed);

    let size = 0;
    for (const d of completed) {
      if (d.local_uri) {
        const file = new File(d.local_uri);
        if (file.exists) size += file.size ?? 0;
      }
    }
    setTotalSize(size);
  };

  useEffect(() => {
    loadDownloads();
  }, []);

  const clearAllCache = async () => {
    for (const item of AUDIO_DIR.list()) {
      item.delete();
    }
    // SQLite-এ status রিসেট করো
    for (const d of downloads) {
      await DownloadRepository.upsertStatus(
        d.item_id,
        d.item_type,
        "pending",
        null,
      );
    }
    loadDownloads();
  };

  return (
    <View className="flex-1 bg-white dark:bg-gray-900 p-6">
      <Text className="text-lg font-semibold text-gray-900 dark:text-white mb-1">
        মোট ডাউনলোড করা অডিও: {downloads.length}টি
      </Text>
      <Text className="text-gray-500 dark:text-gray-400 mb-6">
        মোট জায়গা ব্যবহার হচ্ছে: {formatBytes(totalSize)}
      </Text>

      <Pressable
        onPress={clearAllCache}
        className="bg-red-500 rounded-xl py-3 items-center mb-6"
        disabled={downloads.length === 0}
      >
        <Text className="text-white font-semibold">সব ডাউনলোড মুছে ফেলো</Text>
      </Pressable>

      <FlatList
        data={downloads}
        keyExtractor={(item) => item.id}
        renderItem={({ item }) => (
          <View className="py-2 border-b border-gray-100 dark:border-gray-800">
            <Text className="text-gray-700 dark:text-gray-300">
              {item.item_id}
            </Text>
          </View>
        )}
      />
    </View>
  );
}
```

**টেস্ট করো:**

1. Word Detail স্ক্রিনে play বাটনে চাপো — audio stream হয়ে বাজা উচিত
2. Download বাটনে চাপো — progress percentage দেখা উচিত, শেষে "ডাউনলোড হয়েছে ✓"
3. Airplane mode অন করে আবার play করো — এবার local file থেকে বাজা উচিত (network ছাড়াই)
4. Cache management স্ক্রিনে গিয়ে total size আর "সব মুছে ফেলো" টেস্ট করো

---

## এই Step-এ যা হলো (সারসংক্ষেপ)

- `expo-audio` দিয়ে global single-instance player (একসাথে দুইটা audio বাজবে না)
- `expo-file-system`-এর নতুন class-based API দিয়ে progress-সহ download
- **Offline-first playback logic**: local file থাকলে local, না থাকলে stream — network অবস্থা নির্বিশেষে best-effort playback
- `downloads` টেবিল (Step 4) দিয়ে download status track — app restart করলেও state থাকে
- Cache management স্ক্রিন — total size দেখা ও clear করার সুবিধা
- Reusable `AudioPlayButton` ও `AudioDownloadButton` component — Lessons, Flashcards (Step 8) সব জায়গায় reuse হবে

**ব্রুটাল ট্রুথ:** এখানে **bulk/batch download** (পুরো lesson-এর সব word audio একসাথে ডাউনলোড) নেই — এখন word-by-word download। যদি ইউজার experience হিসেবে "পুরো lesson offline করো" বাটন দরকার মনে করো, সেটা ছোট addition হিসেবে পরে যোগ করা যাবে (download queue loop করে)। এখন base building block ঠিকভাবে তৈরি হয়ে গেছে, সেটাই গুরুত্বপূর্ণ ছিল।

**পরবর্তী যৌক্তিক ধাপ:** Step 7 — Learning Flow (Lesson → Word → Sentence → Conversation → Progress tracking), যেখানে এখন পর্যন্ত বানানো সব piece (SQLite, sync, audio) একসাথে একটা কোহেসিভ learning experience-এ যুক্ত হবে।
