---
description: "Discord REST API v10 reference — auth, messaging, file upload, rate limits, and channel/thread targeting for mid-run API calls"
---

# Knowledge: Discord REST API

Reference for making direct Discord REST API calls during an orbh session. Use this when you need to interact with Discord programmatically — sending mid-run progress messages, querying message history, uploading files, or reading channel state.

All examples use the env vars the Discord adapter injects into your session. See [[dev-init-odisc]] for the full env-var list.

## Auth and Base URL

Every request requires a bot token in the `Authorization` header.

| Component | Value |
|-----------|-------|
| Base URL | `https://discord.com/api/v10` |
| Auth header | `Authorization: Bot $DISCORD_BOT_TOKEN` |
| Content-Type | `application/json` (unless multipart) |

The token comes from `$DISCORD_BOT_TOKEN`, injected by the adapter at session launch. Never hard-code tokens.

```bash
# Verify auth — returns the bot's own user object
curl -s -H "Authorization: Bot $DISCORD_BOT_TOKEN" \
  https://discord.com/api/v10/users/@me
```

## Channel and Thread Targeting

The adapter injects two channel-related env vars:

| Env var | Set when | Use |
|---------|----------|-----|
| `$DISCORD_CHANNEL_ID` | Always | The parent channel |
| `$DISCORD_THREAD_ID` | Session was invoked in a thread | The thread within the channel |

**Rule: prefer `$DISCORD_THREAD_ID` when set; fall back to `$DISCORD_CHANNEL_ID`.**

All endpoint URLs take a single channel ID parameter. Threads are channels in the Discord API — a thread ID works anywhere a channel ID does.

Use this pattern in scripts:

```bash
TARGET="${DISCORD_THREAD_ID:-$DISCORD_CHANNEL_ID}"
```

Every example below uses `$TARGET` for the channel parameter, assuming the above assignment.

## Send a Message

**Endpoint:** `POST /channels/{channel.id}/messages`

### Plain Content

```bash
TARGET="${DISCORD_THREAD_ID:-$DISCORD_CHANNEL_ID}"

curl -s -X POST \
  -H "Authorization: Bot $DISCORD_BOT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"content": "Build passed — all 42 tests green."}' \
  "https://discord.com/api/v10/channels/$TARGET/messages"
```

Content is limited to 2000 characters. The API returns HTTP 400 if exceeded.

### With Embeds

Embeds are rich-formatted blocks with optional title, description, color, and fields.

```bash
TARGET="${DISCORD_THREAD_ID:-$DISCORD_CHANNEL_ID}"

curl -s -X POST \
  -H "Authorization: Bot $DISCORD_BOT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "embeds": [{
      "title": "Deployment Status",
      "description": "**Service:** api-gateway\n**Environment:** staging\n**Commit:** `a1b2c3d`",
      "color": 3066993,
      "fields": [
        {"name": "Duration", "value": "2m 14s", "inline": true},
        {"name": "Status", "value": "Success", "inline": true}
      ]
    }]
  }' \
  "https://discord.com/api/v10/channels/$TARGET/messages"
```

### Embed Limits

| Field | Max length |
|-------|-----------|
| `title` | 256 characters |
| `description` | 4096 characters |
| `fields` | 25 fields per embed |
| `fields[].name` | 256 characters |
| `fields[].value` | 1024 characters |
| `footer.text` | 2048 characters |
| Total characters | 6000 across all embed text |
| Embeds per message | 10 |

### Color Values

Color is a decimal integer. Common values:

| Color | Decimal | Hex |
|-------|---------|-----|
| Green | `3066993` | `#2ECC71` |
| Red | `15158332` | `#E74C3C` |
| Blue | `3447003` | `#3498DB` |
| Yellow | `16776960` | `#FFFF00` |
| Grey | `9807270` | `#95A5A6` |

### Combining Content and Embeds

A single message can carry both `content` (plain text above the embed) and `embeds`:

```bash
TARGET="${DISCORD_THREAD_ID:-$DISCORD_CHANNEL_ID}"

curl -s -X POST \
  -H "Authorization: Bot $DISCORD_BOT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "content": "Here is the latest report:",
    "embeds": [{
      "title": "Weekly Summary",
      "description": "3 tasks completed, 1 in review, 0 blocked.",
      "color": 3066993
    }]
  }' \
  "https://discord.com/api/v10/channels/$TARGET/messages"
```

## Query Messages

**Endpoint:** `GET /channels/{channel.id}/messages`

Returns an array of message objects, newest first by default.

### Query Parameters

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `limit` | integer | 50 | Number of messages to return (1–100) |
| `before` | snowflake | — | Get messages before this message ID |
| `after` | snowflake | — | Get messages after this message ID |
| `around` | snowflake | — | Get messages around this message ID |

Only one of `before`, `after`, or `around` can be used per request.

### Get Recent Messages

```bash
TARGET="${DISCORD_THREAD_ID:-$DISCORD_CHANNEL_ID}"

curl -s \
  -H "Authorization: Bot $DISCORD_BOT_TOKEN" \
  "https://discord.com/api/v10/channels/$TARGET/messages?limit=10"
```

### Paginate Backwards

To walk backwards through history, use `before` with the ID of the oldest message from the previous page:

```bash
TARGET="${DISCORD_THREAD_ID:-$DISCORD_CHANNEL_ID}"

# Page 1
curl -s \
  -H "Authorization: Bot $DISCORD_BOT_TOKEN" \
  "https://discord.com/api/v10/channels/$TARGET/messages?limit=100"

# Page 2 — use the ID of the last (oldest) message from page 1
curl -s \
  -H "Authorization: Bot $DISCORD_BOT_TOKEN" \
  "https://discord.com/api/v10/channels/$TARGET/messages?limit=100&before=OLDEST_MESSAGE_ID"
```

### Get Messages After a Known Point

```bash
TARGET="${DISCORD_THREAD_ID:-$DISCORD_CHANNEL_ID}"

curl -s \
  -H "Authorization: Bot $DISCORD_BOT_TOKEN" \
  "https://discord.com/api/v10/channels/$TARGET/messages?after=KNOWN_MESSAGE_ID&limit=50"
```

Messages returned with `after` are sorted oldest-first.

## Upload a File

**Endpoint:** `POST /channels/{channel.id}/messages` (multipart/form-data)

File uploads use multipart encoding. The `payload_json` form field carries the message JSON; file fields carry the binary data.

### Single File with Message

```bash
TARGET="${DISCORD_THREAD_ID:-$DISCORD_CHANNEL_ID}"

curl -s -X POST \
  -H "Authorization: Bot $DISCORD_BOT_TOKEN" \
  -F 'payload_json={"content": "Here is the build log."}' \
  -F "files[0]=@/tmp/build.log;filename=build.log" \
  "https://discord.com/api/v10/channels/$TARGET/messages"
```

### Image Upload (Displays Inline)

PNG, JPG, GIF, and WebP images display inline in the message. Other file types appear as download links.

```bash
TARGET="${DISCORD_THREAD_ID:-$DISCORD_CHANNEL_ID}"

curl -s -X POST \
  -H "Authorization: Bot $DISCORD_BOT_TOKEN" \
  -F 'payload_json={"content": "Screenshot of the dashboard:"}' \
  -F "files[0]=@/tmp/screenshot.png;filename=screenshot.png" \
  "https://discord.com/api/v10/channels/$TARGET/messages"
```

### Multiple Files

Index the `files[]` field starting from 0:

```bash
TARGET="${DISCORD_THREAD_ID:-$DISCORD_CHANNEL_ID}"

curl -s -X POST \
  -H "Authorization: Bot $DISCORD_BOT_TOKEN" \
  -F 'payload_json={"content": "Attached: logs and config."}' \
  -F "files[0]=@/tmp/app.log;filename=app.log" \
  -F "files[1]=@/tmp/config.toml;filename=config.toml" \
  "https://discord.com/api/v10/channels/$TARGET/messages"
```

### File with Embed

Combine `payload_json` with embeds and file attachments in a single request:

```bash
TARGET="${DISCORD_THREAD_ID:-$DISCORD_CHANNEL_ID}"

curl -s -X POST \
  -H "Authorization: Bot $DISCORD_BOT_TOKEN" \
  -F 'payload_json={"embeds": [{"title": "Test Report", "description": "All tests passed. Full report attached.", "color": 3066993}]}' \
  -F "files[0]=@/tmp/report.html;filename=test-report.html" \
  "https://discord.com/api/v10/channels/$TARGET/messages"
```

### Upload Limits

| Tier | Max file size |
|------|--------------|
| Free / default | 25 MB |
| Nitro (server boost) | Up to 100 MB |
| Attachments per message | 10 |

## Rate Limit Handling

Discord enforces per-route rate limits. Every response includes rate-limit headers.

### Response Headers

| Header | Type | Description |
|--------|------|-------------|
| `X-RateLimit-Limit` | integer | Total requests allowed in the current window |
| `X-RateLimit-Remaining` | integer | Requests remaining before hitting the limit |
| `X-RateLimit-Reset` | float | Unix timestamp (seconds) when the window resets |
| `X-RateLimit-Reset-After` | float | Seconds until the window resets |
| `X-RateLimit-Bucket` | string | Opaque bucket ID (routes sharing a bucket share a limit) |

### Handling HTTP 429

When you exceed the rate limit, Discord returns HTTP 429 with a JSON body:

```json
{
  "message": "You are being rate limited.",
  "retry_after": 1.234,
  "global": false
}
```

| Field | Description |
|-------|-------------|
| `retry_after` | Seconds to wait before retrying (float) |
| `global` | If `true`, the limit applies to all endpoints, not just this route |

### Backoff Strategy

1. **Check before sending.** Read `X-RateLimit-Remaining` from the previous response. If it's `0`, sleep for `X-RateLimit-Reset-After` seconds before the next request.
2. **React to 429.** If you get HTTP 429, read `retry_after` from the JSON body and sleep that many seconds before retrying.
3. **Global limits.** If `global` is `true`, all requests (not just the current route) must wait. This is rare and indicates very aggressive usage.

### Practical Example

```bash
TARGET="${DISCORD_THREAD_ID:-$DISCORD_CHANNEL_ID}"

response=$(curl -s -w "\n%{http_code}" -X POST \
  -H "Authorization: Bot $DISCORD_BOT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"content": "Progress update: step 3 of 5 complete."}' \
  "https://discord.com/api/v10/channels/$TARGET/messages")

http_code=$(echo "$response" | tail -1)
body=$(echo "$response" | sed '$d')

if [ "$http_code" = "429" ]; then
  retry_after=$(echo "$body" | python3 -c "import sys,json; print(json.load(sys.stdin)['retry_after'])")
  sleep "$retry_after"
  # Retry the request
fi
```

### Common Rate Limits

| Route | Limit | Window |
|-------|-------|--------|
| POST message (per channel) | 5 requests | 5 seconds |
| DELETE message | 5 requests | 1 second |
| PATCH message | 5 requests | 5 seconds |
| Global | 50 requests | 1 second |

These are approximate — Discord can adjust them. Always read the headers rather than hard-coding limits.

## Quick Reference

### Env Vars (Injected by Adapter)

| Variable | Description |
|----------|-------------|
| `$DISCORD_BOT_TOKEN` | Bot auth token |
| `$DISCORD_SERVER_ID` | Guild (server) ID |
| `$DISCORD_CHANNEL_ID` | Parent channel ID |
| `$DISCORD_THREAD_ID` | Thread ID (empty if not in a thread) |
| `$DISCORD_USER_ID` | ID of the user who triggered the session |
| `$ORBH_SESSION_ID` | The orbh session ID |

### Target Pattern

```bash
TARGET="${DISCORD_THREAD_ID:-$DISCORD_CHANNEL_ID}"
```

### Endpoint Summary

| Operation | Method | Path |
|-----------|--------|------|
| Send message | `POST` | `/channels/{id}/messages` |
| Get messages | `GET` | `/channels/{id}/messages` |
| Get single message | `GET` | `/channels/{id}/messages/{msg.id}` |
| Edit message | `PATCH` | `/channels/{id}/messages/{msg.id}` |
| Delete message | `DELETE` | `/channels/{id}/messages/{msg.id}` |
| Upload file | `POST` | `/channels/{id}/messages` (multipart) |
| Get channel | `GET` | `/channels/{id}` |
| Create reaction | `PUT` | `/channels/{id}/messages/{msg.id}/reactions/{emoji}/@me` |
