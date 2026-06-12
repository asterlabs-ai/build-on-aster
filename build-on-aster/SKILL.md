---
name: build-on-aster
description: Build, run, and automate on the Aster Agents platform via its REST API — create and update agents, knowledge bases, file uploads, skills, tags, and user invitations; invoke agents and read their results; all from code. Use whenever the task involves programmatically managing or running an Aster org (asteragents.com) from Claude Code, scripts, or any external application using an Aster API key.
---

# Building on Aster Agents via the API

You are helping someone build on the [Aster Agents](https://www.asteragents.com) platform from the outside — using their organization's API key, not platform-internal access. Everything here works with a standard API key; no database or internal tooling required.

## Ground rules

1. **Base URL**: `https://www.asteragents.com/api`
2. **Auth**: `Authorization: Bearer $ASTER_API_KEY` on every request. The key comes from Control Hub → Settings → API Access, is valid for **6 months**, and is scoped to one organization. Store it in an env var; never hardcode it.
3. **Everything is org-scoped.** You can only see and touch resources in the key's org. Admin endpoints (`/admin/*`, `/agents/sync-tools`) additionally require the key's user to be `org:admin`.
4. **Consult [reference.md](reference.md)** for exact request/response shapes before calling any endpoint — field names are precise and a few are surprising (see Gotchas). Do not guess fields.
5. **The platform ships fast.** For agent-building concepts (prompting, tools, KBs, extraction schemas, models), fetch the live docs at `https://docs.asteragents.com/<path>.md` (append `.md` to any docs path — e.g. `/features/build-an-agent.md`, `/features/choosing-a-model.md`, `/promptguide.md`). Never assume the model list or tool catalog from memory.

## The core workflows

### Create an agent (the right order)

1. `GET /tools` — pull the live tool catalog. Pick tools by `name`; copy each tool's `description` + `parameters` **verbatim** into your payload. Tools needing an integration show `requiredIntegration`; they only appear once the org connects it (pass `?all=true` to see what's gated).
2. Compose the `tools` field as an **object keyed by tool name** — not an array:
   ```json
   {
     "search_knowledge_base": {
       "description": "...from /tools...",
       "parameters": { "...from /tools..." : "..." },
       "config": { "accessibleKnowledgeBaseIds": [123, 124] }
     },
     "call_agent": { "description": "...", "parameters": {}, "config": { "callableAgentIds": ["45"] } }
   }
   ```
   Per-tool `config` is where cross-references live: `accessibleKnowledgeBaseIds` (search_knowledge_base, numbers), `writableKnowledgeBaseIds` (write_to_knowledge_base, numbers), `callableAgentIds` (call_agent, **strings**), `skillIds` (load_skill).
3. `POST /agents` with `name` (the only required field), `systemPrompt`, `model` (full `provider:model-id` string — check `/features/choosing-a-model.md` for current IDs), `tools`, `conversationStarters`, `stage` (`development` until tested, then `released`), `showInChat`.
4. **Update = same endpoint with `id`.** `GET /agents` → find the agent → modify the payload → `POST /agents` including `id`. There is no PATCH; send the full object.
5. **Clone pattern**: GET the source agent, strip `id` and `emailSlug`, edit, POST.

### Stand up a knowledge base with documents

```
POST /kb/manage                      → create KB (name + embeddingModel required; embeddingModel is IMMUTABLE after creation)
for each file:
  POST /upload/presigned             → { url, fileId, key, contentKey, metadata }
  PUT bytes to url                   → MUST set 4 headers (see Gotchas) or SignatureDoesNotMatch
  POST /kb/files                     → associate: { kbId, fileId, fileName, fileKey: <key>, contentKey, contentType, fileSize }
poll GET /kb/files?kbId=N            → until every file's processingStatus is "completed"
```

Then point an agent at it via `search_knowledge_base.config.accessibleKnowledgeBaseIds`. If files land in `parse_failed` / `chunk_failed` / `extract_failed`, surface them to the user — there is no retry endpoint in the public API (retry exists in the Control Hub UI).

**Extraction schemas** (optional, set at KB create/update): a JSON Schema of fields to pull from every file into structured `extractedData` — use when the user will ask aggregate questions across many similar documents (invoices, CIMs, reports). Set `extractionModel` too or nothing extracts.

**KB triggers**: `trigger: { enabled, agentId, prompt }` on the KB runs an agent automatically on every new file — the building block for "process each incoming document" pipelines.

### Invoke an agent (run it, not just build it)

`POST /agents` only *builds* agents. To actually **run** one, `POST /chat`. This is the endpoint that makes the platform programmable from Claude Code.

```
POST /chat
  header: X-Stream-Response: false        # the practical mode for automation
  body: {
    "agentId": 123,
    "message": { "role": "user", "parts": [{ "type": "text", "text": "..." }] }
    // omit "threadId" to start fresh; pass an existing one to continue a thread
  }
→ 202 { "threadId": "<uuid>", "status": "in_progress" }   (also in X-Thread-ID header)
```

Then **poll** `GET /getConversation?threadId=<uuid>` until `streamStatus` is terminal (`completed` / `stopped` / `error` / `timeout`), and read the assistant turn(s) from `messages[].parts`. The non-streaming 202+poll pattern is the right one for scripts and agents — it sidesteps platform request timeouts on long, multi-tool runs. (Omitting the header instead streams an SSE `text/event-stream`; only reach for that in a browser-like client.)

Prerequisites that bite: the org must have an API key for the agent's **model provider** set in Control Hub → Providers, or the run ends `streamStatus: "error"` with nothing useful. The agent must exist in your org (else `404`). To continue a multi-turn conversation, reuse the `threadId` — the body key is `threadId`, not `id`.

### Read what an agent did

`GET /getConversation?threadId=<uuid>` returns the full message history (`messages[].parts` with text, tool invocations, tool results) plus `streamStatus`. Thread IDs come from `POST /chat`, the Aster UI URL, or wherever the conversation was initiated. `renderedSystemPrompt` on the response is the exact prompt the agent ran with — useful for debugging behavior.

Server-side invocation paths that need no polling loop and run on their own: **scheduled tasks**, **email-to-agent** (`emailSlug`), and **KB triggers**. Reach for these when the work is recurring or event-driven rather than request/response.

### Package a skill

Skills = a `SKILL.md` (YAML frontmatter with `name` required + `description`) plus up to 20 bundled files.

```
POST /skills/manage    { skillMdContent: "---\nname: ...\n---\n..." }   (max 100KB)
POST /skills/files     { skillId, fileName, contentType }  → { uploadUrl } → PUT bytes (10-min expiry)
```

Attach to an agent via the `load_skill` tool with `config.skillIds`. Update SKILL.md through `PUT /skills/manage?id=N` (you cannot delete SKILL.md via /skills/files).

### Manage the org

- `POST /admin/invitations` — bulk invite (existing Clerk accounts are added directly, no email round-trip; idempotent for existing members).
- `POST /agent-tags` then `tagIds` on the agent — idempotent by name; use tags to keep the Control Hub organized as agent count grows.
- `POST /agents/sync-tools` (admin) — refresh every agent to latest tool schemas after the platform ships tool updates; per-tool `config` is preserved.

## Gotchas (each of these has burned someone)

- **Presigned PUT needs exactly these headers** or R2 rejects with `SignatureDoesNotMatch`: `Content-Type: application/octet-stream` (always octet-stream — NOT the file's real type; that's deliberate, the real type travels in metadata), `x-amz-meta-file-type: <metadata.contentType>`, `x-amz-meta-org-id: <metadata.orgId>`, `x-amz-meta-user-id: <metadata.userId>`. URL expires in 10 minutes. The response also includes a `headUrl` — issue a HEAD to it after upload to verify the byte count landed.
- **Field rename across the upload flow**: `/upload/presigned` returns `key`; `/kb/files` wants it as `fileKey`.
- **Two file IDs**: files have a `fileId` (UUID, used at association time) and an `id` (integer DB record, used for `DELETE /kb/files?fileId=<int>`). Deletes take the **integer**.
- **`tools` is an object, not an array of names.** Sending `["search_google"]` fails validation.
- **`callableAgentIds` are strings; `accessibleKnowledgeBaseIds` are numbers.** Yes, really.
- **`embeddingModel` cannot change after KB creation** — pick deliberately (`openai:text-embedding-3-small` is the safe default).
- **`POST /agents` returns 200 on create** (not 201) and is create-or-update by presence of `id` — a forgotten `id` silently creates a duplicate agent instead of updating.
- **`GET /agents` and `GET /agent-tags` return bare JSON arrays**, not `{ data: [...] }` wrappers; most other endpoints wrap (`{ success, knowledgeBases }`, `{ tools, total }`).
- **`emailSlug` is globally unique and validated** (lowercase, hyphens, reserved words rejected) — leave it off on clones.
- **Soft deletes everywhere**: deleting agents/KBs/skills hides them but conversations and files persist server-side. Treat deletes as irreversible from the API's perspective anyway.
- **Invoking returns 202, not the answer.** `POST /chat` with `X-Stream-Response: false` returns a `threadId` immediately and runs in the background — you must poll `GET /getConversation` for the result; the `202` body has no agent output. And to *continue* a thread the body key is `threadId`, NOT `id` — the wrong key silently starts a new conversation.
- **A run that 202s can still fail.** If the org lacks a provider API key for the agent's model, the request succeeds but the conversation ends `streamStatus: "error"`. Always check the terminal status, not just that the POST returned 202.

## How to behave when building

- **Read before write.** List the org's existing agents/KBs/tags first; reuse and update rather than duplicating. Names are how humans find things — collisions are confusing even where the API allows them.
- **Stage new agents as `development`** (and consider `showInChat: false` for infrastructure agents) until the user has tested them; promote to `released` explicitly.
- **Verify after mutating.** GET the resource back, and for KB ingests poll until terminal status — "uploaded" is not "searchable."
- **System prompts**: follow `https://docs.asteragents.com/promptguide.md`. Keep prompts timeless — query live sources rather than baking in today's facts. Prompt variables `{{CURRENT_DATE}}`, `{{USER_EMAIL}}`, `{{ORG_NAME}}` are available.
- **Don't burn the user's key**: on 401, the key is bad or expired (6-month life) — stop and tell them to mint a new one in Control Hub; don't retry.
- For full request/response detail on any endpoint: [reference.md](reference.md).
