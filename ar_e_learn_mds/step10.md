# Step 10: Polish & Performance

**একটা গুরুত্বপূর্ণ পর্যবেক্ষণ শুরুতেই:** Step 5 থেকে Step 9 পর্যন্ত প্রায় প্রতিটা screen-এ loading/empty/error state আলাদা করে লেখা হয়েছে (একই `<Text>লোড হচ্ছে...</Text>` প্যাটার্ন ৮ বারের বেশি copy-paste হয়েছে)। এটা তোমার নিজের "Never duplicate business logic" আর "No duplicated code" rule ভাঙে। Step 10-এর প্রধান কাজ এই duplication clean করা, তারপর Dark Mode, animation, accessibility, performance।

**নতুন library লাগবে না** — Nativewind v4-এর built-in dark mode API আর React Native-এর built-in `Animated` API-ই যথেষ্ট।

---

## Step 10.1 — Reusable Async State Component (Duplication সমাধান)

`src/components/AsyncStateView.tsx`

```tsx
import { View, Text } from "react-native";
import { ReactNode } from "react";
import { SkeletonList } from "./SkeletonList";

interface Props<T> {
  isLoading: boolean;
  isError: boolean;
  data: T[] | null | undefined;
  loadingLabel?: string;
  errorLabel?: string;
  emptyLabel?: string;
  skeletonCount?: number;
  children: (data: T[]) => ReactNode;
}

export function AsyncStateView<T>({
  isLoading,
  isError,
  data,
  errorLabel = "কিছু একটা সমস্যা হয়েছে। আবার চেষ্টা করো।",
  emptyLabel = "এখনো কিছু নেই।",
  skeletonCount = 5,
  children,
}: Props<T>) {
  if (isLoading) {
    return <SkeletonList count={skeletonCount} />;
  }

  if (isError) {
    return (
      <View
        className="flex-1 items-center justify-center px-6"
        accessibilityRole="alert"
        accessibilityLabel={errorLabel}
      >
        <Text className="text-red-500 text-center">{errorLabel}</Text>
      </View>
    );
  }

  if (!data || data.length === 0) {
    return (
      <View className="flex-1 items-center justify-center px-6">
        <Text
          className="text-gray-400 text-center"
          accessibilityLabel={emptyLabel}
        >
          {emptyLabel}
        </Text>
      </View>
    );
  }

  return <>{children(data)}</>;
}
```

**এইভাবে ব্যবহার করো** (Categories screen, Step 5.11 থেকে refactor):

```tsx
<AsyncStateView
  isLoading={isLoading}
  isError={isError}
  data={categories}
  emptyLabel="এখনো কোনো ক্যাটাগরি যোগ হয়নি।"
>
  {(data) => (
    <FlatList
      data={data}
      keyExtractor={(item) => item.id}
      refreshControl={
        <RefreshControl refreshing={refreshing} onRefresh={onRefresh} />
      }
      renderItem={({ item }) => <CategoryCard category={item} />}
    />
  )}
</AsyncStateView>
```

একই প্যাটার্ন Lessons, Words, Bookmarks — সব browsing screen-এ apply করো, প্রতিটা থেকে ৮-১০ লাইন করে boilerplate সরে যাবে।

---

## Step 10.2 — Skeleton Loading (Shimmer, বিনা extra library)

`src/components/SkeletonList.tsx`

```tsx
import { View, Animated } from "react-native";
import { useEffect, useRef } from "react";

function useShimmer() {
  const opacity = useRef(new Animated.Value(0.3)).current;

  useEffect(() => {
    const loop = Animated.loop(
      Animated.sequence([
        Animated.timing(opacity, {
          toValue: 1,
          duration: 700,
          useNativeDriver: true,
        }),
        Animated.timing(opacity, {
          toValue: 0.3,
          duration: 700,
          useNativeDriver: true,
        }),
      ]),
    );
    loop.start();
    return () => loop.stop();
  }, [opacity]);

  return opacity;
}

function SkeletonRow() {
  const opacity = useShimmer();
  return (
    <Animated.View
      style={{ opacity }}
      className="bg-gray-200 dark:bg-gray-800 rounded-xl h-20 mb-3"
    />
  );
}

export function SkeletonList({ count = 5 }: { count?: number }) {
  return (
    <View
      className="p-4"
      accessibilityLabel="লোড হচ্ছে"
      accessibilityRole="progressbar"
    >
      {Array.from({ length: count }).map((_, i) => (
        <SkeletonRow key={i} />
      ))}
    </View>
  );
}
```

এটা আগের সব screen-এর `<Text>লোড হচ্ছে...</Text>` replace করবে — user experience অনেক বেশি polished মনে হবে ("perceived performance")।

---

## Step 10.3 — Dark Mode (System-aware + Manual Override)

**Decision:** Device settings অনুসরণ করাই default (recommended approach), কিন্তু user চাইলে app-এর ভেতর থেকে override করতে পারবে — সেই override টা device-local settings-এ (নতুন ছোট SQLite টেবিল, backend sync দরকার নেই কারণ এটা per-device UI preference, per-user data না)।

`src/db/migrations/007_add_app_settings.ts`

```ts
import { SQLiteDatabase } from "expo-sqlite";
import { Migration } from "./index";

export const migration_007_add_app_settings: Migration = {
  version: 7,
  name: "add_app_settings",
  up: async (db: SQLiteDatabase) => {
    await db.execAsync(`
      CREATE TABLE IF NOT EXISTS app_settings (
        key TEXT PRIMARY KEY,
        value TEXT NOT NULL
      );
    `);
  },
};
```

`migrations/index.ts`-এ যোগ করো (আগের migration গুলোর তালিকায় শেষে):

```ts
import { migration_007_add_app_settings } from "./007_add_app_settings";
// migrations array-তে migration_006_add_user_profile-এর পরে যোগ করো
```

`src/theme/ThemeContext.tsx`

```tsx
import {
  createContext,
  useContext,
  useEffect,
  useState,
  ReactNode,
} from "react";
import { useColorScheme as useNativewindColorScheme } from "nativewind";
import { getDatabase } from "../db/connection";

type ThemePreference = "light" | "dark" | "system";

interface ThemeContextValue {
  preference: ThemePreference;
  setPreference: (pref: ThemePreference) => void;
}

const ThemeContext = createContext<ThemeContextValue | null>(null);

async function loadSavedPreference(): Promise<ThemePreference> {
  const db = await getDatabase();
  const row = await db.getFirstAsync<{ value: string }>(
    "SELECT value FROM app_settings WHERE key = 'theme_preference'",
  );
  return (row?.value as ThemePreference) ?? "system";
}

async function savePreference(pref: ThemePreference): Promise<void> {
  const db = await getDatabase();
  await db.runAsync(
    `INSERT INTO app_settings (key, value) VALUES ('theme_preference', ?)
     ON CONFLICT(key) DO UPDATE SET value = excluded.value`,
    [pref],
  );
}

export function ThemeProvider({ children }: { children: ReactNode }) {
  const { setColorScheme } = useNativewindColorScheme();
  const [preference, setPreferenceState] = useState<ThemePreference>("system");

  useEffect(() => {
    loadSavedPreference().then((pref) => {
      setPreferenceState(pref);
      setColorScheme(pref);
    });
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, []);

  const setPreference = (pref: ThemePreference) => {
    setPreferenceState(pref);
    setColorScheme(pref);
    savePreference(pref);
  };

  return (
    <ThemeContext.Provider value={{ preference, setPreference }}>
      {children}
    </ThemeContext.Provider>
  );
}

export function useTheme(): ThemeContextValue {
  const ctx = useContext(ThemeContext);
  if (!ctx) throw new Error("useTheme must be used within ThemeProvider");
  return ctx;
}
```

`app/_layout.tsx`-এ wrap করো (সবচেয়ে বাইরে, `AudioPlayerProvider`-এর পাশে):

```tsx
import { ThemeProvider } from "../src/theme/ThemeContext";

<SQLiteProvider
  databaseName="arabic_learning.db"
  onInit={initDatabase}
  useSuspense
>
  <ThemeProvider>
    <AudioPlayerProvider>{/* বাকি app */}</AudioPlayerProvider>
  </ThemeProvider>
</SQLiteProvider>;
```

`app.json`-এ নিশ্চিত করো:

```json
{
  "expo": {
    "userInterfaceStyle": "automatic"
  }
}
```

**Settings screen-এ toggle:**

```tsx
import { useTheme } from "../../../src/theme/ThemeContext";

const { preference, setPreference } = useTheme();

// তিনটা বাটন: 'light' | 'dark' | 'system' — setPreference(value) কল করলেই পুরো app বদলে যাবে
```

---

## Step 10.4 — Simple Animations (Built-in Animated API)

`src/components/FadeInView.tsx`

```tsx
import { Animated, ViewProps } from "react-native";
import { useEffect, useRef } from "react";

export function FadeInView({ children, style, ...rest }: ViewProps) {
  const opacity = useRef(new Animated.Value(0)).current;
  const translateY = useRef(new Animated.Value(8)).current;

  useEffect(() => {
    Animated.parallel([
      Animated.timing(opacity, {
        toValue: 1,
        duration: 250,
        useNativeDriver: true,
      }),
      Animated.timing(translateY, {
        toValue: 0,
        duration: 250,
        useNativeDriver: true,
      }),
    ]).start();
  }, [opacity, translateY]);

  return (
    <Animated.View
      style={[style, { opacity, transform: [{ translateY }] }]}
      {...rest}
    >
      {children}
    </Animated.View>
  );
}
```

Card/List item render হওয়ার সময় wrap করো (যেমন Categories/Words card):

```tsx
<FadeInView>
  <Pressable className="bg-gray-100 dark:bg-gray-800 rounded-xl p-4 mb-3">
    {/* ... */}
  </Pressable>
</FadeInView>
```

**Quiz-এ answer select করার সময়** simple scale feedback (Step 8.6-এর `quiz.tsx`-এ option button-এ):

```tsx
const scale = useRef(new Animated.Value(1)).current;

const selectAnswer = async (option: string) => {
  if (selected) return;
  Animated.sequence([
    Animated.timing(scale, {
      toValue: 0.96,
      duration: 80,
      useNativeDriver: true,
    }),
    Animated.timing(scale, { toValue: 1, duration: 80, useNativeDriver: true }),
  ]).start();
  // ... বাকি লজিক আগের মতোই
};
```

**ব্রুটাল ট্রুথ:** Screen transition animation (route থেকে route-এ) Expo Router নিজেই handle করে (native stack transitions), সেটা নিয়ে আলাদা কাজ করার দরকার নেই। এখানে যা করা হলো তা **micro-interactions** — এগুলোই আসলে "polish" অনুভূতি তৈরি করে, বড় animation library দরকার হয় না।

---

## Step 10.5 — Accessibility

মূলনীতি: প্রতিটা interactive element-এ `accessibilityRole` + `accessibilityLabel`, প্রতিটা image-only button-এ label (screen reader-এর জন্য টেক্সট বিকল্প)।

**উদাহরণ — আগে বানানো কিছু component আপডেট করো:**

`AudioPlayButton` (Step 6.7):

```tsx
<Pressable
  onPress={() => toggle(itemId, audioUrl)}
  accessibilityRole="button"
  accessibilityLabel={showPlaying ? 'অডিও থামাও' : 'অডিও চালাও'}
  className="w-10 h-10 rounded-full bg-emerald-500 items-center justify-center"
>
```

`BookmarkButton` (Step 9.8):

```tsx
<Pressable
  onPress={handleToggle}
  accessibilityRole="button"
  accessibilityLabel={bookmarked ? 'বুকমার্ক থেকে সরাও' : 'বুকমার্ক করো'}
  accessibilityState={{ selected: bookmarked }}
  className="w-10 h-10 items-center justify-center"
>
```

MCQ options (Step 8.6):

```tsx
<Pressable
  onPress={() => selectAnswer(option)}
  accessibilityRole="radio"
  accessibilityState={{ selected: isSelected, disabled: !!selected }}
  accessibilityLabel={option}
>
```

**Checklist (বাকি component-গুলোতে একই প্যাটার্নে apply করো):**

- সব `Pressable`/`TouchableOpacity`-তে `accessibilityRole="button"`
- Icon-only বাটনে (যেমন play/pause শুধু ▶/⏸ symbol) সবসময় `accessibilityLabel` থাকা আবশ্যক
- Text input-এ `accessibilityLabel` (Search screen-এ placeholder ছাড়াও)
- Progress bar-এ `accessibilityRole="progressbar"` + `accessibilityValue={{ min: 0, max: 100, now: progress }}`
- Color-ই একমাত্র signal না হওয়া উচিত (Quiz-এ correct/wrong শুধু color না, একটা ✓/✗ icon-ও যোগ করো ভালো হয়)
- Minimum touch target ৪৪×৪৪ px রাখা (এখন পর্যন্ত ব্যবহৃত `w-10 h-10` = ৪০px, ধারাবাহিকভাবে `w-11 h-11` বা padding বাড়িয়ে ৪৪+ করে নাও)

---

## Step 10.6 — Performance Optimization

**FlatList tuning** — সব browsing list-এ (Categories, Lessons, Words, Bookmarks) যোগ করো:

```tsx
<FlatList
  data={data}
  keyExtractor={(item) => item.id}
  renderItem={renderItem}
  initialNumToRender={10}
  maxToRenderPerBatch={10}
  windowSize={5}
  removeClippedSubviews
  // ডাটা ছোট আকারের (word/lesson count) হওয়ায় getItemLayout দরকার নেই এখন —
  // যদি ভবিষ্যতে হাজার হাজার item হয় তখন fixed-height item-এর জন্য যোগ করা যাবে
/>
```

**`renderItem` memoize করো** — প্রতিবার re-render-এ নতুন function তৈরি হওয়া বন্ধ করতে (Categories screen উদাহরণ):

```tsx
import { memo, useCallback } from "react";

const CategoryCard = memo(function CategoryCard({
  category,
  onPress,
}: {
  category: CategoryRow;
  onPress: () => void;
}) {
  return (
    <Pressable
      onPress={onPress}
      className="bg-gray-100 dark:bg-gray-800 rounded-xl p-4 mb-3"
    >
      <Text className="text-lg font-semibold text-gray-900 dark:text-white">
        {category.name_bn}
      </Text>
      <Text className="text-gray-500 dark:text-gray-400">
        {category.name_en}
      </Text>
    </Pressable>
  );
});

// স্ক্রিনের ভেতরে:
const renderItem = useCallback(
  ({ item }: { item: CategoryRow }) => (
    <CategoryCard
      category={item}
      onPress={() => router.push(`/categories/${item.id}/lessons`)}
    />
  ),
  [],
);
```

**TanStack Query cache tuning** — content খুব একটা পরিবর্তন হয় না (lessons/words), তাই aggressive refetch দরকার নেই:

```ts
// useCategories, useLessons, useWords hooks-এ যোগ করো
useQuery({
  queryKey: [...],
  queryFn: ...,
  staleTime: 5 * 60 * 1000, // ৫ মিনিট পর্যন্ত fresh ধরা হবে, বারবার re-query হবে না
});
```

**Database query batching** — Step 5-এ Words/Sentences আলাদা query-তে আনা হতো; Word Detail screen-এ দুইটা আলাদা hook (word + sentences) parallel-এই চলে যেহেতু TanStack Query independent queries হিসেবে চালায়, এখানে অতিরিক্ত অপ্টিমাইজেশনের দরকার নেই — কিন্তু ভবিষ্যতে বড় lesson-এ (৫০+ word) pagination যোগ করা উচিত `useWords`-এ (এখন সব একসাথে লোড হয়)।

---

**টেস্ট করো:**

1. Device dark mode অন/অফ করো — app automatically বদলে যাওয়া উচিত (`userInterfaceStyle: automatic` কাজ করছে প্রমাণ)
2. Settings থেকে manual override করো (dark পছন্দ করলে device light mode-এও app dark থাকা উচিত), app restart করো — পছন্দ persist হওয়া উচিত
3. স্লো নেটওয়ার্কে (throttle করে) কোনো screen খোলো — shimmer skeleton দেখা উচিত, blank সাদা স্ক্রিন না
4. VoiceOver (iOS) / TalkBack (Android) অন করে Word Detail screen ঘুরে দেখো — play/bookmark বাটনের label শোনা উচিত
5. একটা lesson-এ ৫০+ শব্দ (test data দিয়ে) ঢুকিয়ে scroll করো — jank/lag ছাড়া স্মুথ scroll হওয়া উচিত

---

## এই Step-এ যা হলো (সারসংক্ষেপ)

- `AsyncStateView` + `SkeletonList` — ৮+ screen জুড়ে ছড়িয়ে থাকা duplicate loading/empty/error code এক জায়গায় centralize হলো
- System-aware Dark Mode + manual override, device-local SQLite-এ persist (backend sync দরকার নেই, per-device preference)
- Built-in `Animated` API দিয়ে micro-interactions (fade-in card, button press feedback) — নতুন heavy library ছাড়াই
- Accessibility — `accessibilityRole`/`accessibilityLabel` প্যাটার্ন প্রতিষ্ঠিত হলো, checklist দেওয়া হলো বাকি সব component-এ apply করার জন্য
- FlatList tuning, `React.memo`, TanStack Query `staleTime` — অপ্রয়োজনীয় re-render ও re-fetch কমানো হলো

**ব্রুটাল ট্রুথ:** এই Step-এ আমি সব ৮-১০টা screen-এ line-by-line গিয়ে refactor করে দেখাইনি (Categories screen-টা উদাহরণ হিসেবে দেখানো হয়েছে) — কারণ সেটা purely mechanical repetition হতো, একই কাজ ৮ বার। প্যাটার্নটা এখন স্পষ্ট, তুমি (বা আমি পরের ছোট ধাপে) বাকি screen-গুলোতে `AsyncStateView` আর accessibility props apply করে দিতে পারি যদি চাও।

**পরবর্তী যৌক্তিক ধাপ:** Step 11 — Testing & Deployment (Unit tests, API tests, EAS Build, Backend deployment, Production checklist), যেটা পুরো ১০টা step-এর কাজকে production-এ নেওয়ার জন্য শেষ ধাপ। অথবা চাইলে আগে বাকি screen গুলোতে এই Step 10-এর প্যাটার্ন apply করে "cleanup pass" করতে পারি।
