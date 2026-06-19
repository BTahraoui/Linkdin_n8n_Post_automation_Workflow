# Architecture

> Node-by-node deep dive of the LinkedIn n8n Auto-Poster workflow.

The workflow is organized as four logical layers (matching the colored sticky notes in n8n):

1. **Triggers** — when the workflow runs
2. **Content Generation** — what to write and draw about
3. **Image Generation** — making the cover image
4. **Publish & Log** — sending it live and telling you about it

![Architecture diagram](../assets/architecture/workflow-diagram.png)

---

## ⚙️ Conventions used in this document

| Symbol       | Meaning                                                              |
| ------------ | -------------------------------------------------------------------- |
| 🚦           | Trigger node                                                         |
| 🔀           | Routing / conditional                                                |
| 🧠           | Code node (JavaScript) with embedded business logic                  |
| 🤖           | LangChain / AI agent node                                            |
| 🌐           | HTTP request to an external API                                      |
| 📥 / 📤      | Input / output data shape                                            |
| ⚠️           | Failure path you should be aware of                                  |

---

## Layer 1 — Triggers

Two trigger nodes feed the rest of the workflow. Either will start a single execution.

### 🚦 `When clicking 'Execute workflow'` *(Manual Trigger)*
- **Position:** `[688, 112]`
- Connects → `Topic Randomiser`

Manually fires the workflow from the n8n editor or via the n8n REST API. Use this for testing.

### 🚦 `Schedule Trigger`
- **Position:** `[480, 112]`
- **Rule:** fires at `triggerAtHour: 8` and `triggerAtHour: 18` every day (UTC)
- Connects → `Topic Randomiser`

> Tip: n8n evaluates `triggerAtHour` against the **n8n instance's configured timezone**. If your UI shows "08:00", that's your local time, not necessarily UTC. Adjust to match when you want posts to go live.

Both triggers converge on the same next node — there is no race because each execution is independent and n8n serializes a workflow's executions by default.

---

## Layer 2 — Content Generation

This layer fans out into two parallel branches (post text + image prompt) that converge later in the pipeline.

### 🧠 `Topic Randomiser`
- **Position:** `[96, 480]`
- **Persistence:** keeps a rolling window of the last 7 used topics in `staticData.usedTopics` (scope: `global`)
- **Output:**
  ```json
  { "topic": "How DNS finds website IPs" }
  ```

**Logic**
1. Maintain a hard-coded list of ~40 networking / CCNA / cybersecurity topics.
2. Look at the last 7 topics used; pick uniformly at random from the remaining pool.
3. Append the chosen topic to `staticData.usedTopics`.

**Why the 7-topic window?** Beyond ~7, the pattern can become predictable. Inside ~7, it's basically without collision for a pool of 40.

> 📥 Topic
> 📤 `{ topic: string }`

### 🧠 `Style Randomiser`
- **Position:** `[80, 704]`
- **Input:** receives the topic from `Topic Randomiser`
- **Output:**
  ```json
  {
    "topic": "...",
    "styleName": "Neon Cyberpunk",
    "styleDescription": "Dark cyberpunk aesthetic with neon glow, ..."
  }
  ```

**Logic**
1. Maintain 8 visual styles.
2. Read `staticData.lastStyle`. If set, exclude it from the candidates pool.
3. Pick uniformly at random, write the choice back to `staticData.lastStyle`.

This guarantees you never get the **same style twice in a row**.

### 🤖 `OpenRouter Chat Model` *(LM Provider)*
- Attached **only** to `Post Maker`
- Model: `deepseek/deepseek-v4-flash`
- Credential: `OpenRouter account`

### 🤖 `OpenRouter Chat Model1` *(LM Provider)*
- Attached **only** to `Image Prompt Maker`
- Model: `deepseek/deepseek-chat`
- Credential: `OpenRouter account`

> Two different models on purpose. The writer is a fast instruct model; the image-prompt engineer is a stronger reasoner that can synthesize the topic with the visual style.

### 🤖 `Post Maker` *(LangChain Agent)*
- **Position:** `[480, 480]`
- **Retry policy:** 2 attempts, 5 s wait between
- **System message:** "You are a professional LinkedIn content creator…"
- **User prompt template:** instructs a ≤150-word post with a hook + emoji, bullets, ≤3 hashtags, no markdown, real-world analogy, CTA question
- **Output field:** `output` *(the raw post text)*

> ⚠️ The agent speaks English and doesn't follow markdown. Don't change this without also updating the `Post Trimmer`.

### 🤖 `Image Prompt Maker` *(LangChain Agent)*
- **Position:** `[-256, 592]`
- **Retry policy:** 2 attempts, 5 s wait between
- **Input fields:**
  - `{{ $json.topic }}` from `Style Randomiser`
  - `{{ $json.styleDescription }}`
- **Output field:** `output` *(the FLUX-formatted image prompt)*

The prompt is engineered to avoid text-in-image errors (which Flux is notorious for).

### 🛡 `Post Trimmer` *(Code)*
- **Position:** `[736, 480]`
- **Reads from:** `Post Maker` *(via `$('Post Maker').first().json.output`)*
- **Persistence:** none *(idempotent, runs every time)*

**Logic**
1. Strip the markdown LLM sometimes emits: `**bold**`, `*italic*`, leading `#` headings, ``` backticks.
2. Cap hashtags at exactly 3 by counting from the top.
3. Trim to ≤ 2900 characters on a word boundary (`...` suffix if truncated).
4. **Quality gate:** throw if the cleaned post is empty or < 50 chars — protects against publishing silent garbage.

> 📤 `{ output: string }` — the clean, final text used by both LinkedIn post types.

---

## Layer 3 — Image Generation

A linear pipeline: validate → resolve account ID (cached) → call FLUX → validate response → convert to binary.

### 📦 `Setup workflow` *(Set)*
- **Position:** `[0, 1152]`
- Materializes the FLUX request payload:
  - `prompt` ← `$('Image Prompt Maker').item.json.output`
  - `width` ← 1024 *(fallback if not provided upstream)*
  - `height` ← 1024
  - `seed` ← `Math.floor(Math.random() * 1e8)`
  - `steps` ← 6

### 🔀 `Check image size` *(If)*
- **Position:** `[288, 1152]`
- **Condition:** `$json.width * $json.height > 1048576` *(1 megapixel)*
- **True →** `Image > 1 megapixel` *(Stop and Error)*
- **False →** `Check Account Cache`

> ⚠️ Cloudflare Workers AI rejects requests larger than ~1 MP. Stay at or below — 1024×1024 is the sweet spot.

### 🛑 `Image > 1 megapixel` *(Stop and Error)*
- **Position:** `[496, 1136]`
- Message: "The size of the image (width x height) is larger than 1 megapixel"
- Hard-stops the workflow with a visible error. Change `width`/`height` in `Setup workflow` to recover.

### 🧠 `Check Account Cache` *(Code)*
- **Position:** `[176, 1376]`
- **Persistence:** reads `staticData.cfAccountId`
- **Output:** `{ cached: bool, accountId?: string }`

If we've already discovered the Cloudflare account ID once, we skip the API call. Saves ~250 ms per run and avoids rate limits.

### 🔀 `Account Cached?` *(If)*
- **True →** `Get Flux Schnell image` *(uses cached `accountId`)*
- **False →** `Get accounts` *(resolves the ID first)*

### 🌐 `Get accounts` *(HTTP Request)*
- **Method:** `GET https://api.cloudflare.com/client/v4/accounts`
- **Auth:** Bearer token
- **Retry policy:** 3 attempts, 5 s wait
- **Output:** `{ result: [{ id: "…", name: "…" }, …] }`

### 🧠 `Cache Account ID` *(Code)*
- **Position:** `[688, 1392]`
- Reads `result[0].id`, stores in `staticData.cfAccountId`, returns `{ accountId }`.

### 🌐 `Get Flux Schnell image` *(HTTP Request)*
- **Method:** `POST`
- **URL:** `https://api.cloudflare.com/client/v4/accounts/{accountId}/ai/run/@cf/black-forest-labs/flux-1-schnell`
- **Retry policy:** 3 attempts, 5 s wait
- **`onError: continueRegularOutput`** — the workflow continues even if FLUX errors; the next IF will route around it.
- **Body:** JSON payload assembled from the `Setup workflow` fields.

### 🔀 `Image OK?` *(If)*
- **Condition:** `!!($json.result && $json.result.image)`
- **True →** `Convert to File` *(image intact)*
- **False →** `Create text-only post` *(fallback path)*

### 📥 `Convert to File` *(Convert to File)*
- **Operation:** `toBinary`
- **Source property:** `result.image` *(the base64 string FLUX returns)*
- **Output:** a binary file (LinkedIn consumes directly)

---

## Layer 4 — Publish & Log

A `Merge` node joins the trimmed text with the binary image; LinkedIn publishes; we log and notify.

### 🔀 `Merge` *(Merge node)*
- **Position:** `[944, 496]`
- **Mode:** `combine` by **position** *(waits for both inputs to arrive)*
- **Input 1 (`index: 0`):** trimmed text from `Post Trimmer`
- **Input 2 (`index: 1`):** binary file from `Convert to File`
- **Output:** a single item containing both — perfect for the LinkedIn node that expects text + shareMediaCategory: IMAGE

### 📣 `Create a post` *(LinkedIn)*
- **Position:** `[1248, 496]` — primary path
- **Auth:** LinkedIn OAuth2 (`LinkedIn account`)
- **Person URN:** `QXR9DgjPn4` *(replace with your own)*
- **Visibility:** PUBLIC
- **shareMediaCategory:** IMAGE — uploads the binary from the upstream `Merge`
- **Retry policy:** 2 attempts, 5 s wait
- **`continueOnFail: true`** — failure won't halt the workflow. The next `Log Post` node will notice and report it.

### 📣 `Create text-only post` *(LinkedIn — fallback)*
- **Position:** `[1248, 704]`
- Same text payload, **no image attached**.
- Reached via the `Image OK? == false` branch — fires when FLUX fails.

### 🧠 `Log Post` *(Code)*
- **Position:** `[1472, 592]`
- **Persistence:** writes to `staticData.postLog` (scope: `global`)
- **Logic:**
  1. Find the result of `Create a post`. If it has an `error` OR wasn't reached, fall back to checking `Create text-only post`.
  2. Compose a structured log entry:
     ```json
     {
       "date": "2026-06-19T12:00:00.000Z",
       "topic": "…",
       "postLength": 287,
       "imageStyle": "Neon Cyberpunk",
       "postType": "image" | "text-only" | "unknown",
       "status": "posted" | "failed",
       "error": "…"
     }
     ```
  3. Append (keep only the last 50 to keep static data reasonable)

> 📤 Logged summary, also fed directly to Slack.

### 💬 `Send a message` *(Slack)*
- **Position:** `[1824, 592]`
- **Channel:** `#social` *(`select: "channel"`, resolves by name)*
- **Message:**
  ```
  Topic  : {topic}
  Status : posted | failed
  error  : {error or null}
  ```
- **Auth:** Slack API credential

---

## Data flow at a glance

```
Trigger ──┐
          ├─► [Topic Randomiser] ──┬─► [Post Maker] ─► [Post Trimmer] ─┐
          └─► [Schedule Trigger]  ├─► [Style Randomiser] ─► [Image Prompt Maker] ─► [Setup workflow] ─► [Size Check] ─┬─► [Account Cache] ─┬─► [Get Flux] ─► [Image OK?] ─► [Convert to File] ──┐
                                                                                                                                                                       └─► [Get Accounts] ─► [Cache Account ID] ─┘                                                                                                         │
                                                                                                                                                                                                                                                                                                                       ▼
                                                                                                                                                                                                                                                                                                              [Merge] ─► [Create a post (image)] ─┐
                                                                                                                                                                                                                                                                                                                              │               ▼
                                                                                                                                                                                                                                                                                                                              │         [Log Post] ─► [Slack #social]
                                                                                                                                                                                                                                                                                                                              ▼
                                                                                                                                                                                                                                                                                                          (image failed) [Create text-only post] ─┘
```

---

## State stored in workflow static data

| Key                  | Type                        | Purpose                                                  |
| -------------------- | --------------------------- | -------------------------------------------------------- |
| `usedTopics`         | `string[]`                  | Rolling window of topics already used (last 7)           |
| `lastStyle`          | `string \| undefined`       | Last style used; excluded from next pick                 |
| `cfAccountId`        | `string`                    | Cached Cloudflare account ID after first discovery       |
| `postLog`            | `Array<LogEntry>` *(≤50)*   | Last 50 runs, inspectable from any Code node             |

> Static data survives between executions of the **same instance**. If you migrate workflows or rename, you start with a clean slate.

---

## Why each layer is independent

- You can re-run **only** `Image Generation` without republishing by replacing the LinkedIn nodes with a no-op.
- You can swap the image generator without touching the post generator (the topic-style-prompt chain is the only thing they'd need to know about).
- You can disable Slack by removing one node — nothing else depends on it.

This isolation makes the workflow easy to extend without risk.
