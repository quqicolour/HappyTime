# HappyTime Architecture Design

## Goal

HappyTime is an Android app that lets a user import past conversations from
WeChat, Douyin, SMS, email, and similar sources, then chat with an AI companion
that preserves the other person's memories, tone, relationship context, and
conversation habits.

The first implementation will use a hybrid privacy architecture:

- Android native app for import, review, local encrypted cache, and chat UI.
- Node.js + TypeScript + Fastify backend for normalization, memory/persona
  extraction, retrieval, model orchestration, and streaming chat.
- PostgreSQL for structured records, with pgvector for semantic retrieval.
- DeepSeek and Qwen behind one model-provider interface.

The product must never imply that the AI is the real person. The assistant is a
simulation built from authorized imported data.

## Scope

The first version will implement:

- A backend monorepo skeleton.
- Standard message import API for JSON/CSV/TXT-derived records.
- Shared TypeScript schemas for normalized imported messages.
- Database schema for users, companions, sources, messages, memories, persona
  versions, chat sessions, chat messages, and model runs.
- A persona and memory prompt builder.
- DeepSeek/Qwen provider abstraction.
- Android architecture design and data contracts.

Platform-specific import adapters for WeChat, Douyin, SMS, and email will be
added after the standard import path is stable. The Android first version can
start by importing files that have already been exported or manually selected by
the user.

## Architecture

```text
android/
  app/
    Kotlin + Jetpack Compose UI
    Room local database
    Encrypted local storage
    Import review and sync client

apps/
  api/
    Fastify HTTP API
    Auth, import, companion, chat, memory routes

packages/
  shared/
    Zod schemas and TypeScript types
  core/
    Message normalization, memory extraction contracts, persona pipeline,
    prompt builder, retrieval contracts
  model-gateway/
    DeepSeek and Qwen provider implementations
```

Android owns device-facing capabilities and consent. The backend owns model
secrets, long-running extraction, vector retrieval, and chat generation.

## Android App Design

Technology:

- Kotlin.
- Jetpack Compose.
- Room.
- DataStore for settings.
- Android Keystore backed encryption for sensitive local state.
- Retrofit/OkHttp for backend API calls.

Primary screens:

- Companion list.
- Import source picker.
- Import preview and mapping screen.
- Companion profile and memory review.
- Chat screen with streaming responses.
- Privacy/data management screen.

Local data:

- Imported file metadata.
- Normalized message preview.
- Sync status.
- Recent chat messages.
- User privacy preferences.

The app should make data control visible: imported sources, extracted memories,
and persona versions must be inspectable and deletable.

## Backend Service Design

Fastify will expose:

- `GET /health`
- `POST /v1/imports`
- `GET /v1/companions/:id`
- `GET /v1/companions/:id/memories`
- `POST /v1/companions/:id/persona/rebuild`
- `POST /v1/chat/sessions`
- `POST /v1/chat/sessions/:id/messages`

Layering:

- Routes validate input and return API responses.
- Services coordinate business workflows.
- Repositories handle persistence.
- Model providers are dependency-injected behind interfaces.
- Prompt builders are pure functions wherever possible.

Errors use structured response bodies:

```json
{
  "status": "error",
  "code": "IMPORT_VALIDATION_FAILED",
  "message": "Some messages are missing sentAt or senderRole.",
  "details": []
}
```

## Database Schema

The backend stores canonical data. Android may keep encrypted local mirrors for
offline review and UI speed.

### users

- `id uuid primary key`
- `display_name text`
- `email text unique`
- `created_at timestamptz`
- `updated_at timestamptz`

### companions

- `id uuid primary key`
- `user_id uuid references users(id)`
- `display_name text`
- `relationship_label text`
- `description text`
- `created_at timestamptz`
- `updated_at timestamptz`

### data_sources

- `id uuid primary key`
- `user_id uuid references users(id)`
- `companion_id uuid references companions(id)`
- `source_type text`
- `source_name text`
- `import_status text`
- `message_count integer`
- `metadata jsonb`
- `created_at timestamptz`
- `updated_at timestamptz`

`source_type` values: `wechat`, `douyin`, `sms`, `email`, `imessage`,
`generic_json`, `generic_csv`, `generic_txt`.

### messages

- `id uuid primary key`
- `user_id uuid references users(id)`
- `companion_id uuid references companions(id)`
- `data_source_id uuid references data_sources(id)`
- `external_id text`
- `sender_role text`
- `sender_name text`
- `sent_at timestamptz`
- `content_type text`
- `text text`
- `attachment_refs jsonb`
- `metadata jsonb`
- `embedding vector`
- `created_at timestamptz`

`sender_role` values: `user`, `companion`, `other`, `system`.
`content_type` values: `text`, `image`, `audio`, `video`, `file`,
`sticker`, `mixed`, `unknown`.

### memories

- `id uuid primary key`
- `user_id uuid references users(id)`
- `companion_id uuid references companions(id)`
- `kind text`
- `summary text`
- `evidence_message_ids uuid[]`
- `confidence numeric`
- `status text`
- `embedding vector`
- `created_at timestamptz`
- `updated_at timestamptz`

`kind` values: `shared_event`, `preference`, `relationship_pattern`,
`communication_style`, `boundary`, `fact`, `emotion_trigger`.

### persona_profiles

- `id uuid primary key`
- `user_id uuid references users(id)`
- `companion_id uuid references companions(id)`
- `version integer`
- `status text`
- `profile jsonb`
- `source_memory_ids uuid[]`
- `created_at timestamptz`

The profile stores tone, vocabulary, emotional pattern, values, relationship
dynamics, boundaries, and prompt instructions.

### chat_sessions

- `id uuid primary key`
- `user_id uuid references users(id)`
- `companion_id uuid references companions(id)`
- `title text`
- `created_at timestamptz`
- `updated_at timestamptz`

### chat_messages

- `id uuid primary key`
- `session_id uuid references chat_sessions(id)`
- `role text`
- `content text`
- `model_provider text`
- `model_name text`
- `retrieved_memory_ids uuid[]`
- `created_at timestamptz`

`role` values: `user`, `assistant`, `system`, `tool`.

### model_runs

- `id uuid primary key`
- `user_id uuid references users(id)`
- `companion_id uuid references companions(id)`
- `provider text`
- `model_name text`
- `purpose text`
- `input_tokens integer`
- `output_tokens integer`
- `latency_ms integer`
- `status text`
- `error_message text`
- `created_at timestamptz`

## Standard Import Format

The backend accepts a normalized import batch. Platform adapters can transform
source-specific exports into this shape.

```json
{
  "sourceType": "wechat",
  "sourceName": "WeChat export - June",
  "companion": {
    "id": "optional-existing-companion-id",
    "displayName": "她",
    "relationshipLabel": "partner"
  },
  "messages": [
    {
      "externalId": "msg_001",
      "senderRole": "companion",
      "senderName": "她",
      "sentAt": "2024-06-01T12:30:00+08:00",
      "contentType": "text",
      "text": "今天有点累",
      "attachments": [],
      "metadata": {
        "platform": "wechat",
        "conversationTitle": "我和她"
      }
    }
  ]
}
```

Validation rules:

- `sourceType`, `companion.displayName`, and `messages` are required.
- Each message needs `senderRole`, `sentAt`, and `contentType`.
- Text messages need non-empty `text`.
- Unknown or unsupported media is preserved as metadata rather than discarded.
- Duplicate detection uses `data_source_id + external_id` when available, then a
  fallback hash of sender, timestamp, content type, and text.

## Persona And Memory Pipeline

The backend adapts the ex-skill idea into a service pipeline:

```text
raw source export
  -> platform adapter
  -> normalized import batch
  -> message persistence
  -> memory extraction
  -> memory merge and dedupe
  -> persona profile generation
  -> persona versioning
  -> retrieval during chat
  -> response generation
```

Memory extraction produces small, reviewable facts with evidence message IDs.
Persona generation consumes memories and conversation samples, then creates a
versioned profile. Corrections create new versions rather than mutating history.

## Prompt Template

The final chat prompt has four parts:

### System Policy

```text
You are HappyTime, an AI conversation simulation created from user-authorized
conversation history. You must not claim to be the real person, resurrected,
conscious, or able to know facts outside the provided context. Stay warm,
natural, and faithful to the persona profile. If the user asks whether you are
the real person, answer honestly that you are an AI simulation.
```

### Persona Profile

```text
Companion name: {{displayName}}
Relationship to user: {{relationshipLabel}}

Communication style:
{{communicationStyle}}

Common vocabulary and phrasing:
{{vocabulary}}

Emotional patterns:
{{emotionalPatterns}}

Relationship habits:
{{relationshipDynamics}}

Boundaries:
{{boundaries}}

Do:
{{dos}}

Avoid:
{{donts}}
```

### Retrieved Memories

```text
Relevant memories:
{{#memories}}
- {{summary}}
  Evidence: {{evidence}}
  Confidence: {{confidence}}
{{/memories}}
```

### Conversation Context

```text
Recent conversation:
{{recentMessages}}

User message:
{{userMessage}}
```

## Model Gateway

DeepSeek and Qwen are exposed through one interface:

```ts
export interface ChatModelProvider {
  id: "deepseek" | "qwen";
  streamChat(input: ChatCompletionInput): AsyncIterable<ChatDelta>;
  completeJson<T>(input: JsonCompletionInput): Promise<T>;
}
```

Provider config is environment-driven:

- `DEEPSEEK_API_KEY`
- `DEEPSEEK_BASE_URL`
- `DEEPSEEK_CHAT_MODEL`
- `QWEN_API_KEY`
- `QWEN_BASE_URL`
- `QWEN_CHAT_MODEL`
- `DEFAULT_CHAT_PROVIDER`

The app can choose the provider per companion or session, but the first version
uses `DEFAULT_CHAT_PROVIDER`.

## Privacy And Safety

Required controls:

- Explicit user confirmation before import.
- Delete imported source.
- Delete companion and all related data.
- Review and delete extracted memories.
- Do not put model API keys in the Android app.
- Do not log raw private messages in production logs.
- Mark AI responses as generated simulation in the product surface.
- Store model run metadata without storing full prompts unless explicitly
  enabled for debugging.

## Testing Strategy

Backend tests:

- Import schema accepts valid batches and rejects invalid sender/timestamp data.
- Normalization preserves unknown metadata.
- Prompt builder includes system policy, persona, memories, and user message.
- Model gateway selects DeepSeek or Qwen from config.
- Chat service records retrieved memories and assistant messages.

Android tests:

- Import preview maps parsed records to normalized messages.
- Local Room entities match shared API contracts.
- Chat UI handles streaming, retry, and provider errors.

## Implementation Order

1. Create backend monorepo skeleton.
2. Add shared schemas and tests.
3. Add import API with in-memory repository for first tests.
4. Add prompt builder and model gateway abstraction.
5. Add database migration files.
6. Add Android project skeleton and API contract docs.
7. Replace in-memory repositories with PostgreSQL.
8. Add platform-specific import adapters.

