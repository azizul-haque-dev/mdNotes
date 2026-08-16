# Conversation Module — CRUD

Same folder-per-domain pattern as `sentence` (which also manages a parent + an ordered child list —
here it's `Conversation` + `ConversationLine`, mirroring how `Sentence` manages `SentenceWord`).
Models already exist in `schema.prisma`:

```prisma
model Conversation {
  id String @id @default(cuid())
  topicConversationId String @map("topic_conversation_id")
  topicConversation TopicConversation @relation(fields: [topicConversationId], references: [id], onDelete: Cascade)
  lines ConversationLine[]
  createdAt DateTime @default(now()) @map("created_at") @db.Timestamptz(6)
  updatedAt DateTime @updatedAt @map("updated_at") @db.Timestamptz(6)
  @@index([topicConversationId])
  @@map("conversations")
}

model ConversationLine {
  id String @id @default(cuid())
  conversationId String @map("conversation_id")
  conversation Conversation @relation(fields: [conversationId], references: [id], onDelete: Cascade)
  meaningBn String? @map("meaning_bn")
  meaningEn String? @map("meaning_en")
  sentenceId String @map("sentence_id")
  sentence Sentence @relation(fields: [sentenceId], references: [id], onDelete: Cascade)
  speaker String @db.VarChar(50)
  position Int
  createdAt DateTime @default(now()) @map("created_at") @db.Timestamptz(6)
  updatedAt DateTime @updatedAt @map("updated_at") @db.Timestamptz(6)
  @@unique([conversationId, position])
  @@index([sentenceId])
  @@map("conversation_line")
}
```

`Conversation` has no scalar content of its own — it's just a `topicConversationId` + an ordered
list of lines, so create/update work on the whole `lines[]` array at once, same as how
`SentenceRepository.create`/`.update` handles the `words[]` array via nested writes.

New folder: `server/src/modules/conversation/`

---

## `conversation.validation.ts`

```ts
import { PAGINATION } from "@/shared/constants.js";
import { z } from "zod";

// Each entry is one line of dialogue, referencing an existing Sentence.
const conversationLineSchema = z.object({
  sentenceId: z.string().min(1),
  speaker: z.string().trim().min(1).max(50),
  position: z.coerce.number().int().nonnegative(),
  meaningEn: z.string().trim().optional(),
  meaningBn: z.string().trim().optional(),
});

export const createConversationSchema = z.object({
  topicConversationId: z.string().min(1),
  lines: z.array(conversationLineSchema).min(1, "At least one line is required"),
});

// On update, `lines` (if provided) fully replaces the existing set - same
// replace-all-then-recreate approach sentence.repository.ts uses for categories.
export const updateConversationSchema = z.object({
  topicConversationId: z.string().min(1).optional(),
  lines: z.array(conversationLineSchema).optional(),
});

export const conversationIdParamSchema = z.object({
  id: z.string().min(1),
});

export const listConversationsQuerySchema = z.object({
  page: z.coerce.number().int().positive().default(PAGINATION.DEFAULT_PAGE),
  limit: z.coerce.number().int().positive().max(PAGINATION.MAX_LIMIT).default(PAGINATION.DEFAULT_LIMIT),
  topicConversationId: z.string().optional(),
});

export type CreateConversationInput = z.infer<typeof createConversationSchema>;
export type UpdateConversationInput = z.infer<typeof updateConversationSchema>;
export type ListConversationsQuery = z.infer<typeof listConversationsQuerySchema>;
```

---

## `conversation.repository.ts`

```ts
// Database access layer for conversation operations.
import { Prisma } from "@/generated/prisma/client.js";
import { prisma } from "../../config/database.js";
import { CreateConversationInput, UpdateConversationInput } from "./conversation.validation.js";

export const CONVERSATION_INCLUDE = {
  topicConversation: true,
  lines: {
    include: { sentence: { include: { arabic: true } } },
    orderBy: { position: "asc" },
  },
} satisfies Prisma.ConversationInclude;

export const ConversationRepository = {
  findMany: (
    where: Prisma.ConversationWhereInput,
    skip: number,
    take: number,
  ) =>
    prisma.conversation.findMany({
      where,
      include: CONVERSATION_INCLUDE,
      orderBy: { createdAt: "desc" },
      skip,
      take,
    }),

  count: (where: Prisma.ConversationWhereInput) =>
    prisma.conversation.count({ where }),

  findById: (id: string) =>
    prisma.conversation.findUnique({
      where: { id },
      include: CONVERSATION_INCLUDE,
    }),

  create: (data: CreateConversationInput) =>
    prisma.conversation.create({
      data: {
        topicConversationId: data.topicConversationId,
        lines: { create: data.lines },
      },
      include: CONVERSATION_INCLUDE,
    }),

  // Replaces all lines in one transaction when `lines` is provided, otherwise
  // just patches topicConversationId.
  update: (id: string, data: UpdateConversationInput) => {
    const { lines, topicConversationId } = data;

    if (!lines) {
      return prisma.conversation.update({
        where: { id },
        data: { ...(topicConversationId ? { topicConversationId } : {}) },
        include: CONVERSATION_INCLUDE,
      });
    }

    return prisma.$transaction(async (tx) => {
      await tx.conversationLine.deleteMany({ where: { conversationId: id } });
      await tx.conversationLine.createMany({
        data: lines.map((line) => ({ ...line, conversationId: id })),
      });
      return tx.conversation.update({
        where: { id },
        data: { ...(topicConversationId ? { topicConversationId } : {}) },
        include: CONVERSATION_INCLUDE,
      });
    });
  },

  delete: (id: string) => prisma.conversation.delete({ where: { id } }),
};
```

---

## `conversation.service.ts`

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
import { TopicConversationRepository } from "@/modules/topic-conversation/topic-conversation.repository.js";
import {
  CreateConversationInput,
  ListConversationsQuery,
  UpdateConversationInput,
} from "./conversation.validation.js";
import { ConversationRepository } from "./conversation.repository.js";

type ConversationListResult = {
  items: Awaited<ReturnType<typeof ConversationRepository.findMany>>;
  meta: { page: number; limit: number; total: number; totalPages: number };
};

export async function list(query: ListConversationsQuery) {
  const key = cacheKey(cacheNamespaces.conversations, query);
  const cached = await cacheGet<ConversationListResult>(key);
  if (cached) return cached;

  const { page, limit, topicConversationId } = query;

  const where = topicConversationId ? { topicConversationId } : {};

  const [items, total] = await Promise.all([
    ConversationRepository.findMany(where, (page - 1) * limit, limit),
    ConversationRepository.count(where),
  ]);

  const result: ConversationListResult = {
    items,
    meta: {
      page,
      limit,
      total,
      totalPages: Math.max(1, Math.ceil(total / limit)),
    },
  };
  await cacheSet(key, result, CACHE_TTL.CONVERSATIONS);
  return result;
}

export async function getById(id: string) {
  const conversation = await ConversationRepository.findById(id);
  if (!conversation) throw ApiError.notFound("Conversation not found");
  return conversation;
}

export async function create(input: CreateConversationInput) {
  const topicConversation = await TopicConversationRepository.findById(
    input.topicConversationId,
  );
  if (!topicConversation) throw ApiError.badRequest("Topic conversation does not exist");

  const conversation = await ConversationRepository.create(input);
  await invalidateCacheNamespace(cacheNamespaces.conversations);
  return conversation;
}

export async function update(id: string, input: UpdateConversationInput) {
  await getById(id); // 404s early if it doesn't exist

  if (input.topicConversationId) {
    const topicConversation = await TopicConversationRepository.findById(
      input.topicConversationId,
    );
    if (!topicConversation) throw ApiError.badRequest("Topic conversation does not exist");
  }

  const conversation = await ConversationRepository.update(id, input);
  await invalidateCacheNamespace(cacheNamespaces.conversations);
  return conversation;
}

export async function remove(id: string): Promise<void> {
  await getById(id);

  // Cascades to ConversationLine.
  await ConversationRepository.delete(id);
  await invalidateCacheNamespace(cacheNamespaces.conversations);
}
```

---

## `conversation.controller.ts`

```ts
import { Request, Response } from "express";
import { sendSuccess } from "@/lib/api-response.js";
import { asyncHandler } from "@/lib/async-handler.js";
import * as conversationService from "./conversation.service.js";
import { ListConversationsQuery } from "./conversation.validation.js";

export const list = asyncHandler(async (req: Request, res: Response) => {
  const query = (req as Request & { validatedQuery: ListConversationsQuery })
    .validatedQuery;
  const { items, meta } = await conversationService.list(query);
  sendSuccess(res, 200, "Conversations fetched", { items, meta });
});

export const getOne = asyncHandler(async (req: Request, res: Response) => {
  const conversation = await conversationService.getById(req.params.id as string);
  sendSuccess(res, 200, "Conversation fetched", conversation);
});

export const create = asyncHandler(async (req: Request, res: Response) => {
  const conversation = await conversationService.create(req.body);
  sendSuccess(res, 201, "Conversation created", conversation);
});

export const update = asyncHandler(async (req: Request, res: Response) => {
  const conversation = await conversationService.update(
    req.params.id as string,
    req.body,
  );
  sendSuccess(res, 200, "Conversation updated", conversation);
});

export const remove = asyncHandler(async (req: Request, res: Response) => {
  await conversationService.remove(req.params.id as string);
  sendSuccess(res, 200, "Conversation deleted");
});
```

---

## `conversation.routes.ts`

```ts
import { Router } from "express";
import { requireAuth } from "../../middlewares/auth.middleware.js";
import { validate } from "../../middlewares/validate.middleware.js";
import * as controller from "./conversation.controller.js";
import {
  conversationIdParamSchema,
  createConversationSchema,
  listConversationsQuerySchema,
  updateConversationSchema,
} from "./conversation.validation.js";

const router = Router();

// Reading is public - conversations are learner-facing content.
router.get(
  "/",
  validate({ query: listConversationsQuerySchema }),
  controller.list,
);
router.get(
  "/:id",
  validate({ params: conversationIdParamSchema }),
  controller.getOne,
);

// Writes require an authenticated user.
router.post(
  "/",
  requireAuth,
  validate({ body: createConversationSchema }),
  controller.create,
);
router.patch(
  "/:id",
  requireAuth,
  validate({ params: conversationIdParamSchema, body: updateConversationSchema }),
  controller.update,
);
router.delete(
  "/:id",
  requireAuth,
  validate({ params: conversationIdParamSchema }),
  controller.remove,
);

export default router;
```

---

## `conversation.types.ts`

```ts
/**
 * Conversation Module Types
 * Request/Response DTOs and related type definitions
 */

import { z } from "zod";
import {
  createConversationSchema,
  updateConversationSchema,
  listConversationsQuerySchema,
} from "./conversation.validation.js";

export type CreateConversationInput = z.infer<typeof createConversationSchema>;
export type UpdateConversationInput = z.infer<typeof updateConversationSchema>;
export type ListConversationsQuery = z.infer<typeof listConversationsQuerySchema>;

export interface ConversationLineResponse {
  id: string;
  position: number;
  speaker: string;
  meaningEn?: string | null;
  meaningBn?: string | null;
  sentence: {
    id: string;
    arabic: { text: string };
  };
}

export interface ConversationResponse {
  id: string;
  topicConversationId: string;
  topicConversation: {
    id: string;
    titleEn: string;
    titleBn?: string | null;
  };
  lines: ConversationLineResponse[];
  createdAt?: Date;
  updatedAt?: Date;
}

export interface ListConversationsResponse {
  items: ConversationResponse[];
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
  TOPIC_CONVERSATIONS: 1800,
  CONVERSATIONS: 1800, // ADD THIS
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
  topicConversations: `${CACHE_PREFIX}:topic-conversations`,
  conversations: `${CACHE_PREFIX}:conversations`, // ADD THIS
} as const;
```

**3. `server/src/routes/index.ts`** — mount the router:

```ts
import conversationRoutes from "../modules/conversation/conversation.routes.js";
// ...
router.use("/conversations", conversationRoutes);
```

---

## Endpoints

| Method | Path                     | Auth | Description                                             |
|--------|--------------------------|------|-----------------------------------------------------------|
| GET    | `/api/v1/conversations`     | No   | List (paginated, filter by `topicConversationId`)        |
| GET    | `/api/v1/conversations/:id` | No   | Get one (lines ordered by `position`, each with its `sentence.arabic`) |
| POST   | `/api/v1/conversations`     | Yes  | Create with a full `lines[]` array                       |
| PATCH  | `/api/v1/conversations/:id` | Yes  | Update `topicConversationId` and/or fully replace `lines[]` |
| DELETE | `/api/v1/conversations/:id` | Yes  | Delete (cascades to lines)                                |

### Example create payload

```json
{
  "topicConversationId": "clx...",
  "lines": [
    { "sentenceId": "clx...", "speaker": "A", "position": 1, "meaningEn": "Hello", "meaningBn": "হ্যালো" },
    { "sentenceId": "clx...", "speaker": "B", "position": 2, "meaningEn": "Hi there", "meaningBn": "হাই" }
  ]
}
```

### Notes

- `topicConversationId` existence is checked explicitly in the service layer for a clean 400.
  `sentenceId` on each line is **not** individually pre-checked (would mean N extra queries per
  request) — an invalid `sentenceId` falls back to Prisma's raw FK violation, which
  `error-handler.middleware.ts` currently reports as a generic `400 "Database request failed"`.
  If that's not descriptive enough for the admin panel, add a bulk
  `prisma.sentence.findMany({ where: { id: { in: sentenceIds } } })` existence check before create/update.
- Duplicate `(conversationId, position)` pairs are caught by the `@@unique` constraint and already
  surface as a readable `409` via the existing `P2002` handling in `error-handler.middleware.ts`.