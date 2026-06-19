# Security

> How the workflow handles credentials, what you need to know to keep your secrets safe, and best practices for running this workflow in production.

This document explains:
- Why the committed `workflow.json` is safe to share
- What the original `workflow.jason` contains and why you must **not** publish it
- How n8n stores credentials
- Recommendations for securing your n8n instance
- Steps to take if you suspect a credential leak

---

## 🔐 The Two JSON Files

| File               | Contains                                 | Safe to commit? | Notes |
| ------------------ | ---------------------------------------- | --------------- | ----- |
| `workflow.json`    | **Sanitized** template (credential IDs replaced with placeholders) | ✅ Yes | This is the file you should share, fork, or commit to Git. |
| `workflow.jason`   | Original export with **real credential IDs** | ❌ **No** | Keep this file private. If you accidentally commit it, rotate the affected credentials immediately. |

> The `.jason` extension is a historical typo; the file is still valid JSON.

---

## 🛡 How n8n Stores Credentials

When you create a credential in n8n (e.g., "LinkedIn account"), n8n:
1. Encrypts the secret value (API key, OAuth token, etc.) using an encryption key derived from `N8N_ENCRYPTION_KEY` (set in your n8n environment).
2. Stores the **encrypted blob** in the n8n database (SQLite by default, PostgreSQL if configured).
3. Stores only the **metadata** (name, type, scoped permissions) in plaintext — **never** the raw secret.

When a workflow runs:
- n8n decrypts the credential **in memory** only.
- The decrypted value never touches disk outside of a secure process boundary.
- The workflow nodes receive the secret as if it were a regular variable, but it’s never logged or exposed in the UI.

> 🔑 **Your `N8N_ENCRYPTION_KEY` is the most important secret in your n8n instance.** If it leaks, an attacker with database access could decrypt all credentials. Back it up securely and rotate it if you suspect compromise.

---

## ⚠️ What’s in the Original `workflow.jason`

If you opened `workflow.jason` in a text editor, you’d see sections like this for each credential-using node:

```json
{
  "type": "@n8n/n8n-nodes-langchain.lmChatOpenRouter",
  "position": [480, 752],
  "id": "3e11617a-e3f2-445f-8600-519f78a92b6c",
  "name": "OpenRouter Chat Model",
  "credentials": {
    "openRouterApi": {
      "id": "IkTAcZkrU8QncJM7",   // ← REAL credential ID
      "name": "OpenRouter account"
    }
  },
  ...
}
```

That `"id"` is a reference to a credential stored in your n8n database. If you commit this file to a public repo:
1. Anyone who sees it knows the **internal ID** of your credential.
2. If they also gain access to your n8n instance’s database (e.g., via SSRF, misconfigured permissions, or a backup leak), they could attempt to replay or decrypt the credential.
3. While the ID alone isn’t enough to steal the secret, it reduces the attacker’s workload and should be treated as sensitive.

---

## ✅ What We Did to Sanitize

Before committing `workflow.json`, we:
1. **Replaced every real credential ID** with a clearly marked placeholder:
   - `REPLACE_WITH_LINKEDIN_OAUTH2_ID`
   - `REPLACE_WITH_OPENROUTER_API_ID`
   - `REPLACE_WITH_HTTPBEARER_CF_ID`
   - `REPLACE_WITH_SLACK_API_ID`
2. **Cleared the workflow’s `instanceId`** (a unique fingerprint n8n generates) so the same template can’t be accidentally cross-tracked if imported elsewhere.
3. **Left the credential `name` fields** human-readable (e.g., `"LinkedIn account"`), so importers know what to reconnect.

The result is a workflow that:
- Imports cleanly into any n8n instance.
- Shows red warnings on credential-bearing nodes until you reconnect them.
- Commits zero secrets to version control.

---

## 🔐 Best Practices for Running This Workflow

### 1. Protect your n8n instance
- **If self-hosted:** Put n8n behind a reverse proxy with HTTPS, enable authentication (basic auth, OAuth, or LDAP), and keep it updated.
- **If using n8n.cloud:** Enable two-factor authentication on your account and restrict API access.

### 2. Least-privilege credentials
- **LinkedIn:** Only grant `w_member_social` and `r_liteprofile` scopes — nothing more.
- **OpenRouter:** Use a dedicated API key (not your main account key) and monitor usage.
- **Cloudflare:** Create a token scoped to **Account:Read** + **Workers AI:Run** — no global API keys.
- **Slack:** Use an Incoming Webhook (not a bot token) if you only need to post messages.

### 3. Rotate credentials periodically
- OpenRouter: rotate keys every 90 days.
- Cloudflare: rotate tokens every 180 days.
- LinkedIn: re-authorize the OAuth2 flow if you suspect token leakage (tokens are long-lived but can be revoked from the LinkedIn app page).
- Slack: regenerate the webhook URL if shared unintentionally.

### 4. Monitor usage
- Check OpenRouter and Cloudflare dashboards for unexpected spikes.
- Review LinkedIn’s “Where you’re logged in” section periodically.
- Watch the Slack channel for failure messages — they can hint at misconfigured credentials.

### 5. Treat logs as sensitive
The workflow’s static data (`staticData.postLog`) contains the last 50 runs, including topics and whether they succeeded. While this doesn’t include credentials, it may reveal:
- Your posting schedule
- The topics you cover
- Whether image generation is failing

If you share n8n exports or backups, ensure this static data is cleared (or at least be aware it’s included).

---

## 🚨 If You Accidentally Commit Credentials

If you realize you’ve pushed `workflow.jason` (or a backup) to a public repo:

1. **Do not panic** — the file alone does not reveal the actual secrets.
2. **Rotate the affected credentials anyway**, as a precaution:
   - **OpenRouter:** [Revoke the key](https://openrouter.ai/keys) and create a new one.
   - **Cloudflare:** [Delete the token](https://dash.cloudflare.com/profile/api-tokens) and create a new one.
   - **LinkedIn:** In your LinkedIn app settings → **Auth** → **Deauthorize** (or remove the app entirely and re-add).
   - **Slack:** In your Slack app → **Incoming Webhooks** → click the three dots → **Delete**.
3. **Update the credentials in n8n** with the new values.
4. **Re-import the sanitized `workflow.json`** (or just re-save the workflow after reconnecting credentials) to clear any cached references.
5. **Delete or make private** the commit that leaked the file:
   - On GitHub: use `git filter-repo` or contact support if it’s already public.
   - If a fork exists, assume it’s compromised and rotate regardless.

> 🕒 Act quickly — credential rotation should happen within minutes of discovery, not hours.

---

## 📚 Further Reading

- [n8n Security Documentation](https://docs.n8n.io/security/)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [LinkedIn API Security Best Practices](https://learn.microsoft.com/en-us/linkedin/marketing/integrations/community-management/shares/asset-security?context=linkedin/marketing/context)
- [Cloudflare API Tokens Guide](https://developers.cloudflare.com/api/tokens/create/)

---

## ✅ Summary

- ✅ **Safe to share:** `workflow.json` (sanitized template)
- 🔒 **Keep private:** `workflow.jason` (original export with real credential references)
- 🔑 **Protect the encryption key:** `N8N_ENCRYPTION_KEY` — if it leaks, rotate all credentials.
- 🔐 **Use least-privilege tokens** for each service.
- 🔄 **Rotate credentials** periodically and immediately after any suspected leak.
- 🛡️ **Treat your n8n instance** like any other OAuth-protected service — keep it locked down.

Follow these steps, and your LinkedIn auto-poster will run safely and unattended for months to come.

--- 

*Made with ❤ for the LinkedIn builder community.*