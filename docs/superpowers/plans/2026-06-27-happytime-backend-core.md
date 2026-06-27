# HappyTime Backend Core Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the first testable backend core for HappyTime: shared import schemas, prompt construction, model-provider abstraction, Fastify API skeleton, and database migration draft.

**Architecture:** Use a TypeScript npm workspace with small packages. `packages/shared` owns Zod schemas and inferred types, `packages/core` owns pure domain logic, `packages/model-gateway` owns DeepSeek/Qwen provider selection and interfaces, and `apps/api` wires those pieces into Fastify routes. Start with in-memory import persistence so the API can be tested before PostgreSQL is introduced.

**Tech Stack:** Node.js, TypeScript, Fastify, Zod, node:test, tsx, PostgreSQL migration SQL.

---

## File Structure

- Create `package.json`: npm workspace root with scripts for tests, type checks, and builds.
- Create `tsconfig.base.json`: shared TypeScript compiler defaults.
- Create `packages/shared/package.json`: package manifest for shared schemas.
- Create `packages/shared/src/importSchemas.ts`: Zod schemas and TypeScript types for import batches.
- Create `packages/shared/src/apiTypes.ts`: common API success/error envelope types.
- Create `packages/shared/src/index.ts`: package exports.
- Create `packages/shared/test/importSchemas.test.ts`: schema behavior tests.
- Create `packages/core/package.json`: package manifest for domain logic.
- Create `packages/core/src/promptBuilder.ts`: persona + memories + recent conversation prompt builder.
- Create `packages/core/src/importNormalizer.ts`: normalization and duplicate-key helper.
- Create `packages/core/src/index.ts`: package exports.
- Create `packages/core/test/promptBuilder.test.ts`: prompt builder tests.
- Create `packages/core/test/importNormalizer.test.ts`: normalization tests.
- Create `packages/model-gateway/package.json`: package manifest for model abstraction.
- Create `packages/model-gateway/src/types.ts`: provider interfaces and input/output types.
- Create `packages/model-gateway/src/providerFactory.ts`: provider selection from config.
- Create `packages/model-gateway/src/providers/deepseek.ts`: DeepSeek OpenAI-compatible provider shell.
- Create `packages/model-gateway/src/providers/qwen.ts`: Qwen OpenAI-compatible provider shell.
- Create `packages/model-gateway/src/index.ts`: package exports.
- Create `packages/model-gateway/test/providerFactory.test.ts`: provider selection tests.
- Create `apps/api/package.json`: Fastify API package manifest.
- Create `apps/api/src/server.ts`: Fastify app factory.
- Create `apps/api/src/importStore.ts`: in-memory repository for first import endpoint tests.
- Create `apps/api/src/routes/health.ts`: health route.
- Create `apps/api/src/routes/imports.ts`: import route.
- Create `apps/api/src/index.ts`: server entrypoint.
- Create `apps/api/test/imports.test.ts`: Fastify route tests.
- Create `db/migrations/0001_initial.sql`: PostgreSQL/pgvector schema draft.
- Modify `README.md`: describe first development commands and architecture pointer.

## Task 1: Workspace Skeleton

**Files:**
- Create: `package.json`
- Create: `tsconfig.base.json`
- Modify: `README.md`

- [ ] **Step 1: Write root package metadata**

Create `package.json`:

```json
{
  "name": "happytime",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "workspaces": [
    "apps/*",
    "packages/*"
  ],
  "scripts": {
    "test": "node --test --import tsx \"packages/*/test/*.test.ts\" \"apps/*/test/*.test.ts\"",
    "typecheck": "tsc -p tsconfig.base.json --noEmit",
    "build": "tsc -p tsconfig.base.json"
  },
  "devDependencies": {
    "@types/node": "^22.10.0",
    "tsx": "^4.19.2",
    "typescript": "^5.7.2"
  },
  "dependencies": {
    "@fastify/cors": "^10.0.2",
    "@fastify/helmet": "^12.0.1",
    "fastify": "^5.1.0",
    "zod": "^3.24.1"
  }
}
```

- [ ] **Step 2: Write TypeScript config**

Create `tsconfig.base.json`:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true,
    "rootDir": ".",
    "outDir": "dist",
    "types": ["node"]
  },
  "include": [
    "apps/**/*.ts",
    "packages/**/*.ts"
  ]
}
```

- [ ] **Step 3: Update README**

Replace `README.md` with:

```markdown
# HappyTime

HappyTime is an Android-first AI memory companion app. Users import authorized
conversation history from sources such as WeChat, Douyin, SMS, and email, then
chat with an AI simulation powered by extracted memories and persona profiles.

Architecture design:

- `docs/superpowers/specs/2026-06-27-happytime-architecture-design.md`

First backend core:

- `packages/shared`: shared schemas and API types
- `packages/core`: import normalization and prompt construction
- `packages/model-gateway`: DeepSeek/Qwen provider abstraction
- `apps/api`: Fastify API

Development commands:

```bash
npm install
npm test
npm run typecheck
```
```

- [ ] **Step 4: Install dependencies**

Run: `npm install`

Expected: dependencies are installed and `package-lock.json` is created.

- [ ] **Step 5: Verify baseline test command fails only because tests are absent**

Run: `npm test`

Expected: the command exits with an error because the test glob does not match any files yet, or reports zero tests depending on the local Node version. This is acceptable before Task 2.

- [ ] **Step 6: Commit**

```bash
git add package.json package-lock.json tsconfig.base.json README.md
git commit -m "chore: add TypeScript workspace skeleton"
```

## Task 2: Shared Import Schemas

**Files:**
- Create: `packages/shared/package.json`
- Create: `packages/shared/src/importSchemas.ts`
- Create: `packages/shared/src/apiTypes.ts`
- Create: `packages/shared/src/index.ts`
- Test: `packages/shared/test/importSchemas.test.ts`

- [ ] **Step 1: Write failing schema tests**

Create `packages/shared/test/importSchemas.test.ts`:

```ts
import assert from "node:assert/strict";
import test from "node:test";
import { ImportBatchSchema } from "../src/importSchemas.ts";

test("ImportBatchSchema accepts a valid normalized text import", () => {
  const result = ImportBatchSchema.safeParse({
    sourceType: "wechat",
    sourceName: "WeChat export - June",
    companion: {
      displayName: "她",
      relationshipLabel: "partner",
    },
    messages: [
      {
        externalId: "msg_001",
        senderRole: "companion",
        senderName: "她",
        sentAt: "2024-06-01T12:30:00+08:00",
        contentType: "text",
        text: "今天有点累",
        attachments: [],
        metadata: {
          platform: "wechat",
        },
      },
    ],
  });

  assert.equal(result.success, true);
});

test("ImportBatchSchema rejects text messages with blank text", () => {
  const result = ImportBatchSchema.safeParse({
    sourceType: "sms",
    companion: {
      displayName: "Alex",
    },
    messages: [
      {
        senderRole: "user",
        sentAt: "2024-06-01T12:30:00+08:00",
        contentType: "text",
        text: "   ",
      },
    ],
  });

  assert.equal(result.success, false);
});

test("ImportBatchSchema preserves unknown media metadata", () => {
  const result = ImportBatchSchema.parse({
    sourceType: "generic_json",
    companion: {
      displayName: "Alex",
    },
    messages: [
      {
        senderRole: "companion",
        sentAt: "2024-06-01T12:30:00+08:00",
        contentType: "unknown",
        metadata: {
          originalType: "red_packet",
          raw: { amount: "hidden" },
        },
      },
    ],
  });

  assert.equal(result.messages[0]?.metadata?.originalType, "red_packet");
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `node --test --import tsx packages/shared/test/importSchemas.test.ts`

Expected: FAIL with module-not-found for `../src/importSchemas.ts`.

- [ ] **Step 3: Create shared package manifest**

Create `packages/shared/package.json`:

```json
{
  "name": "@happytime/shared",
  "version": "0.1.0",
  "type": "module",
  "private": true,
  "main": "src/index.ts"
}
```

- [ ] **Step 4: Implement import schemas**

Create `packages/shared/src/importSchemas.ts`:

```ts
import { z } from "zod";

export const SourceTypeSchema = z.enum([
  "wechat",
  "douyin",
  "sms",
  "email",
  "imessage",
  "generic_json",
  "generic_csv",
  "generic_txt",
]);

export const SenderRoleSchema = z.enum([
  "user",
  "companion",
  "other",
  "system",
]);

export const ContentTypeSchema = z.enum([
  "text",
  "image",
  "audio",
  "video",
  "file",
  "sticker",
  "mixed",
  "unknown",
]);

export const AttachmentSchema = z.object({
  id: z.string().min(1).optional(),
  type: ContentTypeSchema,
  uri: z.string().min(1).optional(),
  name: z.string().min(1).optional(),
  mimeType: z.string().min(1).optional(),
  metadata: z.record(z.unknown()).optional(),
});

export const ImportMessageSchema = z
  .object({
    externalId: z.string().min(1).optional(),
    senderRole: SenderRoleSchema,
    senderName: z.string().min(1).optional(),
    sentAt: z.string().datetime({ offset: true }),
    contentType: ContentTypeSchema,
    text: z.string().optional(),
    attachments: z.array(AttachmentSchema).default([]),
    metadata: z.record(z.unknown()).default({}),
  })
  .superRefine((message, ctx) => {
    if (message.contentType === "text" && !message.text?.trim()) {
      ctx.addIssue({
        code: z.ZodIssueCode.custom,
        path: ["text"],
        message: "Text messages require non-empty text.",
      });
    }
  });

export const ImportCompanionSchema = z.object({
  id: z.string().uuid().optional(),
  displayName: z.string().min(1),
  relationshipLabel: z.string().min(1).optional(),
});

export const ImportBatchSchema = z.object({
  sourceType: SourceTypeSchema,
  sourceName: z.string().min(1).optional(),
  companion: ImportCompanionSchema,
  messages: z.array(ImportMessageSchema).min(1),
});

export type SourceType = z.infer<typeof SourceTypeSchema>;
export type SenderRole = z.infer<typeof SenderRoleSchema>;
export type ContentType = z.infer<typeof ContentTypeSchema>;
export type ImportMessage = z.infer<typeof ImportMessageSchema>;
export type ImportCompanion = z.infer<typeof ImportCompanionSchema>;
export type ImportBatch = z.infer<typeof ImportBatchSchema>;
```

- [ ] **Step 5: Add API types**

Create `packages/shared/src/apiTypes.ts`:

```ts
export type ApiSuccess<T> = {
  status: "success";
  data: T;
};

export type ApiError = {
  status: "error";
  code: string;
  message: string;
  details?: unknown[];
};

export type ApiResponse<T> = ApiSuccess<T> | ApiError;
```

- [ ] **Step 6: Add shared exports**

Create `packages/shared/src/index.ts`:

```ts
export * from "./apiTypes.ts";
export * from "./importSchemas.ts";
```

- [ ] **Step 7: Run test to verify it passes**

Run: `node --test --import tsx packages/shared/test/importSchemas.test.ts`

Expected: PASS, 3 tests.

- [ ] **Step 8: Commit**

```bash
git add packages/shared
git commit -m "feat: add shared import schemas"
```

## Task 3: Core Import Normalizer

**Files:**
- Create: `packages/core/package.json`
- Create: `packages/core/src/importNormalizer.ts`
- Create: `packages/core/src/index.ts`
- Test: `packages/core/test/importNormalizer.test.ts`

- [ ] **Step 1: Write failing normalizer tests**

Create `packages/core/test/importNormalizer.test.ts`:

```ts
import assert from "node:assert/strict";
import test from "node:test";
import { buildMessageDedupeKey, normalizeImportBatch } from "../src/importNormalizer.ts";

test("normalizeImportBatch trims text and sorts messages by sentAt", () => {
  const batch = normalizeImportBatch({
    sourceType: "wechat",
    companion: { displayName: "她" },
    messages: [
      {
        senderRole: "companion",
        sentAt: "2024-06-02T12:30:00+08:00",
        contentType: "text",
        text: "  later  ",
      },
      {
        senderRole: "user",
        sentAt: "2024-06-01T12:30:00+08:00",
        contentType: "text",
        text: " earlier ",
      },
    ],
  });

  assert.equal(batch.messages[0]?.text, "earlier");
  assert.equal(batch.messages[1]?.text, "later");
});

test("buildMessageDedupeKey prefers source and external id", () => {
  const key = buildMessageDedupeKey("source-1", {
    externalId: "msg-1",
    senderRole: "user",
    sentAt: "2024-06-01T12:30:00+08:00",
    contentType: "text",
    text: "hello",
    attachments: [],
    metadata: {},
  });

  assert.equal(key, "source-1:external:msg-1");
});

test("buildMessageDedupeKey falls back to stable message content", () => {
  const key = buildMessageDedupeKey("source-1", {
    senderRole: "user",
    sentAt: "2024-06-01T12:30:00+08:00",
    contentType: "text",
    text: "hello",
    attachments: [],
    metadata: {},
  });

  assert.equal(
    key,
    "source-1:fallback:user:2024-06-01T12:30:00+08:00:text:hello",
  );
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `node --test --import tsx packages/core/test/importNormalizer.test.ts`

Expected: FAIL with module-not-found for `../src/importNormalizer.ts`.

- [ ] **Step 3: Create core package manifest**

Create `packages/core/package.json`:

```json
{
  "name": "@happytime/core",
  "version": "0.1.0",
  "type": "module",
  "private": true,
  "main": "src/index.ts",
  "dependencies": {
    "@happytime/shared": "0.1.0"
  }
}
```

- [ ] **Step 4: Implement import normalizer**

Create `packages/core/src/importNormalizer.ts`:

```ts
import type { ImportBatch, ImportMessage } from "@happytime/shared";
import { ImportBatchSchema } from "@happytime/shared";

export function normalizeImportBatch(input: unknown): ImportBatch {
  const batch = ImportBatchSchema.parse(input);
  return {
    ...batch,
    sourceName: batch.sourceName?.trim(),
    companion: {
      ...batch.companion,
      displayName: batch.companion.displayName.trim(),
      relationshipLabel: batch.companion.relationshipLabel?.trim(),
    },
    messages: batch.messages
      .map((message) => ({
        ...message,
        senderName: message.senderName?.trim(),
        text: message.text?.trim(),
      }))
      .sort((left, right) => left.sentAt.localeCompare(right.sentAt)),
  };
}

export function buildMessageDedupeKey(
  dataSourceId: string,
  message: ImportMessage,
): string {
  if (message.externalId) {
    return `${dataSourceId}:external:${message.externalId}`;
  }

  return [
    dataSourceId,
    "fallback",
    message.senderRole,
    message.sentAt,
    message.contentType,
    message.text ?? "",
  ].join(":");
}
```

- [ ] **Step 5: Add core exports**

Create `packages/core/src/index.ts`:

```ts
export * from "./importNormalizer.ts";
```

- [ ] **Step 6: Run test to verify it passes**

Run: `node --test --import tsx packages/core/test/importNormalizer.test.ts`

Expected: PASS, 3 tests.

- [ ] **Step 7: Commit**

```bash
git add packages/core
git commit -m "feat: add import normalization core"
```

## Task 4: Persona And Memory Prompt Builder

**Files:**
- Modify: `packages/core/src/promptBuilder.ts`
- Modify: `packages/core/src/index.ts`
- Test: `packages/core/test/promptBuilder.test.ts`

- [ ] **Step 1: Write failing prompt builder tests**

Create `packages/core/test/promptBuilder.test.ts`:

```ts
import assert from "node:assert/strict";
import test from "node:test";
import { buildCompanionChatPrompt } from "../src/promptBuilder.ts";

test("buildCompanionChatPrompt includes safety policy persona memories and user message", () => {
  const prompt = buildCompanionChatPrompt({
    companion: {
      displayName: "她",
      relationshipLabel: "partner",
    },
    persona: {
      communicationStyle: "温柔，回答简短，会先回应情绪。",
      vocabulary: ["辛苦啦", "抱抱", "慢慢来"],
      emotionalPatterns: ["疲惫时希望被安静陪伴"],
      relationshipDynamics: ["习惯在晚上分享一天的小事"],
      boundaries: ["不要声称自己是真人"],
      dos: ["承认自己是 AI 模拟", "用自然中文回答"],
      donts: ["不要编造未提供的事实"],
    },
    memories: [
      {
        summary: "她曾在雨天和用户一起散步。",
        evidence: "2024-06-01 微信记录",
        confidence: 0.82,
      },
    ],
    recentMessages: [
      { role: "user", content: "今天下雨了" },
      { role: "assistant", content: "又想到那次散步了。" },
    ],
    userMessage: "你还记得我们下雨散步吗？",
  });

  assert.match(prompt.system, /AI conversation simulation/);
  assert.match(prompt.system, /must not claim to be the real person/);
  assert.match(prompt.context, /Companion name: 她/);
  assert.match(prompt.context, /辛苦啦/);
  assert.match(prompt.context, /她曾在雨天和用户一起散步/);
  assert.match(prompt.context, /User message:\n你还记得我们下雨散步吗？/);
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `node --test --import tsx packages/core/test/promptBuilder.test.ts`

Expected: FAIL with module-not-found for `../src/promptBuilder.ts`.

- [ ] **Step 3: Implement prompt builder**

Create `packages/core/src/promptBuilder.ts`:

```ts
export type CompanionPromptInput = {
  companion: {
    displayName: string;
    relationshipLabel?: string;
  };
  persona: {
    communicationStyle: string;
    vocabulary: string[];
    emotionalPatterns: string[];
    relationshipDynamics: string[];
    boundaries: string[];
    dos: string[];
    donts: string[];
  };
  memories: Array<{
    summary: string;
    evidence?: string;
    confidence?: number;
  }>;
  recentMessages: Array<{
    role: "user" | "assistant" | "system" | "tool";
    content: string;
  }>;
  userMessage: string;
};

export type CompanionChatPrompt = {
  system: string;
  context: string;
};

export function buildCompanionChatPrompt(
  input: CompanionPromptInput,
): CompanionChatPrompt {
  const system = [
    "You are HappyTime, an AI conversation simulation created from user-authorized conversation history.",
    "You must not claim to be the real person, resurrected, conscious, or able to know facts outside the provided context.",
    "Stay warm, natural, and faithful to the persona profile.",
    "If the user asks whether you are the real person, answer honestly that you are an AI simulation.",
  ].join(" ");

  const context = [
    `Companion name: ${input.companion.displayName}`,
    `Relationship to user: ${input.companion.relationshipLabel ?? "unspecified"}`,
    "",
    "Communication style:",
    input.persona.communicationStyle,
    "",
    "Common vocabulary and phrasing:",
    formatList(input.persona.vocabulary),
    "",
    "Emotional patterns:",
    formatList(input.persona.emotionalPatterns),
    "",
    "Relationship habits:",
    formatList(input.persona.relationshipDynamics),
    "",
    "Boundaries:",
    formatList(input.persona.boundaries),
    "",
    "Do:",
    formatList(input.persona.dos),
    "",
    "Avoid:",
    formatList(input.persona.donts),
    "",
    "Relevant memories:",
    formatMemories(input.memories),
    "",
    "Recent conversation:",
    formatRecentMessages(input.recentMessages),
    "",
    "User message:",
    input.userMessage,
  ].join("\n");

  return { system, context };
}

function formatList(values: string[]): string {
  return values.length > 0 ? values.map((value) => `- ${value}`).join("\n") : "- none";
}

function formatMemories(input: CompanionPromptInput["memories"]): string {
  if (input.length === 0) {
    return "- none";
  }

  return input
    .map((memory) => {
      const details = [
        `- ${memory.summary}`,
        memory.evidence ? `  Evidence: ${memory.evidence}` : undefined,
        typeof memory.confidence === "number"
          ? `  Confidence: ${memory.confidence}`
          : undefined,
      ].filter(Boolean);
      return details.join("\n");
    })
    .join("\n");
}

function formatRecentMessages(
  messages: CompanionPromptInput["recentMessages"],
): string {
  if (messages.length === 0) {
    return "- none";
  }

  return messages
    .map((message) => `${message.role}: ${message.content}`)
    .join("\n");
}
```

- [ ] **Step 4: Update core exports**

Modify `packages/core/src/index.ts`:

```ts
export * from "./importNormalizer.ts";
export * from "./promptBuilder.ts";
```

- [ ] **Step 5: Run test to verify it passes**

Run: `node --test --import tsx packages/core/test/promptBuilder.test.ts`

Expected: PASS, 1 test.

- [ ] **Step 6: Commit**

```bash
git add packages/core
git commit -m "feat: add companion prompt builder"
```

## Task 5: Model Gateway Provider Selection

**Files:**
- Create: `packages/model-gateway/package.json`
- Create: `packages/model-gateway/src/types.ts`
- Create: `packages/model-gateway/src/providerFactory.ts`
- Create: `packages/model-gateway/src/providers/deepseek.ts`
- Create: `packages/model-gateway/src/providers/qwen.ts`
- Create: `packages/model-gateway/src/index.ts`
- Test: `packages/model-gateway/test/providerFactory.test.ts`

- [ ] **Step 1: Write failing provider factory tests**

Create `packages/model-gateway/test/providerFactory.test.ts`:

```ts
import assert from "node:assert/strict";
import test from "node:test";
import { createModelProvider } from "../src/providerFactory.ts";

test("createModelProvider creates DeepSeek provider from config", () => {
  const provider = createModelProvider({
    defaultProvider: "deepseek",
    deepseek: {
      apiKey: "deepseek-key",
      baseUrl: "https://api.deepseek.com",
      chatModel: "deepseek-chat",
    },
    qwen: {
      apiKey: "qwen-key",
      baseUrl: "https://dashscope.aliyuncs.com/compatible-mode/v1",
      chatModel: "qwen-plus",
    },
  });

  assert.equal(provider.id, "deepseek");
});

test("createModelProvider creates Qwen provider from config", () => {
  const provider = createModelProvider({
    defaultProvider: "qwen",
    deepseek: {
      apiKey: "deepseek-key",
      baseUrl: "https://api.deepseek.com",
      chatModel: "deepseek-chat",
    },
    qwen: {
      apiKey: "qwen-key",
      baseUrl: "https://dashscope.aliyuncs.com/compatible-mode/v1",
      chatModel: "qwen-plus",
    },
  });

  assert.equal(provider.id, "qwen");
});

test("createModelProvider rejects missing provider credentials", () => {
  assert.throws(
    () =>
      createModelProvider({
        defaultProvider: "deepseek",
        deepseek: {
          apiKey: "",
          baseUrl: "https://api.deepseek.com",
          chatModel: "deepseek-chat",
        },
        qwen: {
          apiKey: "qwen-key",
          baseUrl: "https://dashscope.aliyuncs.com/compatible-mode/v1",
          chatModel: "qwen-plus",
        },
      }),
    /DeepSeek API key is required/,
  );
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `node --test --import tsx packages/model-gateway/test/providerFactory.test.ts`

Expected: FAIL with module-not-found for `../src/providerFactory.ts`.

- [ ] **Step 3: Create model gateway manifest**

Create `packages/model-gateway/package.json`:

```json
{
  "name": "@happytime/model-gateway",
  "version": "0.1.0",
  "type": "module",
  "private": true,
  "main": "src/index.ts"
}
```

- [ ] **Step 4: Create model gateway types**

Create `packages/model-gateway/src/types.ts`:

```ts
export type ProviderId = "deepseek" | "qwen";

export type ChatCompletionInput = {
  system: string;
  messages: Array<{
    role: "user" | "assistant" | "system";
    content: string;
  }>;
};

export type JsonCompletionInput = {
  system: string;
  user: string;
};

export type ChatDelta = {
  content: string;
  done: boolean;
};

export interface ChatModelProvider {
  id: ProviderId;
  streamChat(input: ChatCompletionInput): AsyncIterable<ChatDelta>;
  completeJson<T>(input: JsonCompletionInput): Promise<T>;
}

export type ProviderRuntimeConfig = {
  apiKey: string;
  baseUrl: string;
  chatModel: string;
};

export type ModelGatewayConfig = {
  defaultProvider: ProviderId;
  deepseek: ProviderRuntimeConfig;
  qwen: ProviderRuntimeConfig;
};
```

- [ ] **Step 5: Implement provider shells**

Create `packages/model-gateway/src/providers/deepseek.ts`:

```ts
import type {
  ChatCompletionInput,
  ChatDelta,
  ChatModelProvider,
  JsonCompletionInput,
  ProviderRuntimeConfig,
} from "../types.ts";

export class DeepSeekProvider implements ChatModelProvider {
  readonly id = "deepseek" as const;

  constructor(private readonly config: ProviderRuntimeConfig) {
    if (!config.apiKey) {
      throw new Error("DeepSeek API key is required.");
    }
  }

  async *streamChat(_input: ChatCompletionInput): AsyncIterable<ChatDelta> {
    throw new Error("DeepSeek streaming is not implemented yet.");
  }

  async completeJson<T>(_input: JsonCompletionInput): Promise<T> {
    throw new Error("DeepSeek JSON completion is not implemented yet.");
  }
}
```

Create `packages/model-gateway/src/providers/qwen.ts`:

```ts
import type {
  ChatCompletionInput,
  ChatDelta,
  ChatModelProvider,
  JsonCompletionInput,
  ProviderRuntimeConfig,
} from "../types.ts";

export class QwenProvider implements ChatModelProvider {
  readonly id = "qwen" as const;

  constructor(private readonly config: ProviderRuntimeConfig) {
    if (!config.apiKey) {
      throw new Error("Qwen API key is required.");
    }
  }

  async *streamChat(_input: ChatCompletionInput): AsyncIterable<ChatDelta> {
    throw new Error("Qwen streaming is not implemented yet.");
  }

  async completeJson<T>(_input: JsonCompletionInput): Promise<T> {
    throw new Error("Qwen JSON completion is not implemented yet.");
  }
}
```

- [ ] **Step 6: Implement provider factory**

Create `packages/model-gateway/src/providerFactory.ts`:

```ts
import { DeepSeekProvider } from "./providers/deepseek.ts";
import { QwenProvider } from "./providers/qwen.ts";
import type { ChatModelProvider, ModelGatewayConfig } from "./types.ts";

export function createModelProvider(
  config: ModelGatewayConfig,
): ChatModelProvider {
  if (config.defaultProvider === "deepseek") {
    return new DeepSeekProvider(config.deepseek);
  }

  return new QwenProvider(config.qwen);
}
```

- [ ] **Step 7: Add exports**

Create `packages/model-gateway/src/index.ts`:

```ts
export * from "./providerFactory.ts";
export * from "./types.ts";
```

- [ ] **Step 8: Run test to verify it passes**

Run: `node --test --import tsx packages/model-gateway/test/providerFactory.test.ts`

Expected: PASS, 3 tests.

- [ ] **Step 9: Commit**

```bash
git add packages/model-gateway
git commit -m "feat: add model provider gateway"
```

## Task 6: Fastify API Skeleton And Import Route

**Files:**
- Create: `apps/api/package.json`
- Create: `apps/api/src/importStore.ts`
- Create: `apps/api/src/routes/health.ts`
- Create: `apps/api/src/routes/imports.ts`
- Create: `apps/api/src/server.ts`
- Create: `apps/api/src/index.ts`
- Test: `apps/api/test/imports.test.ts`

- [ ] **Step 1: Write failing API route tests**

Create `apps/api/test/imports.test.ts`:

```ts
import assert from "node:assert/strict";
import test from "node:test";
import { buildServer } from "../src/server.ts";

test("GET /health returns ok", async () => {
  const server = buildServer();
  const response = await server.inject({
    method: "GET",
    url: "/health",
  });

  assert.equal(response.statusCode, 200);
  assert.deepEqual(response.json(), { status: "ok" });
});

test("POST /v1/imports stores a normalized import batch", async () => {
  const server = buildServer();
  const response = await server.inject({
    method: "POST",
    url: "/v1/imports",
    payload: {
      sourceType: "wechat",
      sourceName: "WeChat export - June",
      companion: {
        displayName: "她",
        relationshipLabel: "partner",
      },
      messages: [
        {
          externalId: "msg_001",
          senderRole: "companion",
          senderName: "她",
          sentAt: "2024-06-01T12:30:00+08:00",
          contentType: "text",
          text: " 今天有点累 ",
        },
      ],
    },
  });

  assert.equal(response.statusCode, 201);
  assert.equal(response.json().status, "success");
  assert.equal(response.json().data.messageCount, 1);
  assert.equal(response.json().data.companion.displayName, "她");
});

test("POST /v1/imports rejects invalid import batches", async () => {
  const server = buildServer();
  const response = await server.inject({
    method: "POST",
    url: "/v1/imports",
    payload: {
      sourceType: "sms",
      companion: {
        displayName: "Alex",
      },
      messages: [
        {
          senderRole: "user",
          sentAt: "2024-06-01T12:30:00+08:00",
          contentType: "text",
          text: "",
        },
      ],
    },
  });

  assert.equal(response.statusCode, 400);
  assert.equal(response.json().status, "error");
  assert.equal(response.json().code, "IMPORT_VALIDATION_FAILED");
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `node --test --import tsx apps/api/test/imports.test.ts`

Expected: FAIL with module-not-found for `../src/server.ts`.

- [ ] **Step 3: Create API package manifest**

Create `apps/api/package.json`:

```json
{
  "name": "@happytime/api",
  "version": "0.1.0",
  "type": "module",
  "private": true,
  "main": "src/index.ts",
  "dependencies": {
    "@happytime/core": "0.1.0",
    "@happytime/shared": "0.1.0"
  }
}
```

- [ ] **Step 4: Implement in-memory import store**

Create `apps/api/src/importStore.ts`:

```ts
import type { ImportBatch } from "@happytime/shared";

export type StoredImport = {
  id: string;
  batch: ImportBatch;
  createdAt: string;
};

export class ImportStore {
  private readonly imports = new Map<string, StoredImport>();

  save(batch: ImportBatch): StoredImport {
    const id = `import_${this.imports.size + 1}`;
    const stored = {
      id,
      batch,
      createdAt: new Date().toISOString(),
    };
    this.imports.set(id, stored);
    return stored;
  }
}
```

- [ ] **Step 5: Implement routes**

Create `apps/api/src/routes/health.ts`:

```ts
import type { FastifyInstance } from "fastify";

export async function registerHealthRoutes(server: FastifyInstance) {
  server.get("/health", async () => ({ status: "ok" }));
}
```

Create `apps/api/src/routes/imports.ts`:

```ts
import { normalizeImportBatch } from "@happytime/core";
import type { FastifyInstance } from "fastify";
import { ZodError } from "zod";
import type { ImportStore } from "../importStore.ts";

export async function registerImportRoutes(
  server: FastifyInstance,
  store: ImportStore,
) {
  server.post("/v1/imports", async (request, reply) => {
    try {
      const batch = normalizeImportBatch(request.body);
      const stored = store.save(batch);
      return reply.status(201).send({
        status: "success",
        data: {
          importId: stored.id,
          companion: batch.companion,
          messageCount: batch.messages.length,
          createdAt: stored.createdAt,
        },
      });
    } catch (error) {
      if (error instanceof ZodError) {
        return reply.status(400).send({
          status: "error",
          code: "IMPORT_VALIDATION_FAILED",
          message: "Import batch validation failed.",
          details: error.issues,
        });
      }

      throw error;
    }
  });
}
```

- [ ] **Step 6: Implement server factory**

Create `apps/api/src/server.ts`:

```ts
import cors from "@fastify/cors";
import helmet from "@fastify/helmet";
import Fastify from "fastify";
import { ImportStore } from "./importStore.ts";
import { registerHealthRoutes } from "./routes/health.ts";
import { registerImportRoutes } from "./routes/imports.ts";

export function buildServer() {
  const server = Fastify({ logger: false });
  const importStore = new ImportStore();

  void server.register(helmet);
  void server.register(cors, { origin: true });
  void registerHealthRoutes(server);
  void registerImportRoutes(server, importStore);

  return server;
}
```

Create `apps/api/src/index.ts`:

```ts
import { buildServer } from "./server.ts";

const server = buildServer();
const port = Number(process.env.PORT ?? 3000);
const host = process.env.HOST ?? "0.0.0.0";

await server.listen({ port, host });
```

- [ ] **Step 7: Run test to verify it passes**

Run: `node --test --import tsx apps/api/test/imports.test.ts`

Expected: PASS, 3 tests.

- [ ] **Step 8: Commit**

```bash
git add apps/api
git commit -m "feat: add Fastify import API"
```

## Task 7: Database Migration Draft

**Files:**
- Create: `db/migrations/0001_initial.sql`

- [ ] **Step 1: Write migration SQL**

Create `db/migrations/0001_initial.sql`:

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE users (
  id uuid PRIMARY KEY,
  display_name text NOT NULL,
  email text UNIQUE,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE companions (
  id uuid PRIMARY KEY,
  user_id uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  display_name text NOT NULL,
  relationship_label text,
  description text,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE data_sources (
  id uuid PRIMARY KEY,
  user_id uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  companion_id uuid NOT NULL REFERENCES companions(id) ON DELETE CASCADE,
  source_type text NOT NULL,
  source_name text,
  import_status text NOT NULL,
  message_count integer NOT NULL DEFAULT 0,
  metadata jsonb NOT NULL DEFAULT '{}'::jsonb,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE messages (
  id uuid PRIMARY KEY,
  user_id uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  companion_id uuid NOT NULL REFERENCES companions(id) ON DELETE CASCADE,
  data_source_id uuid NOT NULL REFERENCES data_sources(id) ON DELETE CASCADE,
  external_id text,
  sender_role text NOT NULL,
  sender_name text,
  sent_at timestamptz NOT NULL,
  content_type text NOT NULL,
  text text,
  attachment_refs jsonb NOT NULL DEFAULT '[]'::jsonb,
  metadata jsonb NOT NULL DEFAULT '{}'::jsonb,
  embedding vector(1536),
  created_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE memories (
  id uuid PRIMARY KEY,
  user_id uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  companion_id uuid NOT NULL REFERENCES companions(id) ON DELETE CASCADE,
  kind text NOT NULL,
  summary text NOT NULL,
  evidence_message_ids uuid[] NOT NULL DEFAULT '{}',
  confidence numeric NOT NULL,
  status text NOT NULL,
  embedding vector(1536),
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE persona_profiles (
  id uuid PRIMARY KEY,
  user_id uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  companion_id uuid NOT NULL REFERENCES companions(id) ON DELETE CASCADE,
  version integer NOT NULL,
  status text NOT NULL,
  profile jsonb NOT NULL,
  source_memory_ids uuid[] NOT NULL DEFAULT '{}',
  created_at timestamptz NOT NULL DEFAULT now(),
  UNIQUE (companion_id, version)
);

CREATE TABLE chat_sessions (
  id uuid PRIMARY KEY,
  user_id uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  companion_id uuid NOT NULL REFERENCES companions(id) ON DELETE CASCADE,
  title text,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE chat_messages (
  id uuid PRIMARY KEY,
  session_id uuid NOT NULL REFERENCES chat_sessions(id) ON DELETE CASCADE,
  role text NOT NULL,
  content text NOT NULL,
  model_provider text,
  model_name text,
  retrieved_memory_ids uuid[] NOT NULL DEFAULT '{}',
  created_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE model_runs (
  id uuid PRIMARY KEY,
  user_id uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  companion_id uuid REFERENCES companions(id) ON DELETE SET NULL,
  provider text NOT NULL,
  model_name text NOT NULL,
  purpose text NOT NULL,
  input_tokens integer,
  output_tokens integer,
  latency_ms integer,
  status text NOT NULL,
  error_message text,
  created_at timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX messages_companion_sent_at_idx ON messages (companion_id, sent_at);
CREATE INDEX data_sources_companion_idx ON data_sources (companion_id);
CREATE INDEX memories_companion_kind_idx ON memories (companion_id, kind);
CREATE INDEX persona_profiles_companion_version_idx ON persona_profiles (companion_id, version DESC);
CREATE INDEX chat_messages_session_created_at_idx ON chat_messages (session_id, created_at);
```

- [ ] **Step 2: Verify SQL file contains all expected tables**

Run: `rg -n "CREATE TABLE (users|companions|data_sources|messages|memories|persona_profiles|chat_sessions|chat_messages|model_runs)" db/migrations/0001_initial.sql`

Expected: 9 matching table declarations.

- [ ] **Step 3: Commit**

```bash
git add db/migrations/0001_initial.sql
git commit -m "feat: add initial database schema"
```

## Task 8: Full Verification

**Files:**
- Modify only if verification reveals a defect in files from Tasks 1-7.

- [ ] **Step 1: Run full test suite**

Run: `npm test`

Expected: all package and API tests pass.

- [ ] **Step 2: Run TypeScript typecheck**

Run: `npm run typecheck`

Expected: TypeScript exits with code 0.

- [ ] **Step 3: Run build**

Run: `npm run build`

Expected: TypeScript emits JavaScript into `dist` and exits with code 0.

- [ ] **Step 4: Check git status**

Run: `git status --short`

Expected: only intended files are modified or untracked before final commit.

- [ ] **Step 5: Commit verification fixes if needed**

If any code changes were needed during verification:

```bash
git add .
git commit -m "fix: complete backend core verification"
```

If no changes were needed, do not create an empty commit.

## Self-Review

Spec coverage:

- Android native architecture is covered by the design spec, not implemented in
  this first backend-core plan.
- Database schema is covered by Task 7.
- Message import format is covered by Task 2 and Task 6.
- DeepSeek/Qwen abstraction is covered by Task 5.
- Persona + memory prompt template is covered by Task 4.
- ex-skill-to-backend pipeline starts with normalization, prompt construction,
  and provider abstraction here; memory extraction and persona versioning become
  the next implementation plan after this core slice is verified.

Placeholder scan:

- No unresolved marker text or vague deferred-work steps are present.
- Provider methods explicitly throw "not implemented yet" because this plan
  tests provider selection and interface shape, not live model calls.

Type consistency:

- Import schema names match normalizer imports.
- Prompt builder input names match test assertions.
- API route imports `normalizeImportBatch` from `@happytime/core`.
- Model factory returns the shared `ChatModelProvider` interface.
