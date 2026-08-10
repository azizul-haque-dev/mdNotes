# Step 5: Arabic Content (Categories → Lessons → Words → Sentences)

এটাই প্রথম **end-to-end vertical slice**। প্যাটার্ন একবার ঠিকভাবে বানালে Lessons/Words/Sentences একই কাঠামো অনুসরণ করবে।

---

# BACKEND

## Step 5.1 — MongoDB Schemas

`src/models/Category.ts`

```ts
import { Schema, model, Types } from "mongoose";

export interface ICategory {
  _id: Types.ObjectId;
  nameAr: string;
  nameBn: string;
  nameEn: string;
  slug: string;
  icon?: string;
  sortOrder: number;
  isActive: boolean;
  createdAt: Date;
  updatedAt: Date;
}

const categorySchema = new Schema<ICategory>(
  {
    nameAr: { type: String, required: true, trim: true },
    nameBn: { type: String, required: true, trim: true },
    nameEn: { type: String, required: true, trim: true },
    slug: {
      type: String,
      required: true,
      unique: true,
      lowercase: true,
      trim: true,
    },
    icon: { type: String },
    sortOrder: { type: Number, default: 0 },
    isActive: { type: Boolean, default: true },
  },
  { timestamps: true },
);

categorySchema.index({ sortOrder: 1 });

export const CategoryModel = model<ICategory>("Category", categorySchema);
```

`src/models/Lesson.ts`

```ts
import { Schema, model, Types } from "mongoose";

export interface ILesson {
  _id: Types.ObjectId;
  categoryId: Types.ObjectId;
  titleAr: string;
  titleBn: string;
  titleEn: string;
  description?: string;
  difficulty: "beginner" | "intermediate" | "advanced";
  sortOrder: number;
  isActive: boolean;
  createdAt: Date;
  updatedAt: Date;
}

const lessonSchema = new Schema<ILesson>(
  {
    categoryId: {
      type: Schema.Types.ObjectId,
      ref: "Category",
      required: true,
      index: true,
    },
    titleAr: { type: String, required: true, trim: true },
    titleBn: { type: String, required: true, trim: true },
    titleEn: { type: String, required: true, trim: true },
    description: { type: String },
    difficulty: {
      type: String,
      enum: ["beginner", "intermediate", "advanced"],
      default: "beginner",
    },
    sortOrder: { type: Number, default: 0 },
    isActive: { type: Boolean, default: true },
  },
  { timestamps: true },
);

lessonSchema.index({ categoryId: 1, sortOrder: 1 });

export const LessonModel = model<ILesson>("Lesson", lessonSchema);
```

`src/models/Word.ts`

```ts
import { Schema, model, Types } from "mongoose";

export interface IWord {
  _id: Types.ObjectId;
  lessonId: Types.ObjectId;
  arabic: string;
  transliteration: string;
  banglaMeaning: string;
  englishMeaning: string;
  partOfSpeech?: string;
  audioUrl?: string;
  sortOrder: number;
  createdAt: Date;
  updatedAt: Date;
}

const wordSchema = new Schema<IWord>(
  {
    lessonId: {
      type: Schema.Types.ObjectId,
      ref: "Lesson",
      required: true,
      index: true,
    },
    arabic: { type: String, required: true, trim: true },
    transliteration: { type: String, required: true, trim: true },
    banglaMeaning: { type: String, required: true, trim: true },
    englishMeaning: { type: String, required: true, trim: true },
    partOfSpeech: { type: String },
    audioUrl: { type: String },
    sortOrder: { type: Number, default: 0 },
  },
  { timestamps: true },
);

wordSchema.index({ lessonId: 1, sortOrder: 1 });
// Search-এর জন্য text index (Step 5.6-এ ব্যবহার হবে)
wordSchema.index(
  {
    arabic: "text",
    transliteration: "text",
    banglaMeaning: "text",
    englishMeaning: "text",
  },
  {
    weights: {
      arabic: 5,
      transliteration: 3,
      banglaMeaning: 2,
      englishMeaning: 2,
    },
  },
);

export const WordModel = model<IWord>("Word", wordSchema);
```

`src/models/Sentence.ts`

```ts
import { Schema, model, Types } from "mongoose";

export interface ISentence {
  _id: Types.ObjectId;
  wordId: Types.ObjectId;
  arabic: string;
  transliteration: string;
  banglaMeaning: string;
  englishMeaning: string;
  audioUrl?: string;
  createdAt: Date;
  updatedAt: Date;
}

const sentenceSchema = new Schema<ISentence>(
  {
    wordId: {
      type: Schema.Types.ObjectId,
      ref: "Word",
      required: true,
      index: true,
    },
    arabic: { type: String, required: true, trim: true },
    transliteration: { type: String, required: true, trim: true },
    banglaMeaning: { type: String, required: true, trim: true },
    englishMeaning: { type: String, required: true, trim: true },
    audioUrl: { type: String },
  },
  { timestamps: true },
);

export const SentenceModel = model<ISentence>("Sentence", sentenceSchema);
```

---

## Step 5.2 — Repository Layer

`src/repositories/CategoryRepository.ts`

```ts
import { CategoryModel, ICategory } from "../models/Category";

export const CategoryRepository = {
  findAllActive: () =>
    CategoryModel.find({ isActive: true }).sort({ sortOrder: 1 }).lean(),

  findById: (id: string) => CategoryModel.findById(id).lean(),

  create: (data: Partial<ICategory>) => CategoryModel.create(data),
};
```

`src/repositories/LessonRepository.ts`

```ts
import { LessonModel, ILesson } from "../models/Lesson";

export const LessonRepository = {
  findByCategory: (categoryId: string, skip: number, limit: number) =>
    LessonModel.find({ categoryId, isActive: true })
      .sort({ sortOrder: 1 })
      .skip(skip)
      .limit(limit)
      .lean(),

  countByCategory: (categoryId: string) =>
    LessonModel.countDocuments({ categoryId, isActive: true }),

  findById: (id: string) => LessonModel.findById(id).lean(),

  create: (data: Partial<ILesson>) => LessonModel.create(data),
};
```

`src/repositories/WordRepository.ts`

```ts
import { WordModel, IWord } from "../models/Word";

export const WordRepository = {
  findByLesson: (lessonId: string, skip: number, limit: number) =>
    WordModel.find({ lessonId })
      .sort({ sortOrder: 1 })
      .skip(skip)
      .limit(limit)
      .lean(),

  countByLesson: (lessonId: string) => WordModel.countDocuments({ lessonId }),

  search: (query: string, skip: number, limit: number) =>
    WordModel.find(
      { $text: { $search: query } },
      { score: { $meta: "textScore" } },
    )
      .sort({ score: { $meta: "textScore" } })
      .skip(skip)
      .limit(limit)
      .lean(),

  countSearch: (query: string) =>
    WordModel.countDocuments({ $text: { $search: query } }),

  create: (data: Partial<IWord>) => WordModel.create(data),
};
```

`src/repositories/SentenceRepository.ts`

```ts
import { SentenceModel, ISentence } from "../models/Sentence";

export const SentenceRepository = {
  findByWord: (wordId: string) => SentenceModel.find({ wordId }).lean(),

  create: (data: Partial<ISentence>) => SentenceModel.create(data),
};
```

---

## Step 5.3 — Service Layer (pagination + business logic)

`src/services/categoryService.ts`

```ts
import { CategoryRepository } from "../repositories/CategoryRepository";
import { NotFoundError } from "../utils/AppError";

export const categoryService = {
  async getAll() {
    return CategoryRepository.findAllActive();
  },

  async getById(id: string) {
    const category = await CategoryRepository.findById(id);
    if (!category) throw new NotFoundError("Category");
    return category;
  },
};
```

`src/services/lessonService.ts`

```ts
import { LessonRepository } from "../repositories/LessonRepository";
import { NotFoundError } from "../utils/AppError";

interface PaginationParams {
  page: number;
  limit: number;
}

export const lessonService = {
  async getByCategory(categoryId: string, { page, limit }: PaginationParams) {
    const skip = (page - 1) * limit;
    const [items, total] = await Promise.all([
      LessonRepository.findByCategory(categoryId, skip, limit),
      LessonRepository.countByCategory(categoryId),
    ]);

    return {
      items,
      pagination: { page, limit, total, totalPages: Math.ceil(total / limit) },
    };
  },

  async getById(id: string) {
    const lesson = await LessonRepository.findById(id);
    if (!lesson) throw new NotFoundError("Lesson");
    return lesson;
  },
};
```

`src/services/wordService.ts`

```ts
import { WordRepository } from "../repositories/WordRepository";

interface PaginationParams {
  page: number;
  limit: number;
}

export const wordService = {
  async getByLesson(lessonId: string, { page, limit }: PaginationParams) {
    const skip = (page - 1) * limit;
    const [items, total] = await Promise.all([
      WordRepository.findByLesson(lessonId, skip, limit),
      WordRepository.countByLesson(lessonId),
    ]);
    return {
      items,
      pagination: { page, limit, total, totalPages: Math.ceil(total / limit) },
    };
  },

  async search(query: string, { page, limit }: PaginationParams) {
    const skip = (page - 1) * limit;
    const [items, total] = await Promise.all([
      WordRepository.search(query, skip, limit),
      WordRepository.countSearch(query),
    ]);
    return {
      items,
      pagination: { page, limit, total, totalPages: Math.ceil(total / limit) },
    };
  },
};
```

`src/services/sentenceService.ts`

```ts
import { SentenceRepository } from "../repositories/SentenceRepository";

export const sentenceService = {
  async getByWord(wordId: string) {
    return SentenceRepository.findByWord(wordId);
  },
};
```

---

## Step 5.4 — Validation Schemas (Zod)

`src/validations/contentValidation.ts`

```ts
import { z } from "zod";

export const paginationQuerySchema = z.object({
  query: z.object({
    page: z.coerce.number().int().min(1).default(1),
    limit: z.coerce.number().int().min(1).max(50).default(20),
  }),
  params: z.object({}).optional(),
  body: z.object({}).optional(),
});

export const categoryIdParamSchema = z.object({
  params: z.object({ categoryId: z.string().min(1) }),
  query: z.object({
    page: z.coerce.number().int().min(1).default(1),
    limit: z.coerce.number().int().min(1).max(50).default(20),
  }),
  body: z.object({}).optional(),
});

export const lessonIdParamSchema = z.object({
  params: z.object({ lessonId: z.string().min(1) }),
  query: z.object({
    page: z.coerce.number().int().min(1).default(1),
    limit: z.coerce.number().int().min(1).max(50).default(20),
  }),
  body: z.object({}).optional(),
});

export const wordIdParamSchema = z.object({
  params: z.object({ wordId: z.string().min(1) }),
  query: z.object({}).optional(),
  body: z.object({}).optional(),
});

export const searchQuerySchema = z.object({
  query: z.object({
    q: z.string().min(1, "Search query is required"),
    page: z.coerce.number().int().min(1).default(1),
    limit: z.coerce.number().int().min(1).max(50).default(20),
  }),
  params: z.object({}).optional(),
  body: z.object({}).optional(),
});
```

---

## Step 5.5 — Controllers + Routes

`src/controllers/contentController.ts`

```ts
import { Request, Response } from "express";
import { catchAsync } from "../utils/catchAsync";
import { categoryService } from "../services/categoryService";
import { lessonService } from "../services/lessonService";
import { wordService } from "../services/wordService";
import { sentenceService } from "../services/sentenceService";

export const getCategories = catchAsync(
  async (_req: Request, res: Response) => {
    const categories = await categoryService.getAll();
    res.json({ success: true, data: categories });
  },
);

export const getLessonsByCategory = catchAsync(
  async (req: Request, res: Response) => {
    const { categoryId } = req.params;
    const page = Number(req.query.page) || 1;
    const limit = Number(req.query.limit) || 20;
    const result = await lessonService.getByCategory(categoryId, {
      page,
      limit,
    });
    res.json({ success: true, ...result });
  },
);

export const getWordsByLesson = catchAsync(
  async (req: Request, res: Response) => {
    const { lessonId } = req.params;
    const page = Number(req.query.page) || 1;
    const limit = Number(req.query.limit) || 20;
    const result = await wordService.getByLesson(lessonId, { page, limit });
    res.json({ success: true, ...result });
  },
);

export const getSentencesByWord = catchAsync(
  async (req: Request, res: Response) => {
    const { wordId } = req.params;
    const sentences = await sentenceService.getByWord(wordId);
    res.json({ success: true, data: sentences });
  },
);

export const searchWords = catchAsync(async (req: Request, res: Response) => {
  const q = String(req.query.q);
  const page = Number(req.query.page) || 1;
  const limit = Number(req.query.limit) || 20;
  const result = await wordService.search(q, { page, limit });
  res.json({ success: true, ...result });
});
```

`src/routes/contentRoutes.ts`

```ts
import { Router } from "express";
import { validate } from "../middleware/validate";
import { requireAuth } from "../middleware/requireAuth"; // Step 2-এ বানানো JWT middleware
import {
  categoryIdParamSchema,
  lessonIdParamSchema,
  wordIdParamSchema,
  searchQuerySchema,
} from "../validations/contentValidation";
import {
  getCategories,
  getLessonsByCategory,
  getWordsByLesson,
  getSentencesByWord,
  searchWords,
} from "../controllers/contentController";

const router = Router();

router.use(requireAuth); // সব content route auth-protected

router.get("/categories", getCategories);
router.get(
  "/categories/:categoryId/lessons",
  validate(categoryIdParamSchema),
  getLessonsByCategory,
);
router.get(
  "/lessons/:lessonId/words",
  validate(lessonIdParamSchema),
  getWordsByLesson,
);
router.get(
  "/words/:wordId/sentences",
  validate(wordIdParamSchema),
  getSentencesByWord,
);
router.get("/search", validate(searchQuerySchema), searchWords);

export default router;
```

`src/app.ts`-এ mount করো (Step 3.7-এর `app.ts`-এ যোগ করো):

```ts
import contentRoutes from "./routes/contentRoutes";
// ...
app.use("/api/v1/content", contentRoutes);
```

---

## Step 5.6 — Search Index তৈরি

MongoDB-তে text index compound হওয়ায় একবারই তৈরি করতে হয় (Mongoose model load হওয়ার সময় auto-sync হবে dev-এ, কিন্তু production-এ explicit করাই ভালো):

```bash
# MongoDB shell বা Compass-এ, অথবা একটা ছোট script দিয়ে:
db.words.createIndex(
  { arabic: "text", transliteration: "text", banglaMeaning: "text", englishMeaning: "text" },
  { weights: { arabic: 5, transliteration: 3, banglaMeaning: 2, englishMeaning: 2 } }
)
```

**টেস্ট করো:** `GET /api/v1/content/search?q=hello` কল করে দেখো matching words আসে কিনা।

---

## Step 5.7 — Real Data Seed Script

Lorem Ipsum না — বাস্তব আরবি vocabulary দিয়ে seed করছি। প্যাটার্ন অনুসরণ করে পরে আরও category/lesson/word যোগ করতে পারবে।

`src/seed/seedContent.ts`

```ts
import mongoose from "mongoose";
import { env } from "../config/env";
import { CategoryModel } from "../models/Category";
import { LessonModel } from "../models/Lesson";
import { WordModel } from "../models/Word";
import { SentenceModel } from "../models/Sentence";

async function seed() {
  await mongoose.connect(env.MONGO_URI);
  console.log("Connected. Seeding...");

  await Promise.all([
    CategoryModel.deleteMany({}),
    LessonModel.deleteMany({}),
    WordModel.deleteMany({}),
    SentenceModel.deleteMany({}),
  ]);

  // ---------- Category 1: Greetings ----------
  const greetings = await CategoryModel.create({
    nameAr: "التحيات",
    nameBn: "শুভেচ্ছা বিনিময়",
    nameEn: "Greetings",
    slug: "greetings",
    sortOrder: 1,
  });

  const basicGreetings = await LessonModel.create({
    categoryId: greetings._id,
    titleAr: "التحيات الأساسية",
    titleBn: "মৌলিক শুভেচ্ছা",
    titleEn: "Basic Greetings",
    difficulty: "beginner",
    sortOrder: 1,
  });

  const greetingWords = [
    {
      arabic: "مرحبا",
      transliteration: "marhaban",
      banglaMeaning: "হ্যালো / স্বাগতম",
      englishMeaning: "Hello",
      sentence: {
        arabic: "مرحبا، كيف حالك؟",
        transliteration: "Marhaban, kayfa haluk?",
        banglaMeaning: "হ্যালো, আপনি কেমন আছেন?",
        englishMeaning: "Hello, how are you?",
      },
    },
    {
      arabic: "السلام عليكم",
      transliteration: "as-salamu alaykum",
      banglaMeaning: "আপনার উপর শান্তি বর্ষিত হোক",
      englishMeaning: "Peace be upon you",
      sentence: {
        arabic: "السلام عليكم ورحمة الله",
        transliteration: "As-salamu alaykum wa rahmatullah",
        banglaMeaning: "আপনার উপর শান্তি ও আল্লাহর রহমত বর্ষিত হোক",
        englishMeaning: "Peace and God's mercy be upon you",
      },
    },
    {
      arabic: "صباح الخير",
      transliteration: "sabah al-khayr",
      banglaMeaning: "শুভ সকাল",
      englishMeaning: "Good morning",
      sentence: {
        arabic: "صباح الخير يا صديقي",
        transliteration: "Sabah al-khayr ya sadiqi",
        banglaMeaning: "শুভ সকাল, আমার বন্ধু",
        englishMeaning: "Good morning, my friend",
      },
    },
    {
      arabic: "شكرا",
      transliteration: "shukran",
      banglaMeaning: "ধন্যবাদ",
      englishMeaning: "Thank you",
      sentence: {
        arabic: "شكرا جزيلا لمساعدتك",
        transliteration: "Shukran jazilan li-musa'adatik",
        banglaMeaning: "আপনার সাহায্যের জন্য অনেক ধন্যবাদ",
        englishMeaning: "Thank you very much for your help",
      },
    },
    {
      arabic: "مع السلامة",
      transliteration: "ma'a as-salama",
      banglaMeaning: "বিদায় (নিরাপদে থাকুন)",
      englishMeaning: "Goodbye",
    },
  ];

  for (const [index, w] of greetingWords.entries()) {
    const word = await WordModel.create({
      lessonId: basicGreetings._id,
      arabic: w.arabic,
      transliteration: w.transliteration,
      banglaMeaning: w.banglaMeaning,
      englishMeaning: w.englishMeaning,
      sortOrder: index + 1,
    });

    if (w.sentence) {
      await SentenceModel.create({ wordId: word._id, ...w.sentence });
    }
  }

  // ---------- Category 2: Numbers ----------
  const numbers = await CategoryModel.create({
    nameAr: "الأرقام",
    nameBn: "সংখ্যা",
    nameEn: "Numbers",
    slug: "numbers",
    sortOrder: 2,
  });

  const numbersLesson = await LessonModel.create({
    categoryId: numbers._id,
    titleAr: "الأرقام من ١ إلى ٥",
    titleBn: "১ থেকে ৫ পর্যন্ত সংখ্যা",
    titleEn: "Numbers 1 to 5",
    difficulty: "beginner",
    sortOrder: 1,
  });

  const numberWords = [
    {
      arabic: "واحد",
      transliteration: "wahid",
      banglaMeaning: "এক",
      englishMeaning: "One",
    },
    {
      arabic: "اثنان",
      transliteration: "ithnan",
      banglaMeaning: "দুই",
      englishMeaning: "Two",
    },
    {
      arabic: "ثلاثة",
      transliteration: "thalatha",
      banglaMeaning: "তিন",
      englishMeaning: "Three",
    },
    {
      arabic: "أربعة",
      transliteration: "arba'a",
      banglaMeaning: "চার",
      englishMeaning: "Four",
    },
    {
      arabic: "خمسة",
      transliteration: "khamsa",
      banglaMeaning: "পাঁচ",
      englishMeaning: "Five",
    },
  ];

  for (const [index, w] of numberWords.entries()) {
    await WordModel.create({
      lessonId: numbersLesson._id,
      ...w,
      sortOrder: index + 1,
    });
  }

  // ---------- Category 3: Family ----------
  const family = await CategoryModel.create({
    nameAr: "العائلة",
    nameBn: "পরিবার",
    nameEn: "Family",
    slug: "family",
    sortOrder: 3,
  });

  const familyLesson = await LessonModel.create({
    categoryId: family._id,
    titleAr: "أفراد العائلة",
    titleBn: "পরিবারের সদস্যরা",
    titleEn: "Family Members",
    difficulty: "beginner",
    sortOrder: 1,
  });

  const familyWords = [
    {
      arabic: "أب",
      transliteration: "ab",
      banglaMeaning: "বাবা",
      englishMeaning: "Father",
    },
    {
      arabic: "أم",
      transliteration: "umm",
      banglaMeaning: "মা",
      englishMeaning: "Mother",
    },
    {
      arabic: "أخ",
      transliteration: "akh",
      banglaMeaning: "ভাই",
      englishMeaning: "Brother",
    },
    {
      arabic: "أخت",
      transliteration: "ukht",
      banglaMeaning: "বোন",
      englishMeaning: "Sister",
    },
  ];

  for (const [index, w] of familyWords.entries()) {
    await WordModel.create({
      lessonId: familyLesson._id,
      ...w,
      sortOrder: index + 1,
    });
  }

  console.log("✅ Seed complete");
  await mongoose.disconnect();
}

seed().catch((err) => {
  console.error("Seed failed", err);
  process.exit(1);
});
```

`package.json`-এ script যোগ করো:

```json
"scripts": {
  "seed": "ts-node src/seed/seedContent.ts"
}
```

```bash
npm run seed
```

**টেস্ট করো:** MongoDB Compass-এ গিয়ে `categories`, `lessons`, `words`, `sentences` collection-এ ডাটা আছে কিনা দেখো। তারপর `GET /api/v1/content/categories` কল করো — ৩টা category (Greetings, Numbers, Family) আসা উচিত।

---

# MOBILE

## Step 5.8 — API Layer (Axios)

`src/api/contentApi.ts`

```ts
import { apiClient } from "./apiClient"; // Step 2-এ বানানো axios instance

export interface CategoryDto {
  _id: string;
  nameAr: string;
  nameBn: string;
  nameEn: string;
  slug: string;
  icon?: string;
  sortOrder: number;
}

export interface LessonDto {
  _id: string;
  categoryId: string;
  titleAr: string;
  titleBn: string;
  titleEn: string;
  difficulty: string;
  sortOrder: number;
}

export const contentApi = {
  getCategories: async (): Promise<CategoryDto[]> => {
    const { data } = await apiClient.get("/content/categories");
    return data.data;
  },

  getLessonsByCategory: async (categoryId: string): Promise<LessonDto[]> => {
    const { data } = await apiClient.get(
      `/content/categories/${categoryId}/lessons`,
    );
    return data.items;
  },
};
```

---

## Step 5.9 — Mobile SQLite Repositories (Category, Word, Sentence)

Step 4.4-এ `LessonRepository` বানানো হয়েছিল একই প্যাটার্নে বাকিগুলো:

`src/db/repositories/CategoryRepository.ts`

```ts
import { BaseRepository } from "./BaseRepository";

export interface CategoryRow {
  id: string;
  name_ar: string;
  name_bn: string;
  name_en: string;
  icon: string | null;
  sort_order: number;
  updated_at: string;
}

class CategoryRepositoryClass extends BaseRepository<CategoryRow> {
  protected tableName = "categories";

  async upsertMany(categories: CategoryRow[]): Promise<void> {
    const db = await this.db();
    await db.withTransactionAsync(async () => {
      for (const c of categories) {
        await db.runAsync(
          `INSERT INTO categories (id, name_ar, name_bn, name_en, icon, sort_order, updated_at)
           VALUES (?, ?, ?, ?, ?, ?, ?)
           ON CONFLICT(id) DO UPDATE SET
             name_ar=excluded.name_ar, name_bn=excluded.name_bn, name_en=excluded.name_en,
             icon=excluded.icon, sort_order=excluded.sort_order, updated_at=excluded.updated_at`,
          [
            c.id,
            c.name_ar,
            c.name_bn,
            c.name_en,
            c.icon,
            c.sort_order,
            c.updated_at,
          ],
        );
      }
    });
  }
}

export const CategoryRepository = new CategoryRepositoryClass();
```

(`WordRepository`, `SentenceRepository` ঠিক একই প্যাটার্নে — `LessonRepository`-এর কোড কপি করে টেবিল/কলাম নাম বদলালেই হবে। জায়গা বাঁচাতে এখানে repeat করছি না।)

---

## Step 5.10 — Offline-First Sync + Read Hooks

`src/features/categories/categorySync.ts`

```ts
import { contentApi } from "../../api/contentApi";
import { CategoryRepository } from "../../db/repositories/CategoryRepository";

export async function syncCategories(): Promise<void> {
  const remote = await contentApi.getCategories();

  const rows = remote.map((c) => ({
    id: c._id,
    name_ar: c.nameAr,
    name_bn: c.nameBn,
    name_en: c.nameEn,
    icon: c.icon ?? null,
    sort_order: c.sortOrder,
    updated_at: new Date().toISOString(),
  }));

  await CategoryRepository.upsertMany(rows);
}
```

`src/features/categories/useCategories.ts`

```ts
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { useEffect } from "react";
import { CategoryRepository } from "../../db/repositories/CategoryRepository";
import { syncCategories } from "./categorySync";
import { useNetworkStore } from "../../store/networkStore";

// Local read — সবসময় SQLite থেকে, network অবস্থা নির্বিশেষে
export function useCategories() {
  const queryClient = useQueryClient();
  const isOnline = useNetworkStore((s) => s.isOnline);

  const query = useQuery({
    queryKey: ["categories", "local"],
    queryFn: () => CategoryRepository.findAll(),
  });

  useEffect(() => {
    if (!isOnline) return;

    syncCategories()
      .then(() =>
        queryClient.invalidateQueries({ queryKey: ["categories", "local"] }),
      )
      .catch((err) => console.warn("Category sync failed:", err));
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [isOnline]);

  return query;
}
```

**কেন এভাবে:** UI কখনো network-এর অপেক্ষায় থাকে না — SQLite থেকে সাথে সাথে ডাটা দেখায় (offline-এও কাজ করে)। Online থাকলে background-এ sync হয়ে local cache আপডেট হয়, আর query invalidate করলে UI reactive-ভাবে নতুন ডাটা দেখায়।

---

## Step 5.11 — Categories Screen (Vertical Slice UI)

`app/(app)/categories/index.tsx`

```tsx
import { View, Text, FlatList, RefreshControl, Pressable } from "react-native";
import { useState } from "react";
import { useCategories } from "../../../src/features/categories/useCategories";
import { syncCategories } from "../../../src/features/categories/categorySync";
import { useQueryClient } from "@tanstack/react-query";
import { router } from "expo-router";

export default function CategoriesScreen() {
  const { data: categories, isLoading, isError } = useCategories();
  const queryClient = useQueryClient();
  const [refreshing, setRefreshing] = useState(false);

  const onRefresh = async () => {
    setRefreshing(true);
    try {
      await syncCategories();
      await queryClient.invalidateQueries({
        queryKey: ["categories", "local"],
      });
    } catch {
      // offline হলে silently fail — local data-ই দেখানো হবে
    } finally {
      setRefreshing(false);
    }
  };

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
        <Text className="text-red-500 text-center">
          কিছু একটা সমস্যা হয়েছে। আবার চেষ্টা করো।
        </Text>
      </View>
    );
  }

  if (!categories || categories.length === 0) {
    return (
      <View className="flex-1 items-center justify-center bg-white dark:bg-gray-900 px-6">
        <Text className="text-gray-400 text-center">
          এখনো কোনো ক্যাটাগরি যোগ হয়নি।
        </Text>
      </View>
    );
  }

  return (
    <FlatList
      className="flex-1 bg-white dark:bg-gray-900"
      data={categories}
      keyExtractor={(item) => item.id}
      refreshControl={
        <RefreshControl refreshing={refreshing} onRefresh={onRefresh} />
      }
      contentContainerStyle={{ padding: 16 }}
      renderItem={({ item }) => (
        <Pressable
          onPress={() => router.push(`/categories/${item.id}/lessons`)}
          className="bg-gray-100 dark:bg-gray-800 rounded-xl p-4 mb-3"
        >
          <Text className="text-lg font-semibold text-gray-900 dark:text-white">
            {item.name_bn}
          </Text>
          <Text className="text-gray-500 dark:text-gray-400">
            {item.name_en}
          </Text>
          <Text className="text-2xl mt-1 text-right">{item.name_ar}</Text>
        </Pressable>
      )}
    />
  );
}
```

**টেস্ট করো:**

1. Backend seed data নিয়ে চলছে এমন অবস্থায় app খোলো — ৩টা category দেখা উচিত (Greetings, Numbers, Family)
2. Airplane mode অন করে app restart করো — SQLite থেকে ক্যাটাগরি এখনো দেখা উচিত (offline কাজ করছে প্রমাণ)
3. Pull to refresh করো — sync ঠিকভাবে চলছে কিনা console-এ দেখো

---

## এই Step-এ যা হলো (সারসংক্ষেপ)

- MongoDB-তে 4টা schema (Category, Lesson, Word, Sentence) — সঠিক relationship আর index সহ
- Repository → Service → Controller → Routes — backend layered architecture বজায় রাখা হয়েছে
- Text-index ভিত্তিক search endpoint
- Lorem Ipsum না — বাস্তব আরবি vocabulary (Greetings, Numbers, Family) দিয়ে seed
- Mobile-এ offline-first read/sync pattern প্রতিষ্ঠিত হলো: **SQLite = read source, API = background sync source** — এই একই প্যাটার্ন Lessons, Words, Sentences স্ক্রিনেও পুনরায় ব্যবহার হবে

**ব্রুটাল ট্রুথ:** Lessons/Words/Sentences স্ক্রিন এখানে দিইনি — Categories vertical slice-টা প্যাটার্ন প্রতিষ্ঠার জন্য যথেষ্ট। বাকিগুলো mechanical repetition (একই repository + sync + screen প্যাটার্ন), সেটা চাইলে পরের ছোট ধাপ হিসেবে করে দিতে পারি, নাকি সরাসরি Step 6 (Audio System)-এ যেতে চাও?

**পরবর্তী যৌক্তিক ধাপ:** হয় (ক) Lessons + Words + Sentences স্ক্রিন একই প্যাটার্নে শেষ করা, অথবা (খ) Step 6 — Audio System, যেহেতু Word schema-তে ইতিমধ্যে `audioUrl` ফিল্ড আছে।
