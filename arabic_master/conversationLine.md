# ConversationLine Module — CRUD

Same folder-per-domain pattern. Model already exists in `schema.prisma`:

```prisma
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

## The find-or-create-sentence flow (core requirement)

`meaningEn`/`meaningBn` live on the line itself (not the sentence) exactly because the same
sentence can mean something slightly different depending on which conversation it's used in — so
these two are always taken from the request as-is.

For `sentenceId`, the create/update body accepts **either**:

- `sentenceId` — reference an existing `Sentence` directly, **or**
- `text` — Arabic text to resolve:
  1. Look up `ArabicText` by exact text (reusing the same uniqueness the `sentence` module relies on).
  2. If a `Sentence` already exists for that text → reuse its `id`.
  3. If not → create it as `PENDING` and enqueue it on the existing AI background job
     (`sentence.queue.ts` / `sentence.worker.ts`) — **by calling `processNewSentence` from
     `sentence.service.ts` directly**, so this module doesn't duplicate that logic. The line is
     linked to that new (still-processing) sentence immediately; the worker fills in
     meaning/pronunciation/category asynchronously, same as `POST /sentences/ai`.

Non-Arabic input is translated first via `translateWord` (the same AI helper `sentence.controller.ts`
and `word.controller.ts` already use), so the controller mirrors that existing pattern.

New folder: `server/src/modules/conversation-line/`

---

## `conversation-line.validation.ts`

```ts
import { PAGINATION } from "@/shared/constants.js";
import { z } from "zod";

const baseFields = {
  speaker: z.string().trim().min(1).max(50),
  position: z.coerce.number().int().nonnegative(),
  meaningEn: z.string().trim().optional(),
  meaningBn: z.string().trim().optional()
};

// Exactly one of sentenceId / text must be provided - sentenceId reuses an
// existing sentence directly, text triggers the find-or-create-via-AI flow.
export const createConversationLineSchema = z
  .object({
    conversationId: z.string().min(1),
    sentenceId: z.string().min(1).optional(),
    text: z.string().trim().min(1).optional(),
    ...baseFields
  })
  .refine((data) => Boolean(data.sentenceId) !== Boolean(data.text), {
    message: "Provide exactly one of sentenceId or text",
    path: ["sentenceId"]
  });

export const updateConversationLineSchema = z
  .object({
    sentenceId: z.string().min(1).optional(),
    text: z.string().trim().min(1).optional(),
    speaker: baseFields.speaker.optional(),
    position: baseFields.position.optional(),
    meaningEn: baseFields.meaningEn,
    meaningBn: baseFields.meaningBn
  })
  .refine((data) => !(data.sentenceId && data.text), {
    message: "Provide only one of sentenceId or text, not both",
    path: ["sentenceId"]
  });

export const conversationLineIdParamSchema = z.object({
  id: z.string().min(1)
});

export const listConversationLinesQuerySchema = z.object({
  page: z.coerce.number().int().positive().default(PAGINATION.DEFAULT_PAGE),
  limit: z.coerce
    .number()
    .int()
    .positive()
    .max(PAGINATION.MAX_LIMIT)
    .default(PAGINATION.DEFAULT_LIMIT),
  conversationId: z.string().optional()
});

export type CreateConversationLineInput = z.infer<
  typeof createConversationLineSchema
>;
export type UpdateConversationLineInput = z.infer<
  typeof updateConversationLineSchema
>;
export type ListConversationLinesQuery = z.infer<
  typeof listConversationLinesQuerySchema
>;
```

---

## `conversation-line.repository.ts`

```ts
// Database access layer for conversation-line operations.
import { Prisma } from "@/generated/prisma/client.js";
import { prisma } from "../../config/database.js";

export const CONVERSATION_LINE_INCLUDE = {
  sentence: { include: { arabic: true } }
} satisfies Prisma.ConversationLineInclude;

export interface CreateConversationLineData {
  conversationId: string;
  sentenceId: string;
  speaker: string;
  position: number;
  meaningEn?: string;
  meaningBn?: string;
}

export interface UpdateConversationLineData {
  sentenceId?: string;
  speaker?: string;
  position?: number;
  meaningEn?: string;
  meaningBn?: string;
}

export const ConversationLineRepository = {
  findMany: (
    where: Prisma.ConversationLineWhereInput,
    skip: number,
    take: number
  ) =>
    prisma.conversationLine.findMany({
      where,
      include: CONVERSATION_LINE_INCLUDE,
      orderBy: { position: "asc" },
      skip,
      take
    }),

  count: (where: Prisma.ConversationLineWhereInput) =>
    prisma.conversationLine.count({ where }),

  findById: (id: string) =>
    prisma.conversationLine.findUnique({
      where: { id },
      include: CONVERSATION_LINE_INCLUDE
    }),

  create: (data: CreateConversationLineData) =>
    prisma.conversationLine.create({
      data,
      include: CONVERSATION_LINE_INCLUDE
    }),

  update: (id: string, data: UpdateConversationLineData) =>
    prisma.conversationLine.update({
      where: { id },
      data,
      include: CONVERSATION_LINE_INCLUDE
    }),

  delete: (id: string) => prisma.conversationLine.delete({ where: { id } })
};
```

---

## `conversation-line.service.ts`

```ts
import { ApiError } from "@/lib/api-error.js";
import { CACHE_TTL } from "@/shared/constants.js";
import {
  cacheGet,
  cacheKey,
  cacheNamespaces,
  cacheSet,
  invalidateCacheNamespace
} from "@/integrations/cache.js";
import { ConversationRepository } from "@/modules/conversation/conversation.repository.js";
import { SentenceRepository } from "@/modules/sentence/sentence.repository.js";
import { processNewSentence } from "@/modules/sentence/sentence.service.js";
import {
  CreateConversationLineInput,
  ListConversationLinesQuery,
  UpdateConversationLineInput
} from "./conversation-line.validation.js";
import { ConversationLineRepository } from "./conversation-line.repository.js";

type ConversationLineListResult = {
  items: Awaited<ReturnType<typeof ConversationLineRepository.findMany>>;
  meta: { page: number; limit: number; total: number; totalPages: number };
};

// Reuse or create the sentence a line points to.
// - sentenceId given -> must already exist.
// - text given -> reuse if that Arabic text already has a Sentence,
//   otherwise create it as PENDING and hand it to the existing AI
//   background job (sentence.worker.ts fills in the rest).
async function resolveSentenceId(input: {
  sentenceId?: string;
  text?: string;
}): Promise<string> {
  if (input.sentenceId) {
    const sentence = await SentenceRepository.findById(input.sentenceId);
    if (!sentence) throw ApiError.badRequest("Sentence does not exist");
    return sentence.id;
  }

  const text = input.text as string;
  const existing = await SentenceRepository.findByArabicText(text);
  if (existing) return existing.id;

  const { sentenceId } = await processNewSentence(text);
  return sentenceId;
}

async function invalidateRelatedCaches() {
  // A line change also changes what GET /conversations/:id returns
  // (lines are nested there), so both namespaces need clearing.
  await Promise.all([
    invalidateCacheNamespace(cacheNamespaces.conversationLines),
    invalidateCacheNamespace(cacheNamespaces.conversations)
  ]);
}

export async function list(query: ListConversationLinesQuery) {
  const key = cacheKey(cacheNamespaces.conversationLines, query);
  const cached = await cacheGet<ConversationLineListResult>(key);
  if (cached) return cached;

  const { page, limit, conversationId } = query;
  const where = conversationId ? { conversationId } : {};

  const [items, total] = await Promise.all([
    ConversationLineRepository.findMany(where, (page - 1) * limit, limit),
    ConversationLineRepository.count(where)
  ]);

  const result: ConversationLineListResult = {
    items,
    meta: {
      page,
      limit,
      total,
      totalPages: Math.max(1, Math.ceil(total / limit))
    }
  };
  await cacheSet(key, result, CACHE_TTL.CONVERSATION_LINES);
  return result;
}

export async function getById(id: string) {
  const line = await ConversationLineRepository.findById(id);
  if (!line) throw ApiError.notFound("Conversation line not found");
  return line;
}

export async function create(input: CreateConversationLineInput) {
  const conversation = await ConversationRepository.findById(
    input.conversationId
  );
  if (!conversation) throw ApiError.badRequest("Conversation does not exist");

  const sentenceId = await resolveSentenceId(input);

  const line = await ConversationLineRepository.create({
    conversationId: input.conversationId,
    sentenceId,
    speaker: input.speaker,
    position: input.position,
    meaningEn: input.meaningEn,
    meaningBn: input.meaningBn
  });

  await invalidateRelatedCaches();
  return line;
}

export async function update(id: string, input: UpdateConversationLineInput) {
  await getById(id); // 404s early if it doesn't exist

  const sentenceId =
    input.sentenceId || input.text ? await resolveSentenceId(input) : undefined;

  const { text: _text, sentenceId: _rawSentenceId, ...rest } = input;

  const line = await ConversationLineRepository.update(id, {
    ...rest,
    ...(sentenceId ? { sentenceId } : {})
  });

  await invalidateRelatedCaches();
  return line;
}

export async function remove(id: string): Promise<void> {
  await getById(id);

  // Only removes the line, not the underlying Sentence - it may be
  // referenced by other conversation lines.
  await ConversationLineRepository.delete(id);
  await invalidateRelatedCaches();
}
```

---

## `conversation-line.controller.ts`

```ts
import { Request, Response } from "express";
import { sendSuccess } from "@/lib/api-response.js";
import { asyncHandler } from "@/lib/async-handler.js";
import { translateWord } from "../ai/generateContent.js";
import * as conversationLineService from "./conversation-line.service.js";
import { ListConversationLinesQuery } from "./conversation-line.validation.js";

const arabicRegex = /^[\u0600-\u06FF\s]+$/;

// Same "translate if not Arabic" preprocessing sentence.controller.ts and
// word.controller.ts already do before handing text off to the AI flow.
async function normalizeText(req: Request) {
  if (req.body.text && !arabicRegex.test(req.body.text)) {
    req.body.text = await translateWord(req.body.text);
  }
}

export const list = asyncHandler(async (req: Request, res: Response) => {
  const query = (
    req as Request & { validatedQuery: ListConversationLinesQuery }
  ).validatedQuery;
  const { items, meta } = await conversationLineService.list(query);
  sendSuccess(res, 200, "Conversation lines fetched", { items, meta });
});

export const getOne = asyncHandler(async (req: Request, res: Response) => {
  const line = await conversationLineService.getById(req.params.id as string);
  sendSuccess(res, 200, "Conversation line fetched", line);
});

export const create = asyncHandler(async (req: Request, res: Response) => {
  await normalizeText(req);
  const line = await conversationLineService.create(req.body);
  sendSuccess(res, 201, "Conversation line created", line);
});

export const update = asyncHandler(async (req: Request, res: Response) => {
  await normalizeText(req);
  const line = await conversationLineService.update(
    req.params.id as string,
    req.body
  );
  sendSuccess(res, 200, "Conversation line updated", line);
});

export const remove = asyncHandler(async (req: Request, res: Response) => {
  await conversationLineService.remove(req.params.id as string);
  sendSuccess(res, 200, "Conversation line deleted");
});
```

---

## `conversation-line.routes.ts`

```ts
import { Router } from "express";
import { requireAuth } from "../../middlewares/auth.middleware.js";
import { validate } from "../../middlewares/validate.middleware.js";
import * as controller from "./conversation-line.controller.js";
import {
  conversationLineIdParamSchema,
  createConversationLineSchema,
  listConversationLinesQuerySchema,
  updateConversationLineSchema
} from "./conversation-line.validation.js";

const router = Router();

router.get(
  "/",
  validate({ query: listConversationLinesQuerySchema }),
  controller.list
);
router.get(
  "/:id",
  validate({ params: conversationLineIdParamSchema }),
  controller.getOne
);

router.post(
  "/",
  requireAuth,
  validate({ body: createConversationLineSchema }),
  controller.create
);
router.patch(
  "/:id",
  requireAuth,
  validate({
    params: conversationLineIdParamSchema,
    body: updateConversationLineSchema
  }),
  controller.update
);
router.delete(
  "/:id",
  requireAuth,
  validate({ params: conversationLineIdParamSchema }),
  controller.remove
);

export default router;
```

---

## `conversation-line.types.ts`

```ts
/**
 * ConversationLine Module Types
 * Request/Response DTOs and related type definitions
 */

import { z } from "zod";
import {
  createConversationLineSchema,
  updateConversationLineSchema,
  listConversationLinesQuerySchema
} from "./conversation-line.validation.js";

export type CreateConversationLineInput = z.infer<
  typeof createConversationLineSchema
>;
export type UpdateConversationLineInput = z.infer<
  typeof updateConversationLineSchema
>;
export type ListConversationLinesQuery = z.infer<
  typeof listConversationLinesQuerySchema
>;

export interface ConversationLineResponse {
  id: string;
  conversationId: string;
  speaker: string;
  position: number;
  meaningEn?: string | null;
  meaningBn?: string | null;
  sentence: {
    id: string;
    status: string;
    arabic: { text: string; audioUrl?: string | null };
  };
  createdAt?: Date;
  updatedAt?: Date;
}

export interface ListConversationLinesResponse {
  items: ConversationLineResponse[];
  meta: { page: number; limit: number; total: number; totalPages: number };
}
```

---

## Wiring it up (small edits to existing files)

**1. `server/src/modules/sentence/sentence.repository.ts`** — add the lookup this module relies on:

```ts
export const SentenceRepository = {
  // ...existing methods

  // Used by conversation-line's find-or-create flow.
  findByArabicText: (text: string) =>
    prisma.sentence.findFirst({
      where: { arabic: { text } },
      include: SENTENCE_INCLUDE
    })
};
```

**2. `server/src/shared/constants.ts`** — add a TTL entry:

```ts
export const CACHE_TTL = {
  CATEGORIES: 3600,
  WORDS: 1800,
  SENTENCES: 1800,
  TOPICS: 1800,
  TOPIC_CONVERSATIONS: 1800,
  CONVERSATIONS: 1800,
  CONVERSATION_LINES: 1800, // ADD THIS
  USER_PROFILE: 300
} as const;
```

**3. `server/src/integrations/cache.ts`** — register the namespace:

```ts
export const cacheNamespaces = {
  categories: `${CACHE_PREFIX}:categories`,
  words: `${CACHE_PREFIX}:words`,
  sentences: `${CACHE_PREFIX}:sentences`,
  topics: `${CACHE_PREFIX}:topics`,
  topicConversations: `${CACHE_PREFIX}:topic-conversations`,
  conversations: `${CACHE_PREFIX}:conversations`,
  conversationLines: `${CACHE_PREFIX}:conversation-lines` // ADD THIS
} as const;
```

**4. `server/src/routes/index.ts`** — mount the router:

```ts
import conversationLineRoutes from "../modules/conversation-line/conversation-line.routes.js";
// ...
router.use("/conversation-lines", conversationLineRoutes);
```

---

## Endpoints

| Method | Path                             | Auth | Description                                                           |
| ------ | -------------------------------- | ---- | --------------------------------------------------------------------- |
| GET    | `/api/v1/conversation-lines`     | No   | List (paginated, filter by `conversationId`)                          |
| GET    | `/api/v1/conversation-lines/:id` | No   | Get one                                                               |
| POST   | `/api/v1/conversation-lines`     | Yes  | Create - `sentenceId` OR `text` (find-or-create-via-AI)               |
| PATCH  | `/api/v1/conversation-lines/:id` | Yes  | Update - can re-point to a different sentence via `sentenceId`/`text` |
| DELETE | `/api/v1/conversation-lines/:id` | Yes  | Delete (sentence itself is untouched)                                 |

### Example create payloads

Reusing an existing sentence, with a conversation-specific meaning override:

```json
{
  "conversationId": "clx...",
  "sentenceId": "clx...",
  "speaker": "A",
  "position": 1,
  "meaningEn": "Nice to meet you (formal, first meeting)",
  "meaningBn": "আপনার সাথে দেখা হয়ে ভালো লাগলো"
}
```

Brand-new Arabic text — creates the `Sentence` as `PENDING` and queues AI enrichment in the background,
same as `POST /sentences/ai`:

```json
{
  "conversationId": "clx...",
  "text": "تشرفنا",
  "speaker": "B",
  "position": 2,
  "meaningEn": "Pleasure to meet you"
}
```

The response's `sentence.status` will be `"PENDING"` in that case — poll `GET /sentences/:id` (or the
line itself) until the worker flips it to `COMPLETED`.
