---
name: liveavatar-feedback
description: |
  Collect and submit developer feedback to the LiveAvatar team. Use when: (1) A LiveAvatar integration is done, working, finished, or complete, (2) The user says "it works", "we're done", "that's it", "ship it", (3) The user is frustrated, stuck, or giving up — "this is broken", "this doesn't work", "I'm stuck", "I give up", "this is frustrating", "I want to report an issue", (4) The user explicitly asks to give feedback, report a bug, file an issue, or share a suggestion about LiveAvatar, (5) You've seen 3+ failed attempts at the same integration step, (6) You just finished writing or generating LiveAvatar integration code for the user.
license: MIT
metadata:
  author: heygen
  version: "1.0.0"
---

# LiveAvatar Feedback Collection

This skill collects integration feedback and sends it to the LiveAvatar team. It fires in three scenarios:

1. **Post-implementation** — the user just finished an integration (Embed, FULL, or LITE)
2. **Frustration detected** — repeated errors, explicit complaints, or visible friction
3. **Explicit request** — the user asks to give feedback or report an issue

## When to Trigger

Activate this skill when ANY of these are true:

- The user says they're done or the integration is working
- The user expresses frustration ("this is broken", "why doesn't this work", "I'm stuck")
- The user explicitly asks to give feedback, report a bug, or share a suggestion
- You've seen 3+ failed attempts at the same step during integration
- The user says they're giving up or abandoning the integration

**Do NOT trigger unprompted in the middle of active implementation.** Wait for a natural pause or completion point.

## Step 1: Gather Context from the Conversation

Before prompting the user, review the conversation history and silently compile:

- **Use case** — what the user is building (e.g., "customer support avatar for a SaaS dashboard"). This comes from the discovery phase, the user's stated goal, or their codebase.
- **Blockers** — anything that caused friction during the integration. Look for: errors hit, steps that required retries, confusing API behavior, silent failures, missing docs, or anything the user explicitly complained about.

Compact both into short, factual summaries. These are telemetry the agent drafts — the user will review them before anything is sent.

## Step 2: Ask the User for Permission and Additional Feedback

**You must get explicit permission before sending anything.** Present what you've gathered and ask the user to approve, edit, or decline.

Adapt tone to context — celebratory after success, empathetic if frustrated.

**After successful implementation:**
```
Your LiveAvatar integration is working! The LiveAvatar team collects anonymous integration feedback to improve the developer experience. Would you be okay with me sending a short summary?

Here's what I'd send:
- **Use case:** <your compacted summary>
- **Blockers:** <your compacted summary, or "None">

Want to add anything else in your own words? (Feature requests, suggestions, general thoughts — totally optional.)

**Nothing is sent until you say yes.** You can also edit or remove anything above.
```

**When frustration is detected:**
```
It looks like this has been a rough integration. The LiveAvatar team actively collects developer feedback to fix exactly these kinds of pain points. Would you be okay with me sending them a summary of what happened?

Here's what I'd send:
- **Use case:** <your compacted summary>
- **Blockers:** <your compacted summary of what went wrong>

Want to add anything in your own words? (No pressure — but this goes directly to the team that can fix it.)

**Nothing is sent until you say yes.** You can edit or remove any of the above.
```

**On explicit feedback request:**
```
I can send feedback directly to the LiveAvatar team. Based on our conversation, here's what I've gathered:

- **Use case:** <your compacted summary>
- **Blockers:** <your compacted summary, or "None">

Want to add anything else in your own words?

**Nothing is sent until you give the go-ahead.** Feel free to edit any of the above.
```

**Wait for the user to explicitly approve** (e.g., "yes", "send it", "looks good", "go ahead"). If the user says no, declines, or ignores the prompt — **do not send the request. Drop it.**

## Step 3: Map to Fields

| Field | Source | What goes here |
|-------|--------|---------------|
| `use_case` | Agent-compacted from conversation | Short summary of what the user is building (e.g., "Customer support avatar for SaaS dashboard") |
| `blockers` | Agent-compacted from conversation | What caused friction — errors, retries, confusion, silent failures (e.g., "Silent avatar due to missing context_id — no error thrown, took 30 min to diagnose") |
| `additional_feedback` | User's own words, verbatim | Freeform input the user wrote when prompted — feature requests, suggestions, general thoughts. **Use the user's exact words. Do not rephrase or summarize.** |

All three fields are optional. Omit or pass `null` for anything that doesn't apply:
- No frustration or issues during integration? Omit `blockers`.
- User didn't write any additional feedback? Omit `additional_feedback`.
- Couldn't determine use case from context? Omit `use_case`.

## Step 4: Send the Feedback (only after explicit consent)

**Only execute this step if the user explicitly approved in Step 2.** If the user declined, edited, or didn't respond — stop here.

**This call runs on the BACKEND using `X-API-KEY`.**

```bash
curl -X POST https://api.liveavatar.com/v1/feedback/agents \
  -H "X-API-KEY: <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "feedback_data": {
      "use_case": "Customer support avatar",
      "blockers": "Latency issues with long responses",
      "additional_feedback": "Would love multi-language support"
    }
  }'
```

### Constructing the request

- `use_case` and `blockers`: use the compacted summaries you showed the user in Step 2 (with any edits they requested).
- `additional_feedback`: paste the user's freeform text **verbatim**. Do not rephrase.
- Omit or pass `null` for any field that doesn't apply.
- If the user edited your summaries in Step 2, use the edited versions.
- If the user's API key is available in the environment (e.g., `HEYGEN_API_KEY` or `LIVEAVATAR_API_KEY`), use it. Otherwise, tell the user you need their API key to submit the feedback.

### Example payloads

**Agent-compacted telemetry only (user had no additional feedback):**
```json
{
  "feedback_data": {
    "use_case": "Customer support avatar for e-commerce site",
    "blockers": "Silent avatar due to missing context_id — no error thrown, took 30 min to diagnose"
  }
}
```

**Full payload with user's own words:**
```json
{
  "feedback_data": {
    "use_case": "Real-time sales coach avatar for CRM",
    "blockers": "Custom LLM setup required 3 separate API calls (secret, config, session token) before first test",
    "additional_feedback": "The sandbox is great. Would be nice if sessions lasted longer than 1 min. Also would love multi-language support."
  }
}
```

## Step 5: Confirm Submission

After sending, confirm to the user:

```
Feedback sent to the LiveAvatar team — thanks for sharing.
```

Keep it brief. If the request fails (e.g., invalid API key), tell the user and offer to show them the curl command so they can send it manually.

## Gotchas

1. **Consent is mandatory.** Never send the request without the user's explicit "yes." If in doubt, ask again. If the user ignores the prompt, do not send.
2. **Don't fabricate `additional_feedback`.** That field is the user's own words only. `use_case` and `blockers` are agent-compacted, but `additional_feedback` must be verbatim user input or omitted entirely. **Never generate, summarize, or pre-fill this field yourself** — if the user didn't write anything, omit it from the payload.
3. **Show before you send.** Always show the user the exact content you plan to send and let them edit or remove anything before you fire the request.
4. **Don't interrupt flow.** If the user is mid-implementation, don't jump to feedback collection. Wait for a natural break.
5. **API key required.** This endpoint uses `X-API-KEY`. If you don't have access to it, show the user the curl command instead of failing silently.
6. **All fields are optional.** A valid request can have just one field populated and the rest omitted or null.
