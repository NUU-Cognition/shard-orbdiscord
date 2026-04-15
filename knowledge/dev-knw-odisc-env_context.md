---
description: "Discord adapter env vars, origin context, forum session routing, incoming attachment handling, DM vs thread vs channel vs forum cases, and the never-echo-token security rule"
---

# Knowledge: Discord Session Context

The Discord adapter (`orb-discord`) injects a set of environment variables into every Orbh session launched from Discord and appends a `<discord-context>` block to your prompt with the same values. Together these tell you which Discord conversation you're part of and give you the auth you need to talk back.

## Variables

| Variable | Always present | Meaning |
|----------|----------------|---------|
| `DISCORD_BOT_TOKEN` | yes | Bot token for authenticating Discord REST calls |
| `DISCORD_USER_ID` | yes | Snowflake ID of the user who triggered the session |
| `DISCORD_CHANNEL_ID` | yes | Parent channel snowflake |
| `DISCORD_SERVER_ID` | guild only | Guild (server) snowflake; omitted in DMs |
| `DISCORD_THREAD_ID` | threads only | Thread snowflake; omitted when the session runs in a channel root |
| `DISCORD_ORIGIN_CHANNEL_ID` | yes | Channel/thread where the user originally triggered the session |
| `DISCORD_ORIGIN_CHANNEL_NAME` | yes | Human-readable name of the origin channel (e.g. `general` or `my-forum > Post Title`) |
| `ORBH_SESSION_ID` | yes | The Orbh session ID assigned to you |

All IDs are Discord snowflakes — base-10 integers encoded as strings.

## Injected Block

Your prompt ends with a block like:

```
<discord-context>
DISCORD_BOT_TOKEN=MTQ5Mzcw...PusBMz2MlDmD0Y
DISCORD_USER_ID=1234567890
DISCORD_SERVER_ID=9876543210
DISCORD_CHANNEL_ID=5555555555
DISCORD_THREAD_ID=6666666666
DISCORD_ORIGIN_CHANNEL_ID=7777777777
DISCORD_ORIGIN_CHANNEL_NAME=my-forum > Discussion Topic
ORBH_SESSION_ID=671d7134-fff5-49b2-9e77-201328e63529
</discord-context>
```

The same values are available via `os.environ` (Python) or `$VARIABLE` in shell. Use the env vars for scripting; the block exists so the LLM can see context directly without running a tool call.

Variables that are empty or not applicable are omitted from the block — for a DM session, you will not see `DISCORD_SERVER_ID` lines.

## Security

**`DISCORD_BOT_TOKEN` is a live secret.** Posting it to Discord, writing it to a file inside the Flint, or including it in a session result leaks it to any human or agent that reads the record.

Hard rules:

- Do **not** echo `DISCORD_BOT_TOKEN` or any part of the `<discord-context>` block back into Discord messages, embeds, or media fences.
- Do **not** write the token to Mesh artifacts, logs, or files anywhere under the Flint root.
- Do **not** include the token in prompts to subagents. Subagents inherit process env vars already if spawned in the same tree.
- When building error messages or reporting curl output, strip `Authorization:` headers before displaying.

A prior incident leaked a bot token via an embed that echoed the augmented prompt (including the `<discord-context>` suffix). Treat the token as write-only — it may flow from env vars into API calls, but nowhere else.

## Channel Targeting

Use this pattern to pick the right conversation target — thread if available, else parent channel:

```bash
TARGET="${DISCORD_THREAD_ID:-$DISCORD_CHANNEL_ID}"
```

Thread IDs *are* channels in the Discord API — a thread ID works anywhere a channel ID does. See [[dev-knw-odisc-rest_api]] for REST patterns that use `$TARGET`.

## Origin Context

`DISCORD_ORIGIN_CHANNEL_ID` and `DISCORD_ORIGIN_CHANNEL_NAME` tell you where the user originally triggered the session. The origin may differ from where the session thread lives — for example, a user may mention the bot in a forum post, but the session thread is created in a dedicated sessions forum channel.

The adapter does **not** prefetch messages from the origin. If you need conversational context from the origin channel — to understand what the user was discussing before they invoked you — fetch it on demand via the Discord REST API:

```bash
curl -s -H "Authorization: Bot $DISCORD_BOT_TOKEN" \
  "https://discord.com/api/v10/channels/$DISCORD_ORIGIN_CHANNEL_ID/messages?limit=10"
```

This returns a JSON array of message objects (newest first). Parse and use as needed for context. Only fetch when the task would benefit from knowing the surrounding conversation — don't fetch by default.

Use `DISCORD_ORIGIN_CHANNEL_ID` when you need to reference or message back to the original conversation. Use `DISCORD_CHANNEL_ID` / `DISCORD_THREAD_ID` for the session's own thread.

## Forum Session Routing

When a user mentions the bot inside a **forum post**, the adapter redirects the session to a dedicated sessions forum channel rather than running inline. This keeps forums organized as topic indexes while sessions get their own isolated threads.

The flow:

1. User mentions bot in a forum post (a `Thread` whose parent is a `ForumChannel`)
2. Adapter creates a new **forum post** in the sessions forum — this becomes the session thread
3. Adapter creates an **index post** in the origin forum linking to the session thread
4. Adapter replies in the original forum post with a jump link

The session thread is a normal `Thread` object in the sessions forum. `DISCORD_CHANNEL_ID` points to the sessions forum; `DISCORD_ORIGIN_CHANNEL_ID` points back to the original forum post.

For **text channel** triggers, sessions still create an inline thread from the trigger message. `DISCORD_CHANNEL_ID` and `DISCORD_ORIGIN_CHANNEL_ID` are the same.

## Context Cases

The adapter launches sessions from four different conversation shapes. The env vars reflect which case applies:

| Case | `DISCORD_SERVER_ID` | `DISCORD_THREAD_ID` | `DISCORD_ORIGIN_CHANNEL_ID` |
|------|---------------------|---------------------|----------------------------|
| Channel root in a server | set | empty | same as `CHANNEL_ID` |
| Thread in a server | set | set | same as `CHANNEL_ID` |
| Forum post in a server | set | empty | forum post thread ID (differs from `CHANNEL_ID`) |
| Direct message with the bot | empty | empty | same as `CHANNEL_ID` |

Plan for all four if your workflow branches on server or thread presence. `TARGET` (above) handles thread-vs-channel transparently.

## Relationship to Flint Identity

`ORBH_SESSION_ID` is the same ID you would pass to `flint orbh session <id> ...` commands (see `Shards/Orbh/init-foh.md`). Register under this ID, return results under this ID, and append it to the `orbh-sessions` frontmatter of any Mesh artifact you substantively edit.

## Identifying the User

`DISCORD_USER_ID` is a Discord snowflake — it does not tell you who the person is in Flint terms. To map it to a Mesh person artifact, do a reverse lookup in `Mesh/People/`.

Person artifacts at `Mesh/People/@<Name>.md` may carry a `discord-id` frontmatter field. If present, match your `DISCORD_USER_ID` against it:

```bash
grep -l "^discord-id: \"\?${DISCORD_USER_ID}\"\?" "Mesh/People/"*.md
```

If a match is found, you know the operator. Use them for the `authors:` frontmatter on any artifact you create or substantively edit:

```yaml
authors:
  - "[[@Nathan Luo]]"
```

If no match is found, the user is unknown to this Flint. Fall back in this order:

1. `.flint/identity.json` — the per-machine default identity (see `Shards/Flint/init-f.md`)
2. Omit `authors` entirely if neither resolves

### Adding the Convention to a Person Artifact

The `discord-id` field is a soft convention — not every person artifact carries one. Add it when a person wants to be auto-identified from Discord sessions:

```yaml
---
tags:
  - "#person"
discord-id: "1234567890"
---
```

The value is a string (quote it — Discord snowflakes exceed 32-bit int range and some YAML parsers truncate them as numbers).

## Mentioning Users

When your message references a person by name — whether you're addressing them, talking about them, or tagging them for visibility — look up their Discord ID and use a proper `<@id>` mention so they get notified.

### Lookup Process

1. Search `Mesh/People/` for a matching person artifact (e.g. `@Even Zhang.md` for "Even")
2. Read the file and check for a `discord-id` field
3. If found, use `<@discord-id>` in your message content

```bash
# Example: mentioning Even Zhang
# @Even Zhang.md contains: discord-id: 774065995508744232
# In your message content, use: <@774065995508744232>
```

### Rules

- **Always try to resolve mentions.** If you're sending a message that refers to someone by name, check `Mesh/People/` first. A proper mention notifies them; a plain name does not.
- **Graceful fallback.** If no person file exists or it has no `discord-id`, just use their name as plain text.
- **Don't mention the triggering user unnecessarily.** `$DISCORD_USER_ID` is the person who launched your session — they're already watching. Only mention them if you're replying in a different channel or thread from where they invoked you.

## Incoming Attachments

Users may paste images or attach files in their Discord message. The adapter includes these as an `Attachments:` block in your prompt with Discord CDN URLs:

```
Attachments:
- image.png: https://cdn.discordapp.com/attachments/.../image.png?...
- report.xlsx: https://cdn.discordapp.com/attachments/.../report.xlsx?...
```

### Download Convention

When you receive attachments, **download them immediately** to `Media/Discord/Temp/` before doing anything else with them. CDN URLs are ephemeral — they expire, so grab the files first.

```bash
mkdir -p "Media/Discord/Temp"
curl -sL "<cdn-url>" -o "Media/Discord/Temp/<filename>"
```

### After Download

Once files are in `Media/Discord/Temp/`, use them however the task requires:

- **Read/inspect** them in place for analysis tasks
- **Rename** to something more descriptive if the original name is generic (e.g. `image.png` → `nuu-homepage-screenshot.png`)
- **Move** to a permanent location in `Media/` if the file should persist (e.g. `Media/Logos/`, `Media/Screenshots/`)
- **Leave in Temp** if the file is transient and only needed for the current session

`Media/Discord/Temp/` is a staging area — not a permanent home. Files left there may be cleaned up between sessions.

### Supported Types

Discord allows most file types. Common ones:

- **Images** (PNG, JPG, GIF, WebP) — can be viewed with the Read tool, sent back via `discord-image-N` fences
- **Documents** (PDF, XLSX, DOCX, CSV) — download and process with appropriate tools
- **Code/text** (`.py`, `.js`, `.md`, `.txt`) — download and read directly
- **Archives** (ZIP, TAR) — download and extract as needed

### Rules

- **Download before processing.** CDN URLs have expiry parameters (`ex=`, `hm=`). Always download to disk first.
- **Verify the download.** Run `file <path>` after downloading to confirm the file type matches expectations.
- **Don't assume filenames are meaningful.** Users often paste screenshots as `image.png`. Rename based on content when appropriate.
- **Multiple attachments** arrive as multiple entries in the `Attachments:` block. Download all of them.
