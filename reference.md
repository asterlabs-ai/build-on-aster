# Aster Agents Public API Reference

Source: the platform OpenAPI spec (`docs/openapi.yaml`) + `docs/api-reference/introduction.mdx`. When in doubt, the live docs at https://docs.asteragents.com/api-reference/introduction.md win.

## Base URL & Authentication

- **Base URL**: `https://www.asteragents.com/api`
- **Auth header**: `Authorization: Bearer YOUR_API_KEY`
- API keys come from Control Hub Settings (`https://www.asteragents.com/control-hub/settings` → "API Access" → copy key). Keys are valid for **6 months**. The token is a Clerk JWT.
- All data is **organization-scoped** — you only see resources in your org.
- Permission tiers:
  - `/admin/*` — requires `org:admin` role
  - `/agents` POST — requires `org:manage_agents:create` (create) / `org:manage_agents:update` (update) permission, or `org:admin`
  - `/agents/sync-tools` — requires `org:admin`
  - Everything else (`/kb/*`, `/skills/*`, `/upload/*`, `/tools`, `/chat`, `/getConversation`, `/agent-tags`) — standard bearer auth
- Common errors: `400` `{ "error": "User is not part of any organization" }` (or validation detail), `403` `{ "error": "Unauthorized: User is not an admin in this organization" }`, `500` `{ "error": "Internal Server Error" }`. Error body shape is always `{ error: string, details?: object[] }`.

---

## Agents

### GET /agents — List agents
Returns **a bare JSON array** (not wrapped in an object) of all agents in the org, sorted alphabetically by name, each with its `tags`.

Agent object fields: `id` (int), `name` (string), `description` (string|null), `logoUrl` (string|null), `conversationStarters` (string[]|null), `model` (string|null, e.g. `"openai:gpt-4o"`), `maxToolRoundtrips` (int|null), `systemPrompt` (string|null), `tools` (object|null — tool definitions keyed by tool name), `mcpServers` (int[]|null — `mcp_server.id` values), `stage` (`development` | `released`), `showInChat` (bool), `emailSlug` (string|null), `createdAt`/`updatedAt` (ISO date-time), `tags` (`[{ id, name }]`).

### POST /agents — Create or update an agent
One endpoint for both: **omit `id` to create, include `id` to update**. There is no clone endpoint — clone by GET-ing the original, editing the payload, and POSTing without `id`.

**Update semantics (verified against `api/agents.ts`):** with `id`, the body is validated as a *partial* (`updateAgentSchema = insertAgentSchema.partial()`) and only the columns you send are written — `POST /agents { id, systemPrompt }` changes just the prompt. **Two exceptions reset to defaults when omitted:** `stage` → `"development"` and `showInChat` → `true` (the handler force-sets `stage: stage || "development"` and Zod injects `showInChat: true`). `tagIds` is left alone when omitted. **Safe pattern: read-modify-write** — GET the agent, mutate, POST back with `id` and the unchanged `stage`/`showInChat`. `name` is still required on every call (including updates).

Body fields (only `name` is required, for both create and update):
| Field | Type | Required | Notes |
|---|---|---|---|
| `id` | int | update only | Include to update existing agent |
| `name` | string | **yes** | |
| `description` | string | no | |
| `logoUrl` | string | no | |
| `conversationStarters` | string[] | no | e.g. `["How can I help you today?"]` |
| `model` | string | no | `provider:model` format — check live docs for current model IDs |
| `maxToolRoundtrips` | int (min 1) | no | e.g. `50` |
| `systemPrompt` | string | no | |
| `tools` | object | no | **Object keyed by tool name, NOT an array of names.** See below. |
| `integrations` | string[] | no | Enabled integration identifiers |
| `mcpServers` | int[] | no | MCP server IDs (`mcp_server.id` integers) |
| `stage` | enum | no | `development` (default) \| `released` |
| `showInChat` | bool | no | default `true` |
| `emailSlug` | string\|null | no | Lowercase alphanumeric + hyphens, no leading/trailing hyphen, max 63 chars, pattern `^[a-z0-9][a-z0-9-]*[a-z0-9]$\|^[a-z0-9]$`. Unique across all agents. Reserved slugs rejected (`admin`, `support`, `help`, `noreply`, `postmaster`, etc.); cannot match `agent-{id}` / `kb-{id}` patterns. |
| `tagIds` | int[] | no | Must be tags belonging to the org, e.g. `[1, 2]` |

**`tools` format** — an object keyed by tool name; two accepted value shapes:
- **Native tools**: `{ "scrape_url": { "description": "...", "parameters": { ...JSON Schema... } } }` — copy `description` + `parameters` verbatim from `GET /tools`. Schemas are auto-refreshed to the latest platform definitions on save.
- **Provider tools**: keyed under the `provider:tool` name with `{ "isProviderTool": true }`, e.g. `{ "google:code_execution": { "isProviderTool": true } }`.
- Per-tool config (e.g. `callableAgentIds` for `call_agent`, `accessibleKnowledgeBaseIds` for `search_knowledge_base`) goes under a `config` field within the tool entry and is preserved by `POST /agents/sync-tools`.

Response: `200` with the full Agent object (note: 200 even on create, not 201).
Errors: `400` validation (`{ error: "Validation Error", details: [{ message: "Required", path: ["name"] }] }`), reserved slug (`Email slug "admin" is reserved and cannot be used`), duplicate slug; `403` missing permission.

### DELETE /agents — Delete an agent
Soft-delete. **Body** (not query): `{ "agentId": 1 }` (required). Existing conversations are preserved; new ones can't start.
Response: `200` with the Agent object. Errors: `400` `"Agent ID is required"`, `403`.

### POST /agents/sync-tools — Sync agent tool schemas
**Requires `org:admin`.** No request body. Refreshes every agent in the org to the latest platform tool schemas, preserving per-tool config (`callableAgentIds`, `accessibleKnowledgeBaseIds`).
Response `200`: `{ success: bool, updated: int, total: int, agents: [{ id, name }] }`.
Error `403`: `{ "error": "Unauthorized: Only admins can sync agent tools" }`.

---

## Knowledge Bases

### GET /kb/manage — List knowledge bases
No parameters. Response `200`: `{ success: true, knowledgeBases: KnowledgeBase[] }`.

KnowledgeBase object: `id` (int), `name`, `description` (string|null), `embeddingModel` (string, e.g. `"openai:text-embedding-3-small"`), `embeddingDimensions` (int, e.g. 1536), `extractionModel` (string|null), `extractionSchema` (object|null), `source` (string|null — when set, KB is integration-managed), `trigger` (object|null), `createdAt`/`updatedAt` (ISO), `userId` (creator), `fileCount` (int), `lastFileUploadedAt` (ISO|null), `totalSize` (bytes|null), `status` (`empty` | `ready`).

### POST /kb/manage — Create a knowledge base
Body:
| Field | Type | Required | Notes |
|---|---|---|---|
| `name` | string | **yes** | Unique within the org |
| `embeddingModel` | string | **yes** | Model identifier, e.g. `"openai:text-embedding-3-small"`. ⚠️ The spec gives only this one example — check live docs for the valid list. |
| `description` | string\|null | no | |
| `extractionModel` | string\|null | no | Model for structured data extraction from files |
| `extractionSchema` | object\|null | no | **JSON Schema** defining what structured data to extract per file |
| `source` | string\|null | no | e.g. `"salesforce"`. Marks KB integration-managed: UI upload/delete disabled, API access unrestricted. |
| `trigger` | object | no | `{ enabled: bool, agentId: int (same org), prompt: string }` — runs the agent automatically when new files are added; `prompt` is appended after document context. |

Response `201`: `{ success: true, knowledgeBase: KnowledgeBase }`.
Errors `400`: `"Knowledge base name is required"`, `"A knowledge base with this name already exists"`, `"Embedding model is required"`.

### PUT /kb/manage?id={kbId} — Update a knowledge base
Query: `id` (int, required). Body: `name` (string, **required**), `description`, `extractionModel`, `extractionSchema`, `trigger` (same shapes as create). **The embedding model cannot be changed after creation.**
Response `200`: `{ success: true, knowledgeBase }`. Errors: `404` `"Knowledge base not found"`.

### DELETE /kb/manage?id={kbId} — Delete a knowledge base
Soft-delete; files/embeddings become inaccessible; cannot be undone.
Response `200`: `{ success: true, message: "Knowledge base deleted successfully" }`. Error: `404`.

---

## Files & Uploads

### Complete KB upload flow
```
1. POST /upload/presigned   → get signed URL + fileId/key/contentKey/metadata
2. PUT to signed URL        → upload bytes (MUST include the 4 headers below)
3. POST /kb/files           → associate with KB & trigger processing
```

### POST /upload/presigned — Get presigned upload URL
Body: `{ "filename": "quarterly-report.pdf", "contentType": "application/pdf" }` — both **required**.

Response `200`:
```json
{
  "url": "https://account-id.r2.cloudflarestorage.com/bucket/...?X-Amz-Algorithm=...",
  "headUrl": "https://...presigned HEAD URL for post-upload verification...",
  "method": "PUT",
  "fileId": "550e8400-e29b-41d4-a716-446655440000",
  "key": "org_xxx/uploads/550e8400-e29b-41d4-a716-446655440000/report.pdf",
  "contentKey": "org_xxx/contents/550e8400-e29b-41d4-a716-446655440000.md",
  "metadata": { "contentType": "application/pdf", "orgId": "org_xxx", "userId": "user_xxx" }
}
```
The PUT to `url` **must** include exactly these headers (they're part of the V4 signature) or it fails with `SignatureDoesNotMatch`:
- `Content-Type`: **`application/octet-stream`** — always octet-stream regardless of file type (deliberate: avoids Cloudflare edge inspection eating certain content types; the real type is preserved in metadata)
- `x-amz-meta-file-type`: `metadata.contentType` (the file's real content type)
- `x-amz-meta-org-id`: `metadata.orgId`
- `x-amz-meta-user-id`: `metadata.userId`

After uploading, optionally HEAD the `headUrl` and compare `content-length` to verify the upload. URL expires in **10 minutes**. Errors: `400`, `401` `{ "error": "Unauthorized" }`.

### POST /kb/files — Associate uploaded file with a KB
Triggers extraction, chunking/embedding, and (for PDFs) page-image generation. `orgId` comes from the JWT — don't send it.

Body (field-name mapping from `/upload/presigned`):
| Field | Type | Required | Maps from presigned response |
|---|---|---|---|
| `kbId` | int | **yes** | — (your KB ID, e.g. `123`) |
| `fileId` | uuid string | **yes** | `fileId` |
| `fileName` | string | **yes** | original filename |
| `fileKey` | string | **yes** | `key` ⚠️ note the rename: presigned returns `key`, this endpoint takes `fileKey` |
| `contentKey` | string | **yes** | `contentKey` |
| `contentType` | string | **yes** | `metadata.contentType` |
| `fileSize` | int | no (recommended) | bytes |
| `userMetadata` | object\|null | no | arbitrary metadata |

Response `201`: `{ success: true, file: KnowledgeBaseFile }` — file starts at `processingStatus: "parsing"`.
Errors: `400` `"File ID, name, file key, content key, and content type are required"` / `"Valid knowledge base ID is required"`; `401`; `404` `"Knowledge base not found"`.

### GET /kb/files?kbId={kbId} — List files in a KB
Query: `kbId` (int, required). Response `200`: `{ success: true, files: KnowledgeBaseFile[] }`.

KnowledgeBaseFile: `id` (int, DB record ID), `knowledgeBaseId` (int), `fileId` (uuid), `fileName`, `fileKey`, `contentKey`, `contentType`, `fileSize` (int|null), `processingStatus`, `processingError` (string|null), `imageKeys` (string[]|null, PDF page images), `userMetadata` (object|null), `extractedData` (object|null — populated when an extraction schema is configured), `createdAt`/`updatedAt` (ISO).

**`processingStatus` values**: `parsing`, `parsed`, `chunking`, `chunk_failed`, `parse_failed`, `extract_failed`, `completed`. Poll this endpoint to monitor progress; terminal success is `completed`.

Errors: `400` `"Valid knowledge base ID is required"`, `404` `"Knowledge base not found"`.

### DELETE /kb/files?kbId={kbId}&fileId={fileId} — Delete file from KB
Query: `kbId` (int, required) and `fileId` (int, required — **the DB record `id`, NOT the UUID**). Deletes the record and embeddings; cannot be undone.
Response `200`: `{ success: true }`. Errors: `400` `"Valid knowledge base ID and file ID are required for DELETE"`, `404` `"File not found"`.

⚠️ The spec defines only POST/GET/DELETE on `/kb/files` — no retry action is documented in the public API (retry exists in the Control Hub UI).

---

## Skills

A skill = a SKILL.md file (with YAML frontmatter) + up to 20 bundled files.

### GET /skills/manage — List skills / get one skill
- No params → `{ success: true, skills: [{ id, name, description|null, metadata|null, fileCount, createdAt, updatedAt }] }`
- `?id={skillId}` → `{ success: true, skill: { id, name, description, metadata, files: [{ id, fileId (uuid), fileName, contentType, fileSize|null, createdAt }], agents: [{ id, name }], skillMdContent: string|null } }`
- `404` if `id` not found.

### POST /skills/manage — Create a skill
Body: `{ "skillMdContent": "---\nname: PDF Report Generator\ndescription: ...\n---\n\n## Instructions\n..." }` (**required**, max 100KB).
`name` (required) and `description` (optional) are parsed from the YAML frontmatter; extra frontmatter fields are stored as `metadata`.
Response `201`: `{ success: true, skill: { id, name, description, fileCount } }`.
Error `400`: invalid SKILL.md or missing `name` in frontmatter.

### PUT /skills/manage?id={skillId} — Update a skill
Query: `id` (int, required). Body (all optional, but send at least one):
- `skillMdContent` (string, max 100KB) — replaces SKILL.md in storage and re-parses name/description from frontmatter
- `name`, `description` — direct metadata updates, **only used if `skillMdContent` is absent**

Response `200`. Errors: `400` invalid content/missing name, `404` skill not found.

### DELETE /skills/manage?id={skillId} — Delete a skill
Soft-delete; skill disappears from listings and agents can't load it; bundled files retained for recovery.
Response `200`: `{ success: true, message }`. Error: `404`.

### GET /skills/files?skillId={skillId} — List a skill's bundled files
Query: `skillId` (int, required). Response `200`: `{ success: true, files: [{ id, fileId (uuid), fileName, contentType, fileSize|null, createdAt }] }` (includes SKILL.md).
Errors: `400` valid skill ID required, `404` skill not found.

### POST /skills/files — Add a bundled file (two-step upload)
Body: `skillId` (int, **required**), `fileName` (string, **required** — may include subdirectory, e.g. `"scripts/generate_report.py"`), `contentType` (string, **required**, e.g. `"text/x-python"`), `fileSize` (int|null, optional).
Limits: max **20 files per skill**; names sanitized against path traversal; duplicate names within a skill rejected.
Response `201`: `{ success: true, file: { id, fileId, fileName, contentType }, uploadUrl }` — then `PUT` the file bytes directly to `uploadUrl` (expires in 10 minutes). ⚠️ Unlike `/upload/presigned`, no special metadata headers are documented for this PUT.
Errors: `400` missing fields / duplicate name / file limit reached, `404` skill not found.

### DELETE /skills/files?fileId={fileId} — Delete a bundled file
Query: `fileId` (int, required — DB record ID, not UUID). Removes from storage + DB. **SKILL.md cannot be deleted here** — replace it via `PUT /skills/manage`.
Response `200`: `{ success: true, message }`. Errors: `400` cannot delete SKILL.md, `404` file not found.

---

## Conversations

### POST /chat — Invoke an agent (start or continue a conversation)
The endpoint that **runs** an agent (`POST /agents` only builds them). Verified against the live implementation (`api/chat.ts`).

**Headers**: `Authorization: Bearer <key>`, `Content-Type: application/json`, and — for automation — `X-Stream-Response: false`.

**Body** (v5 message format):
| Field | Type | Required | Notes |
|---|---|---|---|
| `agentId` | int | **yes** | Agent must belong to your org |
| `message` | object | **yes**¹ | `{ "role": "user", "parts": [{ "type": "text", "text": "..." }] }` — v5 `parts` array, not a flat `content` string |
| `threadId` | uuid\|null | no | Omit → new conversation (a UUID is generated). Pass an existing one → continue that thread. **Key is `threadId`, not `id`.** |
| `timezone` | string | no | IANA tz (e.g. `"America/New_York"`) for date/time prompt variables |

¹ `message` is required for a normal call. (The `continue` flag that resumes without a new message is internal-only — CRON_SECRET callers — and not part of the public API key surface.)

**Two response modes**, chosen by the `X-Stream-Response` header:

- **Non-streaming** (`X-Stream-Response: false`) — **recommended for API/automation.** Returns `202 Accepted` immediately:
  ```json
  { "threadId": "550e8400-...", "status": "in_progress" }
  ```
  with an `X-Thread-ID` response header. The agent runs in the background (via `waitUntil`), dodging request timeouts on long multi-tool runs. **Poll** `GET /getConversation?threadId={threadId}` until `streamStatus` is terminal, then read `messages`.
- **Streaming** (header omitted or ≠ `false`) — returns `200` with a `text/event-stream` (AI SDK UI-message protocol) and an `X-Thread-ID` header. For browser-like clients only.

**The invoke → poll loop** (the core pattern for Claude Code users):
```
POST /chat   { agentId, message }   header X-Stream-Response: false   → 202 { threadId }
loop: GET /getConversation?threadId={threadId}
        → streamStatus "in_progress"  → wait, poll again
        → "completed"                 → read last assistant message's parts
        → "error"/"timeout"/"stopped" → surface to user (check provider key, agent config)
```

**Errors**: `400` `"Invalid request body"` (Zod `details`) / `"Message is required for non-continuation calls"`; `401` `"Unauthorized"` (bad/expired key); `404` `"Agent not found or unauthorized"` (not in your org); `500`.

⚠️ **A 202 is not success.** The POST returning 202 only means the run was *admitted*. If the org has no provider API key for the agent's `model` (Control Hub → Providers), the conversation ends `streamStatus: "error"` with no useful output. Always poll to the terminal status.

### GET /getConversation?threadId={uuid} — Get a conversation with messages
Query: `threadId` (uuid string, **required**). Conversation must belong to your org.

Response `200`:
- `id` (uuid), `agentId` (int), `userId`, `organizationId` (string|null), `name` (string|null), `upstreamConversationId` (uuid|null — parent for branched conversations), `renderedSystemPrompt` (string|null — the system prompt actually used, for audit), `streamStatus`, `createdAt`/`updatedAt` (ISO), `deletedAt` (ISO|null), `isDeleted` (bool, present/true if soft-deleted)
- `messages`: ordered array of `{ id (uuid), role: "user"|"assistant"|"system", parts: MessagePart[] }`
- MessagePart: `{ type: "text"|"tool-invocation"|"tool-result"|"file", text?, toolInvocation?: { toolCallId, toolName, args }, result?, file?: { name, contentType, url } }`

`streamStatus` values: `in_progress` (generating), `completed`, `stopped` (manually stopped), `error`, `timeout`, `null` (no active stream).

Errors: `400` `"threadId is required"`, `404` `"Conversation not found"` (or wrong org), `405` `"Method not allowed"` (non-GET).

---

## Tools

### GET /tools — Tool catalog
Returns the catalog of tools the org can attach to agents. Default response is filtered to **currently usable** tools (integration-gated tools hidden if not connected; private tools hidden unless admin-enabled; companion tools excluded).

Query: `all` (bool, default `false`) — when `true`, includes every tool with `available: false` and an `unavailableReason` (`missing_integration` | `private_not_enabled`).

Response `200`: `{ tools: ToolCatalogEntry[], total: int }`.

ToolCatalogEntry: `name` (string — the key to use in agent `tools`; provider tools use `provider:tool` form like `google:code_execution`), `displayName`, `description`, `category` (e.g. `"Research"`, `"Aster Agents Platform"`, `"Salesforce"`), `parameters` (object|null — JSON Schema of call arguments; **drop verbatim into the agent's `tools[name]` alongside `description`**; omitted for provider tools), `requiredIntegration` (string|null, e.g. `"salesforce"`, `"hubspot"`, `"box"`), `isPrivate` (bool — must be enabled via `enabledPrivateTools` in org settings), `isProviderTool` (bool — attach with `{ "isProviderTool": true }`), `provider` (string, provider tools only), `available` (bool), `unavailableReason` (enum, only when unavailable).

Don't cache the catalog — it changes as integrations connect and tools ship; call at agent-build time.

---

## Agent Tags

### GET /agent-tags — List tags
Returns a **bare array** of tags with usage counts: `[{ id (int), name, createdBy (user ID), createdAt, agentCount (int), ... }]`.

### POST /agent-tags — Create a tag
Body: `{ "name": "Production" }` — required, 1–50 characters. **Idempotent**: if a tag with the same name exists in the org, the existing tag is returned.
Response `200`: `{ id, name, organizationId, createdBy, createdAt, updatedAt }`.
Error `400`: validation (e.g. `"String must contain at least 1 character(s)"`).

Attach tags to agents via the `tagIds` field on `POST /agents`.

---

## Admin (all require `org:admin`; 403 otherwise)

### GET /admin/users — List active org users
Query: `limit` (int 1–500, default 100), `offset` (int, default 0), `role` (`org:admin` | `org:member`), `orderBy` (string, default `"-created_at"`; prefix `-` desc / `+` asc, e.g. `"+email"`, `"-last_sign_in_at"`).
Response `200`: `{ data: User[], totalCount: int }`. User: `id`, `email`, `firstName|null`, `lastName|null`, `username|null`, `profileImageUrl`, `role`, `publicMetadata` (org-scoped object), `joinedAt`/`lastSignInAt`/`createdAt` (Unix ms, nullable). Pending invites are NOT included — use `/admin/invitations`.

### DELETE /admin/users — Remove users (bulk)
Body: `{ "userIds": ["user_2ABC123DEF", ...] }` (1–50 Clerk IDs). Cannot remove yourself; irreversible.
Response: `200` all succeeded / `207` partial — both return `{ success, total, successful, failed, results: [{ userId, success }], errors?: [{ userId, success: false, error }] }`.

### GET /admin/invitations — List invitations
Query: `limit` (1–500, default 100), `offset` (default 0), `status` (comma-separated; default `pending`; values: `pending`, `accepted`, `revoked`, `expired`).
Response `200`: `{ data: Invitation[], totalCount }`. Invitation: `id` (`orginv_...`), `emailAddress`, `role`, `status`, `publicMetadata|null`, `createdAt`/`updatedAt` (Unix ms).

### POST /admin/invitations — Create invitations (bulk)
Body: `{ "invitations": [{ "email": "user@co.com", "role": "org:member", "metadata": {...} }] }` — 1–50 items; only `email` required per item; `role` defaults `org:member` (enum `org:admin`/`org:member`); `metadata` applied on accept. Smart handling: existing Clerk account → added directly as member (no email, result `type: "direct_membership"` with `userId`/`membershipId`); already a member → idempotent success; new email → invitation sent (result `type: "invitation"` with `invitationId`, `status: "pending"`). Invitations expire after 30 days.
Response: `200` all success / `207` partial; `{ success, total, successful, failed, results, errors }`. `400` on invalid email with Zod-style `details` path.

### DELETE /admin/invitations — Revoke invitations (bulk)
Body: `{ "invitationIds": ["orginv_2ABC123DEF", ...] }` (1–50). Revoked links stop working.
Response: `200` / `207` with BulkOperationResult.

### POST /admin/updateUserMetadata — Update user metadata
Body: `{ "userId": "user_2ABC123DEF", "metadata": { "department": "engineering" } }` — both required; metadata values are strings; **organization-scoped** (same user can have different metadata in different orgs).
Response `200`: `{ success, message }`.
