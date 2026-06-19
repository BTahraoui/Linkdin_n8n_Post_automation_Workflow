# Setup Guide

> Step-by-step instructions to configure all external services required by the LinkedIn n8n Auto-Poster workflow.

This guide walks you through creating and configuring the four credentials the workflow depends on:
1. **LinkedIn OAuth2** – for publishing posts
2. **OpenRouter API** – for LLMs (DeepSeek models)
3. **Cloudflare Bearer Token** – for Workers AI (FLUX.1 Schnell)
4. **Slack API** – for run notifications

> ✅ **Tip:** Perform these steps once per n8n instance. Credentials are reusable across workflows.

---

## Prerequisites

- A running n8n instance (self-hosted or n8n.cloud) ≥ version 1.0
- Administrator access to create credentials in n8n
- Access to the four services listed above (free tiers are sufficient)

---

## 1. LinkedIn OAuth2 Credential

The workflow needs permission to create posts on your LinkedIn profile or page.

### Step 1: Create a LinkedIn Developer Application

1. Go to [https://www.linkedin.com/developers/](https://www.linkedin.com/developers/) and sign in.
2. Click **Create application** → **+ Create new app**.
3. Fill in:
   - **App name:** `LinkedIn Auto-Poster (your name)`
   - **LinkedIn Page:** Select the page you want to post to (or your personal profile)
   - **App logo:** Optional
   - **Privacy policy URL:** `https://github.com/your-username/Linkdin_n8n_Post_automation_Workflow#readme` (or any placeholder)
   - **Business email:** Your email
4. Click **Create app**.

### Step 2: Configure OAuth Settings

1. In your new app dashboard, go to **Auth** → **OAuth 2.0**.
2. Set **Authorized redirect URLs** to:
   ```
   https://<your-n8n-domain>/oauth2/credentials
   ```
   - For n8n.cloud: `https://n8n.<your-subdomain>.n8n.cloud/oauth2/credentials`
   - For self-hosted: `https://your-domain.com/oauth2/credentials` (or `http://localhost:5678/oauth2/credentials` for local)
3. Under **OAuth 2.0 scopes**, select:
   - `w_member_social` (required to create posts)
   - `r_liteprofile` (required to get your person URN)
4. Click **Update**.

### Step 3: Get Client ID & Secret

1. Still in the **Auth** tab, note:
   - **Client ID**
   - **Client Secret**
2. Keep this tab open — you’ll need these values in n8n.

### Step 4: Create the Credential in n8n

1. In n8n, go to **Credentials** → **New Credential**.
2. Search for and select **LinkedIn OAuth2 API**.
3. Fill in:
   - **Name:** `LinkedIn account` (must match the name used in the workflow)
   - **Client ID:** *from LinkedIn Developer App*
   - **Client Secret:** *from LinkedIn Developer App*
   - **Grant Type:** `Authorization Code`
   - **Scope:** `w_member_social r_liteprofile` (space-separated)
   - **Auth URI:** `https://www.linkedin.com/oauth/v2/authorization`
   - **Access Token URI:** `https://www.linkedin.com/oauth/v2/accessToken`
   - **Refresh Token URI:** *(leave blank — LinkedIn doesn’t use refresh tokens in this flow)*
4. Click **Save**.

### Step 5: Authorize

1. After saving, click **Access Token** → **Get Access Token**.
2. You’ll be redirected to LinkedIn to log in and authorize the app.
3. Once authorized, n8n will show an access token — you’re done!

> 🔐 **Security note:** The credential stores an encrypted refresh token in your n8n database. Protect your n8n instance accordingly.

---

## 2. OpenRouter API Credential

OpenRouter provides access to dozens of LLMs via a single API key. We use it for:
- `deepseek/deepseek-v4-flash` (post writer)
- `deepseek/deepseek-chat` (image prompt engineer)

### Step 1: Sign Up & Get API Key

1. Go to [https://openrouter.ai/](https://openrouter.ai/) and sign up (Google/GitHub works).
2. Once logged in, go to [https://openrouter.ai/keys](https://openrouter.ai/keys).
3. Click **Create Key**.
4. Name it something like `n8n-linkedin-poster` and click **Create**.
5. **Copy the key immediately** — you won’t see it again.

### Step 2: Create the Credential in n8n

1. In n8n, go to **Credentials** → **New Credential**.
2. Search for and select **OpenRouter API**.
3. Fill in:
   - **Name:** `OpenRouter account` (must match the workflow)
   - **API Key:** *paste the key from OpenRouter*
4. Click **Save**.

> 💡 **Free tier note:** OpenRouter offers generous free limits. DeepSeek models are currently among the most cost-effective options available.

---

## 3. Cloudflare Bearer Token (HTTP Bearer Auth)

Used to call Cloudflare Workers AI for FLUX.1 Schnell image generation.

### Step 1: Get a Cloudflare API Token

1. Log in to [https://dash.cloudflare.com/](https://dash.cloudflare.com/).
2. In the top-right profile menu → **My Profile**.
3. Scroll down to **API Tokens** → **Create Token**.
4. Click **Create Custom Token**.
5. Configure:
   - **Token name:** `Cloudflare Workers AI for n8n`
   - **Permissions:**
     - **Account Settings** → **Account:Read**
     - **Workers AI** → **Workers AI:Run**
   - **Account Resources:** Select your account (All accounts if you have one)
6. Click **Continue to summary** → **Create Token**.
7. **Copy the token immediately** — you won’t see it again.

### Step 2: Create the Credential in n8n

1. In n8n, go to **Credentials** → **New Credential**.
2. Search for and select **HTTP Bearer Auth** (under *Authentication*).
3. Fill in:
   - **Name:** `Bearer Auth account` (must match the workflow)
   - **Token Type:** `Bearer Token`
   - **Token:** *paste the Cloudflare API Token*
4. Click **Save**.

> ⚙️ **Note:** This token grants read access to your account and permission to run Workers AI models. Restrict it to your account only as shown above.

---

## 4. Slack API Credential (Optional but Recommended)

Used to send a notification to a Slack channel after each run (topic + status + any error).

### Step 1: Create a Slack App

1. Go to [https://api.slack.com/apps](https://api.slack.com/apps) and click **Create New App**.
2. Choose **From scratch**.
3. App name: `LinkedIn Post Notifier`
4. Workspace: Select your workspace → **Create App**.

### Step 2: Add Incoming Webhooks

1. From the app dashboard, select **Incoming Webhooks**.
2. Toggle **Activate Incoming Webhooks** to **On**.
3. Scroll down and click **Add New Webhook to Workspace**.
4. Choose the channel where you want notifications (e.g., `#social`) → **Allow**.
5. Copy the **Webhook URL** that looks like:
   ```
   https://hooks.example.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX
   ```

### Step 3: Create the Credential in n8n

1. In n8n, go to **Credentials** → **New Credential**.
2. Search for and select **Slack API**.
3. Fill in:
   - **Name:** `Slack account` (must match the workflow)
   - **Access Token:** `xoxb-` *or* use **Webhook URL** (n8n Slack node supports both; we use Webhook in this workflow)
   - If using Webhook URL: paste the full URL from Slack into the **Webhook URL** field
4. Click **Save**.

> 📣 **No Slack?** If you don’t want Slack notifications, simply delete the final **Send a message** node from the workflow. Everything else will continue to work.

---

## Verifying Your Setup

Once all four credentials are created:

1. Import `workflow.json` (sanitized) or `workflow.jason` (your original) into n8n.
2. Open each credential-bearing node and verify the credential name matches exactly:
   - `LinkedIn account`
   - `OpenRouter account`
   - `Bearer Auth account`
   - `Slack account`
3. If names don’t match, re-select the correct credential from the dropdown — n8n will update the internal ID automatically.
4. Save the workflow.
5. Click **Execute Workflow** to test the full pipeline.

You should see:
- A topic picked from the list
- A style chosen
- A post generated
- An image generated (or fallback to text-only)
- A LinkedIn post published (or error in the log)
- A Slack notification sent (if configured)

Check the **Executions** list in n8n for details.

---

## Troubleshooting Common Issues

| Symptom | Likely Cause | Fix |
| ------- | ------------ | --- |
| `401 Unauthorized` from LinkedIn | Expired or missing credential | Re-authorize via **Credentials** → **LinkedIn account** → **Access Token** → **Get Access Token** |
| `401 Unauthorized` from OpenRouter | Invalid API key | Regenerate key at [https://openrouter.ai/keys](https://openrouter.ai/keys) and update credential |
| `401 Unauthorized` from Cloudflare | Invalid or scoped token | Create new token with **Account:Read** + **Workers AI:Run** permissions |
| No Slack message | Wrong webhook URL or inactive app | Recreate webhook in Slack app settings and update credential |
| Workflow fails at `Image > 1 megapixel` | Width × height > 1,048,576 | Reduce dimensions in `Setup workflow` node (try 1024×1024) |

---

## Next Steps

- See [docs/CUSTOMIZATION.md](docs/CUSTOMIZATION.md) to tailor topics, styles, prompts, or schedule.
- See [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) for a deep dive into every node.
- See [docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md) for error-specific fixes.

Once everything is green, activate the workflow to let the schedule (08:00 & 18:00) take over — your LinkedIn is now on autopilot!