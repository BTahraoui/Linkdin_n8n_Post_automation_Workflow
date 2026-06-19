# Troubleshooting

> Common issues you might encounter with the LinkedIn n8n Auto-Poster workflow and how to fix them.

This guide covers:
- Execution failures and where to look for clues
- Credential and permission problems
- Image generation hiccups
- LinkedIn posting errors
- Slack notification issues
- Static data and state-related quirks

Always start by checking the **Execution Details** in n8n — click on a failed (or successful) execution to see each node’s input/output and error messages.

---

## 🚦 1. Workflow Doesn’t Trigger

### Symptom
- Schedule Trigger doesn’t fire at the expected time
- Manual “Execute Workflow” does nothing

### Checks
1. **Is the workflow activated?**
   - In the workflow list, the toggle must be **ON** (activated). Saved ≠ activated.
2. **Schedule Trigger timezone**
   - n8n uses the instance’s configured timezone (Settings → General → Timezone).
   - If you set `triggerAtHour: 9` but your n8n runs in UTC, it’ll fire at 09:00 UTC — not necessarily your local time.
3. **Manual trigger blocked?**
   - Rare, but if another execution is already running and you’ve set “Execute Workflow” to “Wait for Last Execution” in Settings, you’ll need to wait or cancel the current run.
4. **Server time drift**
   - If self-hosted, ensure the host clock is synced (via NTP).

### Fix
- Activate the workflow.
- Adjust `triggerAtHour` in the Schedule Trigger node to match your desired **local time** after accounting for n8n’s timezone.
- For testing, use the Manual Trigger — it ignores the schedule.

---

## 🔐 2. Credential Errors (401 Unauthorized)

### Symptom
- Error message: `401 Unauthorized` or `Invalid credentials`
- Fails at: LinkedIn node, OpenRouter node, Cloudflare node, or Slack node

### Most common causes
| Node | Likely cause |
| ---- | ------------ |
| LinkedIn | Expired OAuth2 token or missing scopes (`w_member_social`, `r_liteprofile`) |
| OpenRouter | Invalid or missing API key |
| Cloudflare | Invalid API token or insufficient permissions (needs `Account:Read` + `Workers AI:Run`) |
| Slack | Invalid webhook URL or the Slack app was reinstalled/revoked |

### Fix
1. Open the **Credential** in n8n (not the workflow node).
2. For OAuth2 (LinkedIn): click **Access Token** → **Get Access Token** to re-authorize.
3. For API keys (OpenRouter, Cloudflare, Slack): edit the credential and paste the new value.
4. Save the credential.
5. Re-run the workflow.

> 💡 Tip: Give each credential a clear name and rotate them every 90 days to avoid surprises.

---

## 🖼 3. Image Generation Fails

### Symptom
- Workflow stops at `Image > 1 megapixel` (error node)
- Or fails at `Get Flux Schnell image` with timeout or non-2xx response
- Falls back to text-only post every time (check Slack or logs)

### Cause 1: Image too large (`Image > 1 megapixel` triggers)
- **Why:** Width × height > 1,048,576 pixels (e.g., 1200×1200 = 1.44 MP)
- **Fix:** In the **Setup workflow** node, reduce `width` and/or `height`.  
  Try 1024×1024 (1.048 MP — technically over, but Cloudflare often allows it) or 1024×960 (0.983 MP).

### Cause 2: Cloudflare account not resolved
- **Why:** The `Get accounts` node returned empty or error.
- **Check:** Was the **Bearer Auth account** credential set correctly? Does the token have `Account:Read` scope?
- **Fix:** Recreate the Cloudflare API token with the correct permissions.

### Cause 3: FLUX model error or rate limit
- **Why:** Cloudflare Workers AI returned an error (e.g., model overloaded, invalid prompt).
- **Check:** In the execution, look at the output of `Get Flux Schnell image` — does it contain an `error` field?
- **Fix:**
  1. Wait a few minutes and retry (rate limits are often temporary).
  2. Shorten the prompt in **Image Prompt Maker** — very long prompts can fail.
  3. Ensure the prompt doesn’t contain confusing characters (though the agent is instructed to avoid them).
  4. Temporarily switch to a different model (see [Customization Guide](docs/CUSTOMIZATION.md#7-swap-the-image-model-flux---something-else)).

### Cause 4: Workflow continues after image failure but no Slack message?
- The `Image OK?` node routes to `Create text-only post` on failure — you should still get a Slack message saying `Status: failed`. If you don’t, check the Slack credential.

---

## 💼 4. LinkedIn Posting Fails

### Symptom
- Slack message shows `Status: failed` and an error from the LinkedIn node
- Post never appears on your profile/page

### Common LinkedIn errors
| Error | Meaning | Fix |
| ----- | ------- | --- |
| `401 Unauthorized` | Token missing, expired, or revoked | Re-authorize the LinkedIn credential |
| `403 Forbidden` | Token lacks required scopes | Ensure credential has `w_member_social` and `r_liteprofile` |
| `429 Too Many Requests` | Rate limited | Wait 10–30 minutes; LinkedIn allows ~100 posts/day per user |
| `400 Bad Request` | Malformed request (rare) | Check that the post text isn’t empty and is < 3000 characters |
| `Internal Server Error` | LinkedIn transient issue | Retry later |

### Why posts might be rejected silently
Even if the node returns success, LinkedIn may:
- Not show the post immediately (allow a few minutes for propagation).
- Flag it as spam if you post identical content too often (the workflow’s anti-repeat helps, but still vary your topics).
- Require a profile picture or completed profile for new accounts.

### Fix
1. Re-authorize LinkedIn credential if you see 401/403.
2. Wait and retry if you see 429.
3. Edit the **Post Trimmer** node if you suspect the text is too long or empty.
4. Test with the Manual Trigger to see the exact LinkedIn response.

---

## 💬 5. Slack Notifications Not Sending

### Symptom
- No message appears in the Slack channel, even when the workflow runs successfully.

### Checks
1. **Is the Slack credential configured?**
   - Open the credential — is the Webhook URL field filled?
   - Did you copy the full URL from Slack (including `https://hooks.slack.com/services/...`)?
2. **Is the channel correct?**
   - The workflow uses `select: "channel"` and looks for a channel named `#social` (or whatever you set).
   - If you renamed or deleted the channel, update the node.
3. **Did the Slack app get uninstalled?**
   - Go to your Slack workspace → Manage Apps → look for the Incoming Webhook app.
   - If missing, recreate the webhook and update the credential.
4. **Is the workflow actually reaching the node?**
   - In an execution, check if `Log Post` ran — if yes, the Slack node should follow.
   - If the workflow failed earlier (e.g., LinkedIn error with `continueOnFail: true`), it still should reach Slack unless `Log Post` itself crashed.

### Fix
- Recreate the webhook in Slack if needed and update the credential.
- Test the credential independently: in n8n, create a temporary workflow with just a Slack node and a Set node to populate the message.

---

## 🧠 6. Static Data / State Quirks

### Symptom
- Topics repeat too soon (you saw the same topic twice in a row)
- Styles repeat
- The workflow “forgets” the Cloudflare account ID and calls `Get accounts` every run

### How static data works
The workflow uses `$getWorkflowStaticData('global')` to persist:
- `usedTopics`: array of last 7 topics
- `lastStyle`: string of the last style used
- `cfAccountId`: string (after first successful lookup)
- `postLog`: array of last 50 log entries

This data lives **only** as long as the workflow instance exists in n8n. If you:
- Delete and re-import the workflow
- Rename the workflow
- Move the workflow to another folder
- Instance restart (rarely clears it, but possible)

…then static data resets to empty.

### Fixes
- **Topics/styles repeating too soon:** This is expected if you have fewer than ~15 topics or styles and run frequently. Add more entries to the arrays in **Topic Randomiser** and **Style Randomiser**.
- **Want true randomness without anti-repeat?** Remove the filtering logic in those Code nodes (but you’ll lose the spacing guarantee).
- **Cloudflare account lookup every run:** This means `staticData.cfAccountId` is being cleared. Check:
  1. Are you importing/exporting the workflow frequently? Each import creates a new instance.
  2. Is `n8n` being restarted or wiped (e.g., in a container with non-persistent storage)?
  3. Fix: make your n8n storage persistent (volume mount for self-hosted, or use n8n.cloud).

### To manually inspect static data
Add a temporary Code node anywhere:
```javascript
return [{ json: $getWorkflowStaticData('global') }];
```
Run the workflow and check the output — you’ll see the current state.

---

## 📝 7. Post Quality Issues

### Symptom
- Posts look spammy, off-topic, or violate the rules (no emoji, wrong length, etc.)

### Where the rules live
- **Post Maker** node: defines what the LLM should do.
- **Post Trimmer** node: cleans up what the LLM actually did.

### Common mismatches
| Issue | Likely cause | Fix |
| ----- | ------------ | --- |
| No emoji at start | LLM ignored the “Hook” instruction | Strengthen the prompt in Post Maker: add “You MUST start with an emoji.” |
| Too long (>150 words) | LLM exceeded length | Reduce the limit in the prompt or trust Post Trimmer to cut it (but it may cut awkwardly). |
| No bullet points | LLM skipped formatting | Add “Use bullet points for the main content.” to the prompt. |
| Hashtags not exactly 3 | Post Trimmer logic or LLM output | Verify the JavaScript in Post Trimmer; ensure it runs after the LLM. |
| Bold/italic still present | Post Trimmer regex missed edge cases | Add more `.replace()` patterns (e.g., for `__underline__`). |

### Fix
Edit the **Post Maker** prompt to be more prescriptive, then adjust **Post Trimmer** if needed. Test with Manual Trigger until the output looks good.

---

## 🔄 8. Workflow Seems “Stuck” or Slow

### Symptom
- Executions take >2 minutes to complete
- Node timeouts appear in logs

### Likely bottlenecks
1. **Cloudflare Workers AI cold start** (~1–2s first call after idle)
2. **OpenRouter rate limiting** (if you’re on a free tier and burst)
3. **LinkedIn API slowness** (rare)
4. **Static data growth** (if `postLog` isn’t capped — but it is at 50)

### Fix
- Add a tiny delay (`Wait` node) after `Get Flux Schnell image` if you see consistent timeouts.
- Monitor your OpenRouter and Cloudflare usage dashboards.
- Ensure your n8n instance has enough resources (if self-hosted: 1GB+ RAM, 2+ CPU cores recommended for concurrent workflows).

---

## 📋 9. Empty or Failed Executions with No Clear Error

### Symptom
- Execution shows “Success” but no LinkedIn post and no Slack message
- Or shows “Failed” but the error node didn’t catch it

### Checks
1. **Did the workflow actually run all nodes?**
   - Click the execution → look at the “Nodes executed” list. If it stopped early, see where.
2. **Is there a silent `continueOnFail` swallowing errors?**
   - The LinkedIn nodes have `continueOnFail: true` — if they fail, the workflow continues to `Log Post`.
   - Check `Log Post`’s output: did it record `status: failed`?
3. **Is static data full or corrupted?**
   - Extremely unlikely, but you can test by adding a temporary node that clears static data:
     ```javascript
     $workflow.staticData.set('global', {});
     return [{ json: { cleared: true } }];
     ```

### Fix
- Re-import the sanitized `workflow.json` to start with a clean instance (your credentials will stay intact).
- If the issue persists, check n8n’s own logs (if self-hosted) for crashes.

---

## 🧪 10. Testing Changes Safely

When you edit a node:
1. **Save the workflow** — n8n does not hot-reload code.
2. **Use Manual Trigger** to test immediately.
3. **Check each node’s output** in the execution details.
4. **Only activate** once you’re confident.

> 🛑 Never edit an active workflow and assume changes are live — you must save first.

---

## 📞 Where to Get Help

If you’ve checked this guide and still have issues:

1. **n8n Community Forum**: https://community.n8n.io/
   - Tag your question with `n8n` and `linkedin` or `openrouter` as needed.
2. **GitHub Issues**: If you suspect a bug in the workflow itself, open an issue here (please check if it’s already reported).
3. **Service-specific status pages**:
   - OpenRouter: https://status.openrouter.ai
   - Cloudflare Workers AI: https://www.cloudflarestatus.com/
   - LinkedIn: https://www.linkedin.com/help/linkedin/answer/630
   - Slack: https://status.slack.com/

When asking for help, include:
- The **exact error message** (copy-paste from the execution log)
- Which **node** failed
- What you **tried** to fix it
- Your **n8n version** (Help → About)
- Whether you’re using **self-hosted** or **n8n.cloud**

---

## ✅ Quick Reference: Symptom → Fix Table

| Symptom | First thing to check |
| ------- | -------------------- |
| No trigger fire | Is workflow activated? Is timezone correct? |
| 401 Unauthorized | Re-authorize or rotate the API key for that service |
| Image > 1 megapixel error | Reduce width/height in Setup workflow (try 1024×960) |
| Falls back to text-only every time | Check Cloudflare token and FLUX model status |
| LinkedIn 403 Forbidden | Ensure credential has `w_member_social` + `r_liteprofile` |
| No Slack message | Is webhook URL set? Is channel name correct? |
| Topics repeating too soon | Add more entries to Topic Randomiser’s array |
| Styles repeating | Add more to Style Randomiser’s array |
| Workflow slow on first run after idle | Normal — Cloudflare/OpenRouter cold start |
| Post looks wrong (no emoji, long, etc.) | Edit Post Maker prompt, then Post Trimmer if needed |
| Saved changes not taking effect | Did you **save the workflow** after editing? |

---

*Made with ❤ for the LinkedIn builder community. If this guide helped you unstick, consider dropping a ⭐ on the repo.*