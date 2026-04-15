---
description: "Media-fence grammar for final-output rendering — discord-image-N, discord-embed, discord-attachment syntax, validation behavior, and worked examples"
---

# Knowledge: Output Format Convention

The Discord adapter parses your session's final output for triple-backtick **media fences** that expand into native Discord content — images, rich embeds, and file attachments. Text outside fences passes through unchanged as Markdown.

Use fences when you want the adapter to render rich primitives in the result it posts. For mid-run status messages while you're still working, use direct REST API calls instead (see [[dev-knw-odisc-rest_api]]).

## Fence Types

| Fence | Body | Produces |
|-------|------|----------|
| ` ```discord-image-N ` | Plain file path (not JSON) | Inline image attachment |
| ` ```discord-embed ` | JSON object | Rich embed block |
| ` ```discord-attachment ` | JSON object with `path` and optional `filename` | File attachment |

Only triple backticks are recognized. Tildes (`~~~`) do not work. The fence-type name must be exact — no leading or trailing whitespace inside the name.

## Parser Behavior

The adapter uses three regex patterns (source: `orb-discord/src/orb_discord/formatting.py`):

```
r"```discord-image-(\d+)\s*\n(.+?)\n```"
r"```discord-embed\s*\n(.+?)\n```"
r"```discord-attachment\s*\n(.+?)\n```"
```

| Condition | Result |
|-----------|--------|
| Valid fence, valid body, file exists (if applicable) | Fence removed from text; content attached to the Discord message |
| Fence name matches but JSON body is malformed (embed/attachment) | Warning logged; fence left in text as raw content |
| Image or attachment path missing on disk | Warning logged; fence left in text as raw content |
| Text outside any fence | Passes through as Markdown |

After fences are extracted, the remaining text is trimmed of leading/trailing whitespace. If no text remains, the message carries only the attached primitives.

## Fence Reference

### discord-image-N

Render an image inline. `N` is an integer index controlling ordering when you emit multiple images — lower indices render first.

Body: a single line containing a file path (not JSON). Path must exist on disk at render time. Supported types render inline in Discord: PNG, JPG, GIF, WebP. Other formats become download links.

````
```discord-image-0
/tmp/screenshot.png
```
````

For multiple images:

````
```discord-image-0
/tmp/before.png
```

```discord-image-1
/tmp/after.png
```
````

### discord-embed

Render a rich embed block. Body is a JSON object. All fields optional.

| Field | Type | Meaning |
|-------|------|---------|
| `title` | string | Embed title (max 256 chars) |
| `description` | string | Main body text, supports Markdown (max 4096 chars) |
| `color` | integer | Decimal color value (e.g. `3066993` for green) |
| `fields` | array | Each element: `{"name": str, "value": str, "inline": bool}` |

See [[dev-knw-odisc-rest_api#Color Values]] for common color decimals and [[dev-knw-odisc-rest_api#Embed Limits]] for full character caps.

````
```discord-embed
{"title": "Build complete", "description": "All 42 tests passed.", "color": 3066993}
```
````

### discord-attachment

Attach an arbitrary file to the message. Body is a JSON object.

| Field | Required | Meaning |
|-------|----------|---------|
| `path` | yes | Absolute path to a file on disk |
| `filename` | no | Display name shown in Discord; defaults to `basename(path)` |

````
```discord-attachment
{"path": "/tmp/report-raw.pdf", "filename": "test-report.pdf"}
```
````

## No Markdown Tables in Return Output

Discord does not render markdown pipe-tables. Inside embed descriptions, `| col | col |` and `|----|----|` appear as raw pipe characters with no column layout — the result looks broken.

**Rule:** do not emit markdown tables in the text you deliver via `flint orbh session <id> return "..."`. The adapter wraps that text in an embed (or paginated embeds); tables do not survive.

Use instead:

| Mechanism | When |
|-----------|------|
| Bullet list with bold labels | Short key/value readout |
| `discord-embed` with `fields[]` | Multiple structured rows, optionally inline columns |
| `discord-attachment` with a `.md` or `.csv` file | Larger tables that belong as a download |

**Avoid** (renders as raw pipe characters in Discord):

```markdown
| Test | Result |
|------|--------|
| auth | pass |
| api | pass |
```

**Prefer — bullet list:**

```markdown
- **auth:** pass
- **api:** pass
```

**Prefer — embed fields:**

````
```discord-embed
{"title": "Results", "fields": [
  {"name": "auth", "value": "pass", "inline": true},
  {"name": "api", "value": "pass", "inline": true}
]}
```
````

This rule applies only to text the adapter renders into Discord. Markdown tables inside knowledge files, Mesh artifacts, and anywhere else humans read via Obsidian are fine — the rule is return-output-only.

## When to Use Fences vs Direct API Calls

| Use case | Mechanism |
|----------|-----------|
| Your session's final result returned via `flint orbh session <id> return "..."` | **Fences** inside the returned Markdown |
| Mid-run progress update while you're still working | **REST API** (see [[dev-knw-odisc-rest_api]]) |
| Ad-hoc single message from a shell script | **REST API** |
| Output that will never be rendered by the adapter (raw subprocess stdout, log files) | Not parsed — ignored |

Fences only activate when the adapter renders your output. Text that does not flow through the adapter — anything you write straight to stdout for your own inspection, for example — is never parsed.

## Worked Examples

### Example 1 — Final result with an embed

A session finishes running a test suite and returns a structured summary. Text before and after the fence renders as plain Markdown; the fence becomes a green embed attached to the same message.

````markdown
The suite completed in 2m 14s.

```discord-embed
{
  "title": "Test Report",
  "description": "All 42 tests passed.",
  "color": 3066993,
  "fields": [
    {"name": "Duration", "value": "2m 14s", "inline": true},
    {"name": "Failed", "value": "0", "inline": true}
  ]
}
```

Full log attached for reference.
````

Deliver via:

```bash
flint orbh session $ORBH_SESSION_ID return "$(cat <<'EOF'
The suite completed in 2m 14s.

\`\`\`discord-embed
{"title": "Test Report", "description": "All 42 tests passed.", "color": 3066993}
\`\`\`
EOF
)"
```

### Example 2 — Final result with an inline image

Capture a screenshot, then emit a `discord-image-0` fence pointing at the saved file. The numeric suffix controls ordering (only relevant with multiple images).

````markdown
Here is the rendered dashboard after the refactor.

```discord-image-0
/tmp/dashboard-2026-04-15.png
```
````

Multiple images order by their numeric suffix — `-0` before `-1`, and so on.

### Example 3 — Embed plus downloadable attachment

One output combining two fences. Both are stripped from the text; both attach to the same Discord message.

````markdown
```discord-embed
{"title": "Deploy succeeded", "description": "Logs attached below.", "color": 3066993}
```

```discord-attachment
{"path": "/tmp/deploy-2026-04-15.log", "filename": "deploy.log"}
```
````

## Common Pitfalls

- **Wrong fence name.** ` ```discord-image ` without the `-N` suffix will not match the image parser and renders as a literal code block. The suffix is mandatory.
- **Unquoted JSON strings.** `discord-embed` and `discord-attachment` bodies must be valid JSON. Python-style `{'title': '...'}` fails — use double quotes.
- **Missing file paths.** Images and attachments must exist on disk at render time. If the path is missing, the fence is left in the output as-is (no silent drop).
- **Fences inside fence bodies.** The parser's regex is greedy; nesting triple backticks inside a fence body can break parsing. Keep bodies flat.
- **Trying to fence mid-run messages.** Fences activate only when the adapter posts your output. Writing a fence to stdout mid-run does nothing — use REST calls for in-progress messages.
- **Echoing the token inside an embed description.** Even a fenced embed runs through Discord; if `description` contains `DISCORD_BOT_TOKEN`, the token is posted publicly. See [[dev-knw-odisc-env_context#Security]].
