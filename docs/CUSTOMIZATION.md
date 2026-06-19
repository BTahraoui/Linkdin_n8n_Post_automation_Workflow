# Customization Guide

> How to tailor the LinkedIn n8n Auto-Poster workflow to your niche, tone, and schedule.

This guide shows you exactly which nodes to edit to make the workflow your own — whether you want to post about different topics, use a different LLM, change the posting cadence, or tweak the prompts.

---

## 🎯 1. Change the Topics

The workflow picks topics from a hard-coded list in the **Topic Randomiser** node.

### Where to edit
- Open the workflow in n8n
- Find the node: **Topic Randomiser** (type: `n8n-nodes-base.code`)
- Look for the `topics` array near the top of the JavaScript code

### Example
```javascript
const topics = [
  "The OSI Model layers",
  "TCP vs UDP traffic",
  "How DNS finds website IPs",
  // ... your topics here
];
```

### Tips
- Keep each topic under ~80 characters for readability in prompts.
- Mix broad and narrow topics (e.g., "What is a VPN?" + "Split tunneling in WireGuard").
- Aim for 20–50 topics to keep variety high without maintenance overhead.
- **Do not** include markdown or special characters — they can confuse the LLMs.

> 💡 Pro tip: Add a comment `// TODO: rotate monthly` above the array and revisit it quarterly to keep content fresh.

---

## 🎨 2. Change the Visual Styles

The workflow selects a visual style (and its description) from the **Style Randomiser** node.

### Where to edit
- Node: **Style Randomiser** (`n8n-nodes-base.code`)
- Edit the `styles` array (each element has `name` and `description`)

### Example style object
```javascript
{ 
  name: "Neon Cyberpunk", 
  description: "Dark cyberpunk aesthetic with neon glow, holographic accents, deep shadows, electric colors" 
}
```

### Guidelines for good descriptions
- Keep it to **one sentence**.
- Focus on **visual keywords** that FLUX understands: lighting, mood, color palette, composition.
- Avoid complex sentences or rare words — Flux scrambles them.
- If you need specific text in the image, wrap it in quotes: `'The word "SECURE'"`.

### Preset styles (current)
1. 3D Render
2. Neon Cyberpunk
3. Flat Design
4. Isometric
5. Realistic Photo
6. Line Art
7. Glassmorphism
8. Retro Synthwave

Feel free to replace any or all — just keep the `name` and `description` keys.

---

## ✍️ 3. Tweak the Post Prompt (What the LLM writes)

The **Post Maker** node contains the instructions for the LinkedIn post.

### Where to edit
- Node: **Post Maker** (`@n8n/n8n-nodes-langchain.agent`)
- Field: **Prompt** → **Text**

### Current prompt (truncated for brevity)
```
=You are an expert technical writer and LinkedIn content creator managing a profile for a network technician refreshing their computer science and CCNA Security knowledge.

Write a LinkedIn post about this topic: {{ $json.topic }}

Follow these rules:
1. Language: Write in English.
2. Tone: Professional and encouraging.
3. Hook: Start with a catchy first sentence and a relevant emoji.
4. Simplicity: Use short sentences and a fun real-world analogy.
5. Formatting: Use bullet points and lots of white space.
6. Call to Action: End with a question to grow the network.
7. Length: Under 150 words.
8. Hashtags: Exactly 3 hashtags at the bottom.
9. No markdown: Do not use bold (**text**) or italic (*text*) formatting.
```

### What you can change
- **Role/audience:** Change "network technician refreshing their CCNA knowledge" to "cloud architect", "junior dev", etc.
- **Tone:** Swap "Professional and encouraging" for "Witty and irreverent" or "Academic and precise".
- **Hook requirement:** Remove or change the emoji requirement.
- **Formatting:** Prefer numbered lists? Change "bullet points".
- **Length:** Increase/decrease the 150-word limit (LinkedIn truncates after ~3k chars, but shorter performs better).
- **Hashtag count:** Change "Exactly 3" to 2 or 4 (but note: the Post Trimmer node enforces a max of 3; you’d need to update it too if you want more).
- **No markdown rule:** Keep this unless you also edit the `Post Trimmer` node’s regexes.

> ⚠️ If you change the rules, also check the **Post Trimmer** node — it strips markdown and enforces the 3-hashtag cap. Update its JavaScript if you change expectations here.

---

## 🖼 4. Tweak the Image Prompt (What FLUX gets)

The **Image Prompt Maker** node engineers the prompt for FLUX.1 Schnell.

### Where to edit
- Node: **Image Prompt Maker** (`@n8n/n8n-nodes-langchain.agent`)
- Field: **Prompt** → **Text**

### Current prompt
```
=You are an expert AI image prompt engineer for FLUX.1. Your job is to turn a simple IT topic into a sharp, high-quality visual prompt. 

Follow these strict rules to build the prompt for: {{ $json.topic }}

VISUAL STYLE: You MUST use this specific style: {{ $json.styleDescription }}

1. NO TEXT BLUNDERS: Do not include complex sentences, long words, or random numbers in the image description. Flux scrambles them. If text is absolutely needed, specify it clearly in quotation marks (e.g., 'The single word \\"SECURE\\" written in a clean font').
2. SIMPLICITY & FOCUS: Describe one clear main subject and a simple background. Do not add too many competing elements or over-explain minor details. 
3. ATTENTION-GRABBING: Use strong visual keywords to create high contrast, vibrant lighting, and an eye-catching composition that matches the style.
4. OUTPUT ONLY: Output only the final expanded image prompt. Do not include any introductory text, explanations, or conversational filler.
```

### What you can change
- **Role:** Change "AI image prompt engineer for FLUX.1" to "Stable Diffusion prompt crafter" if you swap models later.
- **Rule #1 (text blunders):** Keep this unless you’re using a model that handles text well (e.g., DALL·E 3 via OpenRouter — then you might relax it).
- **Rule #2 (simplicity):** Adjust based on your preferred image complexity.
- **Rule #3 (attention-grabbing):** Emphasize different aspects (e.g., "pastel colors" vs "high contrast").
- **Rule #4 (output only):** Keep this — it prevents preamble from breaking the image generation.

> 🔁 If you ever swap the image model (e.g., to Stable Diffusion XL), you may want to revisit this prompt engineering.

---

## ⏰ 5. Change the Posting Schedule

The workflow runs twice daily by default (08:00 and 18:00). Adjust the **Schedule Trigger** node.

### Where to edit
- Node: **Schedule Trigger** (`n8n-nodes-base.scheduleTrigger`)
- Field: **Rules**

### Current setting
```json
{
  "rule": {
    "interval": [
      { "triggerAtHour": 8 },
      { "triggerAtHour": 18 }
    ]
  }
}
```

### Examples
- **Once daily at 09:00:**
  ```json
  { "rule": { "interval": [ { "triggerAtHour": 9 } ] } }
  ```
- **Three times daily (08:00, 12:00, 18:00):**
  ```json
  { "rule": { "interval": [
    { "triggerAtHour": 8 },
    { "triggerAtHour": 12 },
    { "triggerAtHour": 18 }
  ] } }
  ```
- **Weekdays only at 09:00:**
  ```json
  {
    "rule": {
      "interval": [
        { "triggerAtHour": 9, "triggerAtDays": [1,2,3,4,5] } // Mon-Fri
      ]
    }
  }
  ```

> 🕐 **Timezone note:** n8n uses the **instance’s configured timezone**, not necessarily UTC. Check Settings → General → Timezone if your posts fire at unexpected local times.

---

## 🔧 6. Swap the LLMs (OpenRouter Models)

The workflow uses two OpenRouter models:
- **Post Maker:** `deepseek/deepseek-v4-flash` (fast, cheap, good for instruction following)
- **Image Prompt Maker:** `deepseek/deepseek-chat` (stronger reasoning for prompt engineering)

### Where to edit
- Node: **OpenRouter Chat Model** (for Post Maker)
- Node: **OpenRouter Chat Model1** (for Image Prompt Maker)
- Field: **Model**

### Popular alternatives on OpenRouter
| Use Case                  | Model Suggestion                | Notes                                  |
| ------------------------- | ------------------------------- | -------------------------------------- |
| Faster/cheaper post       | `google/gemma-2-9b-it`          | Good if DeepSeek is rate-limited       |
| Better reasoning          | `anthropic/claude-3-haiku`      | More expensive, stronger style following |
| Longer context            | `mistralai/mistral-nemo`        | 128k context (overkill here)           |
| Open weights              | `meta-llama/llama-3-8b-instruct`| Requires routing via a provider        |

> 🔐 Remember: changing the model here **does not** change the credential — you still use the same OpenRouter API key.

---

## 🖼 7. Swap the Image Model (FLUX → Something Else)

The workflow calls Cloudflare Workers AI for FLUX.1 Schnell. To change the model or provider, edit two nodes.

### Nodes to edit
1. **Get Flux Schnell image** (`n8n-nodes-base.httpRequest`)
   - **URL:** change the model identifier in the path
   - Current: `@cf/black-forest-labs/flux-1-schnell`
   - Examples:
     - `@cf/black-forest-labs/flux-1` (standard, slower)
     - `@cf/stabilityai/stable-diffusion-xl-base-1.0` (SDXL)
2. **Setup workflow** (`n8n-nodes-base.set`)
   - Ensure the payload matches what the new model expects (FLUX uses `prompt`, `width`, `height`, `steps`, `seed`; others may differ)

### Example: Switch to SDXL
1. In **Get Flux Schnell image**, change URL to:
   ```
   https://api.cloudflare.com/client/v4/accounts/{{ $json.accountId }}/ai/run/@cf/stabilityai/stable-diffusion-xl-base-1.0
   ```
2. In **Setup workflow**, you may need to adjust:
   - `steps`: SDXL often likes 20–30
   - `width`/`height`: SDXL works best at 1024×1024 or 1152×896, etc.
   - Add `refiner` or other params if the model supports them

> 📚 Consult [Cloudflare Workers AI docs](https://developers.cloudflare.com/workers-ai/models/) for the exact input schema of each model.

---

## 🧹 8. Adjust the Post Trimmer (Output Sanitization)

The **Post Trimmer** node cleans up the LLM’s output before publishing.

### Where to edit
- Node: **Post Trimmer** (`n8n-nodes-base.code`)
- Field: **JavaScript Code**

### What it does (current)
1. Strips `**bold**` and `*italic*` markdown.
2. Removes leading markdown headings (`# `).
3. Strips backticks (```).
4. Caps hashtags at exactly 3 (removes extras from the end).
5. Truncates to ≤ 2900 characters on a word boundary.
6. Throws if result is empty or < 50 chars (quality gate).

### What you might change
- **Hashtag cap:** change `if (hashtags.length > 3)` to `> 2` or `> 4`.
- **Max length:** change `const MAX_LENGTH = 2900;` to match LinkedIn’s actual limit (currently ~3000).
- **Quality gate:** adjust the `post.length < 50` threshold.
- **Additional stripping:** add more `.replace()` lines if you see unwanted patterns.

> ✅ Keep the quality gate — it prevents publishing empty or nonsensical content.

---

## 📊 9. Change the Slack Notification (or Remove It)

The final **Send a message** node posts to Slack.

### Where to edit
- Node: **Send a message** (`n8n-nodes-base.slack`)
- Fields:
  - **Channel:** change `#social` to another channel name
  - **Text:** adjust the message template
    ```
    Topic  : {{ $json.topic }}
    Status : {{ $json.status }}
    error  : {{ $json.error }}
    ```

### To disable Slack entirely
1. Delete the **Send a message** node.
2. Delete its incoming connection from **Log Post**.
3. Save — nothing else breaks.

---

## 🧪 10. Test Your Changes Safely

Always test after editing:

1. **Manual trigger:** Click **Execute Workflow** — watch the execution in real-time.
2. **Check each branch:** Use the **Run Once** feature on individual nodes to verify outputs.
3. **Inspect static data:** In any Code node, you can temporarily add `return [{ json: $getWorkflowStaticData('global') }];` to peek at `usedTopics`, `lastStyle`, etc.
4. **Verify credential connections:** Nodes should show a green check, not a red warning.

> 🔁 Remember: n8n does **not** auto-reload code when you edit a node — you must **save the workflow** for changes to take effect.

---

## 📁 Where to Find Everything

| What you want to change           | Node name (search in n8n)               |
| --------------------------------- | --------------------------------------- |
| Topic list                        | Topic Randomiser                        |
| Style list + descriptions         | Style Randomiser                        |
| Post writing instructions         | Post Maker                              |
| Image prompt instructions         | Image Prompt Maker                      |
| Post cleanup rules                | Post Trimmer                            |
| FLUX model / image settings       | Get Flux Schnell image + Setup workflow |
| Schedule (cron)                   | Schedule Trigger                        |
| LLM models                        | OpenRouter Chat Model / OpenRouter Chat Model1 |
| Slack message                     | Send a message                          |
| LinkedIn person URN               | Create a post                           |

---

## ✅ Quick Reference: Safe Edit Zones

You can safely edit these without breaking the workflow’s logic:

| Node                     | Safe-to-edit fields                                               |
| ------------------------ | ----------------------------------------------------------------- |
| Topic Randomiser         | `topics` array                                                    |
| Style Randomiser         | `styles` array                                                    |
| Post Maker               | Prompt text (keep input/output variable names)                    |
| Image Prompt Maker       | Prompt text (keep input/output variable names)                    |
| Schedule Trigger         | `triggerAtHour` rules                                             |
| OpenRouter Chat Model*   | Model name                                                        |
| Get Flux Schnell image   | URL (model endpoint), ensure method stays POST                    |
| Setup workflow           | JSON payload keys (must match what the image model expects)       |
| Post Trimmer             | Hashtag cap, max length, quality gate thresholds                  |
| Send a message (Slack)   | Channel, message text                                             |
| Create a post            | Person URN, visibility                                            |

> \* Both OpenRouter nodes use the same credential — changing the model here is always safe.

---

## 🎉 You’re Ready

Edit, test, and enjoy your custom LinkedIn auto-poster. Once you’re happy:

1. Activate the workflow to let the schedule run it automatically.
2. Occasionally check the **Executions** tab and the **Slack** channel (if enabled) to ensure it’s healthy.
3. Revisit the **Topic Randomiser** and **Style Randomiser** arrays every few months to keep content fresh.

If you build something cool — a new set of topics for a different niche, a clever prompt tweak, or a fallback image strategy — consider contributing it back!

--- 

*Made with ❤ for the LinkedIn builder community.*