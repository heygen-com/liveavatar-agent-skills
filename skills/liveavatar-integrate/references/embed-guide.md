# Embed Implementation Guide

One API call, one iframe. No SDK, no WebRTC, no event handling.

## Prerequisites

- LiveAvatar API key (get one at https://app.liveavatar.com)
- An `avatar_id` — browse on the dashboard or `GET /v1/avatars`
- A `context_id` — create via `POST /v1/contexts` with a `prompt` field

## Implementation

### 1. Create the embed (BACKEND — `X-API-KEY`)

```bash
curl -X POST https://api.liveavatar.com/v2/embeddings \
  -H "X-API-KEY: <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "avatar_id": "<avatar_id>",
    "context_id": "<context_id>"
  }'
```

Response:
```json
{
  "code": 1000,
  "data": {
    "url": "https://embed.liveavatar.com/v1/<id>",
    "script": "<iframe src=\"https://embed.liveavatar.com/v1/<id>\" allow=\"microphone\" title=\"LiveAvatar Embed\" style=\"aspect-ratio: 16/9;\"></iframe>"
  }
}
```

### 2. Drop into HTML (FRONTEND)

```html
<iframe
  src="https://embed.liveavatar.com/v1/<id>"
  allow="microphone"
  title="LiveAvatar Embed"
  style="aspect-ratio: 16/9; width: 100%; border: none;"
></iframe>
```

Done.

## Sandbox testing

Use `"is_sandbox": true` with avatar ID `65f9e3c9-d48b-4118-b73a-4ae2e3cbb8f0`:

```json
{
  "avatar_id": "65f9e3c9-d48b-4118-b73a-4ae2e3cbb8f0",
  "context_id": "<context_id>",
  "is_sandbox": true
}
```

No credits consumed. Note: embed sandbox avatar ID (`65f9e3c9-...`) differs from session sandbox avatar ID (`dd73ea75-...`).

## Gotchas

1. **`allow="microphone"` is required on the iframe.** Without it, the browser blocks mic access and the avatar can't hear.
2. **API key must stay on the backend.** The iframe URL is safe for clients; the API call that creates it is not.
3. **No `context_id` = non-conversational avatar.** The avatar may not respond to speech without a context.
