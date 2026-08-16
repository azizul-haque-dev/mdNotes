# TopicConversation Module — CRUD

Same folder-per-domain pattern as `topic`/`category`. Model already exists in `schema.prisma`:

```prisma
model TopicConversation {
  id       String @id @default(cuid())
  topicId  String @map("topic_id")
  topic    Topic  @relation(fields: [topicId], references: [id], onDelete: Cascade)
  titleEn  String @map("title_en")
  titleBn  String? @map("title_bn")
  conversations Conversation[]
  createdAt DateTime @default(now()) @map("created_at") @db.Timestamptz(6)
  updatedAt DateTime @updatedAt @map("updated_at") @db.Timestamptz(6)
  @@index([topicId])
  @@map("topic_conversations")
}
```

Note: no `@@unique` on `titleEn` here (unlike `Topic`), since the same title could reasonably repeat
under different topics — so no app-level uniqueness check either, just a topic-existence check on create.

New folder: `server/src/modules/topic-conversation/` (kebab-case since it's a two-word domain;
file prefix stays `topic-conversation.*` to match).

---

## `topic-conversation.validation.ts`

```ts
import { PAGINATION, VALIDATION } from "@/shared/constants.js";
import { z } from "zod";

export const createTopicConversationSchema = z.object({
  topicId: z.string().min(1),
  titleEn: z
    .string()
    .trim()
    .min(VALIDATION.CATEGORY_NAME.MIN_LENGTH)
    .max(VALIDATION.CATEGORY_NAME.MAX_LENGTH),
  titleBn: z.string().trim().max(VALIDATION.CATEGORY_NAME.MAX_LENGTH).optional(),
});

export const updateTopicConversationSchema = createTopicConversationSchema.partial();

export const topicConversationIdParamSchema = z.object({
  id: z.string().min(1),
});

export const listTopicConversationsQuerySchema = z.object({
  page: z.coerce.number().int().positive().default(PAGINATION.DEFAULT_PAGE),
  limit: z.coerce.number().int().positive().max(PAGINATION.MAX_LIMIT).default(PAGINATION.DEFAULT_LIMIT),
  topicId: z.string().optional(),
  search: z.string().trim().optional(),
});

export type CreateTopicConversationInput = z.infer<typeof createTopicConversationSchema>;
export type UpdateTopicConversationInput = z.infer<typeof updateTopicConversationSchema>;
export type ListTopicConversationsQuery = z.infer<typeof listTopicConversationsQuerySchema>;
```

---

## `topic-conversation.repository.ts`

```ts
// Database access layer for topic-conversation operations.
import { Prisma } from "@/generated/prisma/client.js";
import { prisma } from "../../config/database.js";
import { UpdateTopicConversationInput } from "./topic-conversation.validation.js";

export const TOPIC_CONVERSATION_INCLUDE = {
  topic: true,
  _count: { select: { conversations: true } },
} satisfies Prisma.TopicConversationInclude;

// Flattens the _count relation into a plain conversationCount field.
export function presentTopicConversation(
  topicConversation: Prisma.TopicConversationGetPayload<{
    include: typeof TOPIC_CONVERSATION_INCLUDE;
  }>,
) {
  const { _count, ...rest } = topicConversation;
  return { ...rest, conversationCount: _count.conversations };
}

export const TopicConversationRepository = {
  findMany: (
    where: Prisma.TopicConversationWhereInput,
    skip: number,
    take: number,
  ) =>
    prisma.topicConversation.findMany({
      where,
      include: TOPIC_CONVERSATION_INCLUDE,
      orderBy: { createdAt: "desc" },
      skip,
      take,
    }),

  count: (where: Prisma.TopicConversationWhereInput) =>
    prisma.topicConversation.count({ where }),

  findById: (id: string) =>
    prisma.topicConversation.findUnique({
      where: { id },
      include: TOPIC_CONVERSATION_INCLUDE,
    }),

  create: (data: { topicId: string; titleEn: string; titleBn?: string }) =>
    prisma.topicConversation.create({
      data,
      include: TOPIC_CONVERSATION_INCLUDE,
    }),

  update: (id: string, data: UpdateTopicConversationInput) =>
    prisma.topicConversation.update({
      where: { id },
      data,
      include: TOPIC_CONVERSATION_INCLUDE,
    }),

  delete: (id: string) => prisma.topicConversation.delete({ where: { id } }),
};
```

---

## `topic-conversation.service.ts`

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
import { TopicRepository } from "@/modules/topic/topic.repository.js";
import {
  CreateTopicConversationInput,
  ListTopicConversationsQuery,
  UpdateTopicConversationInput,
} from "./topic-conversation.validation.js";
import {
  TopicConversationRepository,
  presentTopicConversation,
} from "./topic-conversation.repository.js";

type TopicConversationListResult = {
  items: ReturnType<typeof presentTopicConversation>[];
  meta: { page: number; limit: number; total: number; totalPages: number };
};

export async function list(query: ListTopicConversationsQuery) {
  const key = cacheKey(cacheNamespaces.topicConversations, query);
  const cached = await cacheGet<TopicConversationListResult>(key);
  if (cached) return cached;

  const { page, limit, topicId, search } = query;

  const where = {
    ...(topicId ? { topicId } : {}),
    ...(search
      ? {
          OR: [
            { titleEn: { contains: search, mode: "insensitive" as const } },
            { titleBn: { contains: search, mode: "insensitive" as const } },
          ],
        }
      : {}),
  };

  const [items, total] = await Promise.all([
    TopicConversationRepository.findMany(where, (page - 1) * limit, limit),
    TopicConversationRepository.count(where),
  ]);

  const result: TopicConversationListResult = {
    items: items.map(presentTopicConversation),
    meta: {
      page,
      limit,
      total,
      totalPages: Math.max(1, Math.ceil(total / limit)),
    },
  };
  await cacheSet(key, result, CACHE_TTL.TOPIC_CONVERSATIONS);
  return result;
}

export async function getById(id: string) {
  const topicConversation = await TopicConversationRepository.findById(id);
  if (!topicConversation) throw ApiError.notFound("Topic conversation not found");
  return presentTopicConversation(topicConversation);
}

export async function create(input: CreateTopicConversationInput) {
  const topic = await TopicRepository.findById(input.topicId);
  if (!topic) throw ApiError.badRequest("Topic does not exist");

  const topicConversation = await TopicConversationRepository.create(input);
  await invalidateCacheNamespace(cacheNamespaces.topicConversations);
  return presentTopicConversation(topicConversation);
}

export async function update(id: string, input: UpdateTopicConversationInput) {
  await getById(id); // 404s early if it doesn't exist

  if (input.topicId) {
    const topic = await TopicRepository.findById(input.topicId);
    if (!topic) throw ApiError.badRequest("Topic does not exist");
  }

  const topicConversation = await TopicConversationRepository.update(id, input);
  await invalidateCacheNamespace(cacheNamespaces.topicConversations);
  return presentTopicConversation(topicConversation);
}

export async function remove(id: string): Promise<void> {
  await getById(id);

  // Cascades to Conversation -> ConversationLine.
  await TopicConversationRepository.delete(id);
  await invalidateCacheNamespace(cacheNamespaces.topicConversations);
}
```

---

## `topic-conversation.controller.ts`

```ts
import { Request, Response } from "express";
import { sendSuccess } from "@/lib/api-response.js";
import { asyncHandler } from "@/lib/async-handler.js";
import * as topicConversationService from "./topic-conversation.service.js";
import { ListTopicConversationsQuery } from "./topic-conversation.validation.js";

export const list = asyncHandler(async (req: Request, res: Response) => {
  const query = (req as Request & { validatedQuery: ListTopicConversationsQuery })
    .validatedQuery;
  const { items, meta } = await topicConversationService.list(query);
  sendSuccess(res, 200, "Topic conversations fetched", { items, meta });
});

export const getOne = asyncHandler(async (req: Request, res: Response) => {
  const topicConversation = await topicConversationService.getById(
    req.params.id as string,
  );
  sendSuccess(res, 200, "Topic conversation fetched", topicConversation);
});

export const create = asyncHandler(async (req: Request, res: Response) => {
  const topicConversation = await topicConversationService.create(req.body);
  sendSuccess(res, 201, "Topic conversation created", topicConversation);
});

export const update = asyncHandler(async (req: Request, res: Response) => {
  const topicConversation = await topicConversationService.update(
    req.params.id as string,
    req.body,
  );
  sendSuccess(res, 200, "Topic conversation updated", topicConversation);
});

export const remove = asyncHandler(async (req: Request, res: Response) => {
  await topicConversationService.remove(req.params.id as string);
  sendSuccess(res, 200, "Topic conversation deleted");
});
```

---

## `topic-conversation.routes.ts`

```ts
import { Router } from "express";
import { requireAuth } from "../../middlewares/auth.middleware.js";
import { validate } from "../../middlewares/validate.middleware.js";
import * as controller from "./topic-conversation.controller.js";
import {
  createTopicConversationSchema,
  listTopicConversationsQuerySchema,
  topicConversationIdParamSchema,
  updateTopicConversationSchema,
} from "./topic-conversation.validation.js";

const router = Router();

// Reading is public - needed to browse conversations under a topic.
router.get(
  "/",
  validate({ query: listTopicConversationsQuerySchema }),
  controller.list,
);
router.get(
  "/:id",
  validate({ params: topicConversationIdParamSchema }),
  controller.getOne,
);

// Writes require an authenticated user.
router.post(
  "/",
  requireAuth,
  validate({ body: createTopicConversationSchema }),
  controller.create,
);
router.patch(
  "/:id",
  requireAuth,
  validate({
    params: topicConversationIdParamSchema,
    body: updateTopicConversationSchema,
  }),
  controller.update,
);
router.delete(
  "/:id",
  requireAuth,
  validate({ params: topicConversationIdParamSchema }),
  controller.remove,
);

export default router;
```

---

## `topic-conversation.types.ts`

```ts
/**
 * TopicConversation Module Types
 * Request/Response DTOs and related type definitions
 */

import { z } from "zod";
import {
  createTopicConversationSchema,
  updateTopicConversationSchema,
  listTopicConversationsQuerySchema,
} from "./topic-conversation.validation.js";

export type CreateTopicConversationInput = z.infer<typeof createTopicConversationSchema>;
export type UpdateTopicConversationInput = z.infer<typeof updateTopicConversationSchema>;
export type ListTopicConversationsQuery = z.infer<typeof listTopicConversationsQuerySchema>;

export interface TopicConversationResponse {
  id: string;
  topicId: string;
  topic: {
    id: string;
    titleEn: string;
    titleBn?: string | null;
  };
  titleEn: string;
  titleBn?: string | null;
  conversationCount: number;
  createdAt?: Date;
  updatedAt?: Date;
}

export interface ListTopicConversationsResponse {
  items: TopicConversationResponse[];
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
  TOPICS: 1800,
  TOPIC_CONVERSATIONS: 1800, // ADD THIS
  USER_PROFILE: 300,
} as const;
```

**2. `server/src/integrations/cache.ts`** — register the namespace:

```ts
export const cacheNamespaces = {
  categories: `${CACHE_PREFIX}:categories`,
  words: `${CACHE_PREFIX}:words`,
  sentences: `${CACHE_PREFIX}:sentences`,
  topics: `${CACHE_PREFIX}:topics`,
  topicConversations: `${CACHE_PREFIX}:topic-conversations`, // ADD THIS
} as const;
```

**3. `server/src/routes/index.ts`** — mount the router:

```ts
import topicConversationRoutes from "../modules/topic-conversation/topic-conversation.routes.js";
// ...
router.use("/topic-conversations", topicConversationRoutes);
```

---

## Endpoints

| Method | Path                          | Auth | Description                              |
|--------|-------------------------------|------|-------------------------------------------|
| GET    | `/api/v1/topic-conversations`     | No   | List (paginated, filter by `topicId`, search) |
| GET    | `/api/v1/topic-conversations/:id` | No   | Get one (includes parent `topic`)        |
| POST   | `/api/v1/topic-conversations`     | Yes  | Create (validates `topicId` exists)      |
| PATCH  | `/api/v1/topic-conversations/:id` | Yes  | Update (re-validates `topicId` if changed) |
| DELETE | `/api/v1/topic-conversations/:id` | Yes  | Delete (cascades to conversations + lines) |

Note: creating/updating with a bad `topicId` returns a clean `400 "Topic does not exist"` from the
service layer rather than relying on Prisma's raw `P2003` foreign-key error, which the global
error handler doesn't special-case.