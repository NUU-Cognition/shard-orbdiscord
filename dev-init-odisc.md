---
required-reading:
  - knowledge/dev-knw-odisc-env_context.md
  - knowledge/dev-knw-odisc-output_format.md
  - "[[knw-odisc-rest_api]]"
---

# OrbDiscord

Knowledge for Orbh sessions running inside a Discord thread or channel. Tells the session what environment context the Discord adapter injects and what media-fence grammar the adapter parses so output renders natively as embeds, images, and attachments.

## What This Shard Is For

The Discord adapter (`orb-discord`) launches Orbh sessions on behalf of Discord users. It injects Discord-specific environment variables into the session and parses the session's final output for media fences that expand into rich Discord content.

This shard is documentation-only — no skills, no workflows. Load it when:

- You need to know what env vars are available in your session
- You want to emit an image, rich embed, or file attachment in your result
- You need to make direct Discord REST API calls (see [[dev-knw-odisc-rest_api]])

## You Are an Orbh Session

Every Discord session runs through Orbh — `$ORBH_SESSION_ID` is the same session ID you pass to `flint orbh session <id> ...` commands. Load `Shards/Orbh/init-foh.md` and follow all Orbh operating rules:

- Register immediately: `flint orbh session $ORBH_SESSION_ID register "<title>" "<description>"`
- Track status: `flint orbh session $ORBH_SESSION_ID status in-progress`
- Deliver the result: `flint orbh session $ORBH_SESSION_ID return "<markdown>"` — this is the text the adapter parses for media fences
- Track artifacts: `flint orbh artifact $ORBH_SESSION_ID "(Type) NNN Name"`

OrbDiscord layers Discord-specific context and output grammar on top of Orbh; it does not replace Orbh conventions.

## Session Context Summary

When a session is launched from Discord, the adapter sets process env vars and appends a `<discord-context>` block to your prompt. Full detail in [[dev-knw-odisc-env_context]].

| Variable | Always set | Purpose |
|----------|------------|---------|
| `DISCORD_BOT_TOKEN` | yes | Bot auth for REST calls (sensitive — never echo) |
| `DISCORD_USER_ID` | yes | User who triggered the session |
| `DISCORD_CHANNEL_ID` | yes | Parent channel |
| `DISCORD_SERVER_ID` | guild only | Guild/server ID (empty in DMs) |
| `DISCORD_THREAD_ID` | threads only | Thread ID (empty in channel roots) |
| `DISCORD_ORIGIN_CHANNEL_ID` | yes | Channel/thread where the user originally triggered the session |
| `DISCORD_ORIGIN_CHANNEL_NAME` | yes | Human-readable name of the origin (e.g. `general` or `my-forum > Post Title`) |
| `ORBH_SESSION_ID` | yes | Your Orbh session ID |

**Channel targeting:** prefer `$DISCORD_THREAD_ID` when set; fall back to `$DISCORD_CHANNEL_ID`:

```bash
TARGET="${DISCORD_THREAD_ID:-$DISCORD_CHANNEL_ID}"
```

## Output Channels

Two ways to put content in Discord:

1. **Media fences in your final output** — the adapter parses ` ```discord-embed `, ` ```discord-image-N `, and ` ```discord-attachment ` fences in the result and turns them into native Discord objects. Markdown outside fences passes through unchanged. Full grammar and worked examples in [[dev-knw-odisc-output_format]].
2. **Direct REST API calls for mid-run messages** — `curl` against `https://discord.com/api/v10` while you work. Use env vars for auth and targeting. Full reference in [[dev-knw-odisc-rest_api]].

## Rules

- **Never echo `DISCORD_BOT_TOKEN`** in any Discord message, embed, fence, log, or Mesh artifact. See [[dev-knw-odisc-env_context#Security]].
- **Fence grammar is strict.** Triple-backtick only, exact fence-type names, image fences require a numeric suffix. Malformed fences are left in the output as raw text.
- **Markdown outside fences passes through.** Only emit fences when you need native Discord primitives; normal prose renders fine as-is.
- **No markdown tables in `orbh return` output.** Discord embeds render pipe-tables as raw characters with no column layout. Use bullet lists, bold key/value pairs, or `discord-embed` with the `fields` array instead. See [[dev-knw-odisc-output_format#No Markdown Tables in Return Output]].
- **Identify the user before acting on their behalf.** Reverse-lookup `$DISCORD_USER_ID` in `Mesh/People/` to find the person artifact. If found, use them for the `authors:` field on artifacts you create; if not, fall back to `.flint/identity.json`. See [[dev-knw-odisc-env_context#Identifying the User]].
- **Prefer `$DISCORD_THREAD_ID` over `$DISCORD_CHANNEL_ID`** for messaging — never hard-code IDs or tokens.
- **Download incoming attachments immediately** to `Media/Discord/Temp/`. CDN URLs expire — grab files before processing. See [[dev-knw-odisc-env_context#Incoming Attachments]].
- **Mention people when referring to them.** When your message references a person by name, look them up in `Mesh/People/` and check for a `discord-id` field. If found, use Discord's `<@discord-id>` mention syntax so they get notified. See [[dev-knw-odisc-env_context#Mentioning Users]].
- **Fetch origin context on demand.** The adapter does not prefetch messages from the origin channel. If you need conversation context from where the user invoked you, use the REST API with `$DISCORD_ORIGIN_CHANNEL_ID`. See [[dev-knw-odisc-env_context#Origin Context]].
