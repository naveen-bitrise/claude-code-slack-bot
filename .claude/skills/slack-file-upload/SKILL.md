---
name: slack-upload
description: Upload a local file to the current Slack thread. Use when the user asks to send, share, post, or upload a file to Slack or to "this thread".
tools: Bash
---

# Slack File Upload

Upload a local file into the currently active Slack thread, without using any public file hosting services.

## CRITICAL SECURITY RULE

NEVER echo, print, or include the SLACK_BOT_TOKEN value in any output, logs, or messages.
Always reference it as `$SLACK_BOT_TOKEN` in commands — never expand or display its value.
Do NOT post bash commands containing the raw token to Slack threads.

## Prerequisites

- `SLACK_BOT_TOKEN` must be available in the environment
- For video files, `ffmpeg` should be available for re-encoding

## Arguments

- `$ARGUMENTS` — the local path of the file to upload

---

## Step 1 — Resolve the Bot Token

```bash
echo $SLACK_BOT_TOKEN
```

If empty, scan the full environment:
```bash
env | grep -i slack
```

Also get the bot's own user ID — you'll need it to identify bot messages in channel history:
```bash
curl -s -H "Authorization: Bearer $SLACK_BOT_TOKEN" \
  "https://slack.com/api/auth.test"
# Note the "user_id" field
```

---

## Step 2 — Resolve Channel and Thread

Look for `[Slack context: channel_id=..., thread_ts=...]` in the conversation.
If found, use those values directly for CHANNEL and THREAD_TS.
Do NOT search for channels or threads — the provided values are always correct.

---

## Step 3 — Prepare the File

For **video files** (`.mp4`, `.mov`, `.avi`), re-encode with ffmpeg before uploading to ensure playback compatibility:

```bash
ffmpeg -y -i input.mp4 \
  -vcodec libx264 -pix_fmt yuv420p \
  -movflags +faststart \
  -crf 23 \
  output_final.mp4
```

> **Important — `simctl recordVideo` gotcha:** Always stop the recording process with `SIGINT`, not `SIGKILL`. `SIGKILL` prevents the `moov` atom from being written, making the file unplayable (`moov atom not found`):
> ```bash
> kill -SIGINT $RECORD_PID
> wait $RECORD_PID   # wait for it to fully flush to disk
> sleep 2
> ```
> Verify the file is valid before uploading:
> ```bash
> ffprobe -v error -show_entries format=duration,size \
>   -of default=noprint_wrappers=1 output_final.mp4
> ```

---

## Step 4 — Upload via Slack Files v2 API

### 4a — Get a signed upload URL

```bash
FILE_SIZE=$(stat -f%z /path/to/file)   # macOS
# Linux: FILE_SIZE=$(stat -c%s /path/to/file)

UPLOAD_RESPONSE=$(curl -s -X POST \
  -H "Authorization: Bearer $SLACK_BOT_TOKEN" \
  -d "filename=yourfile.mp4&length=${FILE_SIZE}" \
  "https://slack.com/api/files.getUploadURLExternal")

UPLOAD_URL=$(echo "$UPLOAD_RESPONSE" | python3 -c "import json,sys; d=json.load(sys.stdin); print(d['upload_url'])")
FILE_ID=$(echo "$UPLOAD_RESPONSE"    | python3 -c "import json,sys; d=json.load(sys.stdin); print(d['file_id'])")
```

### 4b — Upload the file bytes

```bash
curl -s -X POST "$UPLOAD_URL" \
  -H "Authorization: Bearer $SLACK_BOT_TOKEN" \
  -H "Content-Type: video/mp4" \
  --data-binary @/path/to/file
# Should return: OK - <byte_count>
```

### 4c — Complete the upload and share to the thread

```bash
COMPLETE=$(curl -s -X POST \
  -H "Authorization: Bearer $SLACK_BOT_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{
    \"files\": [{\"id\": \"${FILE_ID}\"}],
    \"channel_id\": \"${CHANNEL}\",
    \"thread_ts\": \"${THREAD_TS}\",
    \"initial_comment\": \"Your message here\"
  }" \
  "https://slack.com/api/files.completeUploadExternal")
```

---

## Step 5 — Verify & Fallback

Inspect the `completeUploadExternal` response:

```bash
echo "$COMPLETE" | python3 -c "
import json,sys
d=json.load(sys.stdin)
print('ok:', d.get('ok'))
f=d.get('files',[{}])[0]
print('shares:', json.dumps(f.get('shares', {})))
print('permalink:', f.get('permalink',''))
"
```

**If `shares` is `{}` (empty)** — this is a known silent failure for DM channels. The file was uploaded to Slack but not shared into the thread. Fall back to posting the file's `permalink` as a message:

```bash
PERMALINK=$(echo "$COMPLETE" | python3 -c "
import json,sys; d=json.load(sys.stdin)
print(d['files'][0]['permalink'])
")

curl -s -X POST \
  -H "Authorization: Bearer $SLACK_BOT_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{
    \"channel\": \"${CHANNEL}\",
    \"thread_ts\": \"${THREAD_TS}\",
    \"text\": \"${PERMALINK}\"
  }" \
  "https://slack.com/api/chat.postMessage"
```

Finally, confirm the message appeared in the thread:

```bash
curl -s -H "Authorization: Bearer $SLACK_BOT_TOKEN" \
  "https://slack.com/api/conversations.replies?channel=${CHANNEL}&ts=${THREAD_TS}&limit=5" \
  | python3 -c "
import json,sys
d=json.load(sys.stdin)
for m in d.get('messages',[])[-3:]:
    print(m.get('ts'), '|', m.get('text','')[:80])
"
```

---

## Key Gotchas

| Issue | What happens | Fix |
|---|---|---|
| Public file hosts | Leaks files outside Slack | **Never use** transfer.sh, catbox.moe, etc. |
| `files.upload` | `method_deprecated` error | Use Files v2 API (Steps 5a–5c) |
| `completeUploadExternal` in DMs | Returns `ok:true` but `shares:{}` — file not visible | Fall back to `chat.postMessage` with permalink (Step 6) |
| `files.sharedPublicURL` | `not_allowed_token_type` | Requires user token, not bot token — don't use |
| `moov atom not found` | Video unplayable | Use `SIGINT` not `SIGKILL` to stop `simctl recordVideo` |
| Channel not found | Bot picks wrong channel | Cross-check recent message text against current conversation |
