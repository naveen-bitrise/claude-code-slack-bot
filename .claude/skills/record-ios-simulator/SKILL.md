---
name: record-ios-simulator
description: Record the iOS Simulator screen to a video file. Use when the user asks to record, capture, or save a screen recording from the iOS Simulator.
tools: Bash
---

# Record iOS Simulator Screen

Record the iOS Simulator screen to a `.mp4` file using `xcrun simctl io`.

## Arguments

- `$ARGUMENTS` — the output file path (e.g. `recording.mp4`). Defaults to `simulator_recording.mp4` in the current directory if omitted.

---

## Step 1 — Resolve the Output Path

```bash
OUTPUT="${ARGUMENTS:-simulator_recording.mp4}"
echo "Output file: $OUTPUT"
---

## Step 2 — Find the Target Simulator

bash
xcrun simctl list devices booted
If exactly one simulator is booted, use booted. If multiple are booted, ask the user which UDID to use.

bash
DEVICE="booted"  # or a specific UDID
---

## Step 3 — Start the Recording

bash
xcrun simctl io "$DEVICE" recordVideo "$OUTPUT" &
RECORD_PID=$!
echo "Recording started (PID $RECORD_PID) → $OUTPUT"
---

## Step 4 — Stop the Recording

Always stop with SIGINT, never SIGKILL. SIGKILL prevents the moov atom from being written, making the file unplayable.

bash
kill -SIGINT $RECORD_PID
wait $RECORD_PID
sleep 1
echo "Recording stopped."
---

## Step 5 — Verify the Output File

bash
ls -lh "$OUTPUT"
ffprobe -v error -show_entries format=duration,size \
  -of default=noprint_wrappers=1 "$OUTPUT"
---

## Step 6 — Optional: Re-encode for Compatibility

bash
ffmpeg -y -i "$OUTPUT" \
  -vcodec libx264 -pix_fmt yuv420p \
  -movflags +faststart \
  -crf 23 \
  "${OUTPUT%.mp4}_final.mp4"
---

## Full One-Shot Script (fixed duration)

bash
OUTPUT="${ARGUMENTS:-simulator_recording.mp4}"
DEVICE="booted"
DURATION=10  # seconds

xcrun simctl io "$DEVICE" recordVideo "$OUTPUT" &
RECORD_PID=$!
echo "Recording for ${DURATION}s..."
sleep "$DURATION"
kill -SIGINT $RECORD_PID
wait $RECORD_PID
sleep 1
echo "Saved to: $OUTPUT"
ls -lh "$OUTPUT"
---

## Key Gotchas

| Issue | What happens | Fix |
|---|---|---|
| Stop with SIGKILL / kill -9 | moov atom not found — file is unplayable | Always use kill -SIGINT + wait |
| No simulator booted | xcrun simctl error | Boot one first: xcrun simctl boot <UDID> |
| Multiple simulators booted | Records wrong device | Specify UDID explicitly |
| Disk full mid-recording | Truncated/corrupt file | Check free space first |
| File not playable in QuickTime | Codec issue | Re-encode with ffmpeg (Step 6) |
| xcrun not found | Xcode CLT missing | Run xcode-select --install |
