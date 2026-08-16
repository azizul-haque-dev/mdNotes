# Topic Module — CRUD

Follows the same folder-per-domain pattern as `category` (routes/controller/service/validation/repository).
Model already exists in `schema.prisma`:

```prisma
model Topic {
  id       String @id @default(cuid())
  titleEn  String @map("title_en")
  titleBn  String? @map("title_bn")
  conversations TopicConversation[]
  createdAt DateTime @default(now()) @map("created_at") @db.Timestamptz(6)
  updatedAt DateTime @updatedAt @map("updated_at") @db.Timestamptz(6)
  @@unique([titleEn])
  @@map("topics")
}
```

New folder: `server/src/modules/topic/`

---

## `topic.validation.ts`

```ts
import { PAGINATION, VALIDATION } from "@/shared/constants.js";
import { z } from "zod";

export const createTopicSchema = z.object({
  titleEn: z
    .string()
    .trim()
    .min(VALIDATION.CATEGORY_NAME.MIN_LENGTH)
    .max(VALIDATION.CATEGORY_NAME.MAX_LENGTH),
  titleBn: z.string().trim().max(VALIDATION.CATEGORY_NAME.MAX_LENGTH).optional(),
});

export const updateTopicSchema = createTopicSchema.partial();

export const topicIdParamSchema = z.object({
  id: z.string().min(1),
});

export const listTopicsQuerySchema = z.object({
  page: z.coerce.number().int().positive().default(PAGINATION.DEFAULT_PAGE),
  limit: z.coerce.number().int().positive().max(PAGINATION.MAX_LIMIT).default(PAGINATION.DEFAULT_LIMIT),
  search: z.string().trim().optional(),
});

export type CreateTopicInput = z.infer<typeof createTopicSchema>;
export type UpdateTopicInput = z.infer<typeof updateTopicSchema>;
export type ListTopicsQuery = z.infer<typeof listTopicsQuerySchema>;
```

---

## `topic.repository.ts`

```ts
// Database access layer for topic operations.
import { Prisma } from "@/generated/prisma/client.js";
import { prisma } from "../../config/database.js";
import { UpdateTopicInput } from "./topic.validation.js";

export const TOPIC_INCLUDE = {
  _count: { select: { conversations: true } },
} satisfies Prisma.TopicInclude;

// Flattens the _count relation into a plain conversationCount field.
export function presentTopic(
  topic: Prisma.TopicGetPayload<{ include: typeof TOPIC_INCLUDE }>,
) {
  const { _count, ...rest } = topic;
  return { ...rest, conversationCount: _count.conversations };
}

export const TopicRepository = {
  findMany: (where: Prisma.TopicWhereInput, skip: number, take: number) =>
    prisma.topic.findMany({
      where,
      include: TOPIC_INCLUDE,
      orderBy: { createdAt: "desc" },
      skip,
      take,
    }),

  count: (where: Prisma.TopicWhereInput) => prisma.topic.count({ where }),

  findById: (id: string) =>
    prisma.topic.findUnique({ where: { id }, include: TOPIC_INCLUDE }),

  findByTitleEn: (titleEn: string) =>
    prisma.topic.findUnique({ where: { titleEn } }),

  create: (data: { titleEn: string; titleBn?: string }) =>
    prisma.topic.create({ data, include: TOPIC_INCLUDE }),

  update: (id: string, data: UpdateTopicInput) =>
    prisma.topic.update({ where: { id }, data, include: TOPIC_INCLUDE }),

  delete: (id: string) => prisma.topic.delete({ where: { id } }),
};
```

---

## `topic.service.ts`

```ts
import { ApiError } from "@/lib/api-error.js";
import { CACHE_TTL } from "@/shared/constants.js";
import {
  cacheGet,
  cacheKey,
  cacheNamespaces,
  cacheSet,
  invalidateCacheNamespace,
} from "@/integrations/cache.js";
import {
  CreateTopicInput,
  ListTopicsQuery,
  UpdateTopicInput,
} from "./topic.validation.js";
import { TopicRepository, presentTopic } from "./topic.repository.js";

type TopicListResult = {
  items: ReturnType<typeof presentTopic>[];
  meta: { page: number; limit: number; total: number; totalPages: number };
};

export async function list(query: ListTopicsQuery) {
  const key = cacheKey(cacheNamespaces.topics, query);
  const cached = await cacheGet<TopicListResult>(key);
  if (cached) return cached;

  const { page, limit, search } = query;

  const where = search
    ? {
        OR: [
          { titleEn: { contains: search, mode: "insensitive" as const } },
          { titleBn: { contains: search, mode: "insensitive" as const } },
        ],
      }
    : {};

  const [items, total] = await Promise.all([
    TopicRepository.findMany(where, (page - 1) * limit, limit),
    TopicRepository.count(where),
  ]);

  const result: TopicListResult = {
    items: items.map(presentTopic),
    meta: {
      page,
      limit,
      total,
      totalPages: Math.max(1, Math.ceil(total / limit)),
    },
  };
  await cacheSet(key, result, CACHE_TTL.TOPICS);
  return result;
}

export async function getById(id: string) {
  const topic = await TopicRepository.findById(id);
  if (!topic) throw ApiError.notFound("Topic not found");
  return presentTopic(topic);
}

export async function create(input: CreateTopicInput) {
  const existing = await TopicRepository.findByTitleEn(input.titleEn);
  if (existing) throw ApiError.conflict("A topic with this title already exists");

  const topic = await TopicRepository.create(input);
  await invalidateCacheNamespace(cacheNamespaces.topics);
  return presentTopic(topic);
}

export async function update(id: string, input: UpdateTopicInput) {
  await getById(id); // 404s early if it doesn't exist

  const topic = await TopicRepository.update(id, input);
  await invalidateCacheNamespace(cacheNamespaces.topics);
  return presentTopic(topic);
}

export async function remove(id: string): Promise<void> {
  await getById(id);

  // Cascades to TopicConversation -> Conversation -> ConversationLine.
  await TopicRepository.delete(id);
  await invalidateCacheNamespace(cacheNamespaces.topics);
}
```

---

## `topic.controller.ts`

```ts
import { Request, Response } from "express";
import { sendSuccess } from "@/lib/api-response.js";
import { asyncHandler } from "@/lib/async-handler.js";
import * as topicService from "./topic.service.js";
import { ListTopicsQuery } from "./topic.validation.js";

export const list = asyncHandler(async (req: Request, res: Response) => {
  const query = (req as Request & { validatedQuery: ListTopicsQuery })
    .validatedQuery;
  const { items, meta } = await topicService.list(query);
  sendSuccess(res, 200, "Topics fetched", { items, meta });
});

export const getOne = asyncHandler(async (req: Request, res: Response) => {
  const topic = await topicService.getById(req.params.id as string);
  sendSuccess(res, 200, "Topic fetched", topic);
});

export const create = asyncHandler(async (req: Request, res: Response) => {
  const topic = await topicService.create(req.body);
  sendSuccess(res, 201, "Topic created", topic);
});

export const update = asyncHandler(async (req: Request, res: Response) => {
  const topic = await topicService.update(req.params.id as string, req.body);
  sendSuccess(res, 200, "Topic updated", topic);
});

export const remove = asyncHandler(async (req: Request, res: Response) => {
  await topicService.remove(req.params.id as string);
  sendSuccess(res, 200, "Topic deleted");
});
```

---

## `topic.routes.ts`

```ts
import { Router } from "express";
import { requireAuth } from "../../middlewares/auth.middleware.js";
import { validate } from "../../middlewares/validate.middleware.js";
import * as controller from "./topic.controller.js";
import {
  createTopicSchema,
  listTopicsQuerySchema,
  topicIdParamSchema,
  updateTopicSchema,
} from "./topic.validation.js";

const router = Router();

// Reading topics is public - needed to browse/render conversation topics.
router.get("/", validate({ query: listTopicsQuerySchema }), controller.list);
router.get("/:id", validate({ params: topicIdParamSchema }), controller.getOne);

// Writes require an authenticated user.
router.post(
  "/",
  requireAuth,
  validate({ body: createTopicSchema }),
  controller.create,
);
router.patch(
  "/:id",
  requireAuth,
  validate({ params: topicIdParamSchema, body: updateTopicSchema }),
  controller.update,
);
router.delete(
  "/:id",
  requireAuth,
  validate({ params: topicIdParamSchema }),
  controller.remove,
);

export default router;
```

---

## `topic.types.ts`

```ts
/**
 * Topic Module Types
 * Request/Response DTOs and related type definitions
 */

import { z } from "zod";
import {
  createTopicSchema,
  updateTopicSchema,
  listTopicsQuerySchema,
} from "./topic.validation.js";

export type CreateTopicInput = z.infer<typeof createTopicSchema>;
export type UpdateTopicInput = z.infer<typeof updateTopicSchema>;
export type ListTopicsQuery = z.infer<typeof listTopicsQuerySchema>;

export interface TopicResponse {
  id: string;
  titleEn: string;
  titleBn?: string | null;
  conversationCount: number;
  createdAt?: Date;
  updatedAt?: Date;
}

export interface ListTopicsResponse {
  items: TopicResponse[];
  meta: { page: number; limit: number; total: number; totalPages: number };
}
```

---

## Wiring it up (small edits to existing files)

**1. `server/src/shared/constants.ts`** — add a TTL entry:

```ts
export const CACHE_TTL = {
  CATEGORIES: 3600,
  WORDS: 1800,
  SENTENCES: 1800,
  TOPICS: 1800, // ADD THIS
  USER_PROFILE: 300,
} as const;
```

**2. `server/src/integrations/cache.ts`** — register the namespace:

```ts
export const cacheNamespaces = {
  categories: `${CACHE_PREFIX}:categories`,
  words: `${CACHE_PREFIX}:words`,
  sentences: `${CACHE_PREFIX}:sentences`,
  topics: `${CACHE_PREFIX}:topics`, // ADD THIS
} as const;
```

**3. `server/src/routes/index.ts`** — mount the router:

```ts
import topicRoutes from "../modules/topic/topic.routes.js";
// ...
router.use("/topics", topicRoutes);
```

---

## Endpoints

| Method | Path              | Auth | Description          |
|--------|--------------------|------|-----------------------|
| GET    | `/api/v1/topics`      | No   | List (paginated, search) |
| GET    | `/api/v1/topics/:id`  | No   | Get one              |
| POST   | `/api/v1/topics`      | Yes  | Create               |
| PATCH  | `/api/v1/topics/:id`  | Yes  | Update               |
| DELETE | `/api/v1/topics/:id`  | Yes  | Delete (cascades to conversations) |

Note: `P2002` (unique `titleEn` violation) and `P2025` (not found) are already handled generically
by `error-handler.middleware.ts`, but the service layer checks `findByTitleEn` first to return a
clean 409 with a readable message before hitting the DB constraint.