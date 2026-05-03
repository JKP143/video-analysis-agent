# Video Analysis Agent — Workflow SOP

## Objective

Given a video sent to a Telegram bot (either an uploaded video file or a pasted YouTube link), produce a structured markdown analysis, save it as a Google Doc in a dedicated "Video Analyses" Drive folder, and reply in Telegram with a short summary and a clickable button that opens the doc.

## Inputs

Any Telegram message to the bot that contains **one of**:

1. A `video` attachment (`.mp4`, `.mov`, etc.) up to **20 MB** (Telegram Bot API's download cap).
2. A message whose text or caption contains a **YouTube URL** — matches `youtube.com/watch?v=...` or `youtu.be/...`.

An optional caption (for a video) or additional text (for a YouTube link) is passed to Gemini as a custom analysis prompt. If none, a default transcribe + describe prompt is used.

Anything else (a plain text message without a YouTube URL, a photo, a PDF, etc.) receives a short rejection reply.

## Outputs

- A new Google Doc in the **Video Analyses** Drive folder with:
  - Title: `Video Analysis — YYYY-MM-DD HH:mm — msg <telegram_message_id>`
  - Body: a markdown document with sections `Summary`, `Key moments`, `Visual walkthrough`, `Transcript`.
- A Telegram reply with a short summary and an inline keyboard button `Open analysis` linking to the doc.

## Topology

```
Telegram Trigger
  -> Parse Input (extract file_id / youtube_url / chat_id / message_id / user_prompt)
  -> Route By Input Type
      [video]   -> Telegram Get File
                -> Gemini Upload Start (resumable, returns x-goog-upload-url header)
                -> Gemini Upload Bytes (POST raw binary to upload URL)
                -> Wait 10s -> Gemini Poll -> Still Processing?
                    true  (state is NOT "ACTIVE" and NOT "FAILED" — i.e. PROCESSING / STATE_UNSPECIFIED / missing) -> Wait 10s (loops until Gemini is done)
                    false (state is ACTIVE or FAILED — terminal) -> Set File URI From Video
      [youtube] -> Set File URI From YouTube (no upload)
      [else]    -> Telegram Reject (terminal)
  -> Gemini Generate Content (gemini-2.5-pro, JSON response)
  -> Parse Analysis (Code — robust JSON parsing with repair/recovery)
  -> Narrative Agent (OpenRouter / Claude Sonnet 4.5 -> markdown)
  -> Create Google Doc (in "Video Analyses" folder)
  -> Write To Google Doc (insert the markdown body)
  -> Build Telegram Message (Code — extract title + summary, build inline keyboard)
  -> Send Telegram Reply (Markdown parse_mode + inline keyboard)
```

## Required credentials (set up in the local n8n at http://localhost:5678 before importing)

| n8n credential name    | Credential type            | What to paste                                           |
|------------------------|---------------------------- |---------------------------------------------------------|
| `Telegram Bot`         | Telegram API                | Bot token (same bot used for OCR Invoice Agent works)   |
| `Gemini API Key`       | Header Auth                 | Name: `x-goog-api-key` · Value: your Gemini API key     |
| `Google Docs`          | Google Docs OAuth2          | OAuth flow with Docs + Drive scopes                     |
| `OpenRouter`           | OpenRouter API              | Your OpenRouter API key                                 |

After import, open each of the four HTTP Request nodes (`Gemini Upload Start`, `Gemini Upload Bytes`, `Gemini Poll`, `Gemini Generate Content`) and select the `Gemini API Key` header-auth credential — the MCP doesn't auto-bind these. Telegram and Google Docs nodes auto-bind on first open if a matching credential name exists.

## One-time setup: the "Video Analyses" folder

1. In Google Drive, create a folder called **Video Analyses** (any name works, but it should match the SOP for consistency).
2. Open the folder in the browser. The URL looks like `https://drive.google.com/drive/folders/1AbCdEfGhIjKlMnOp`.
3. Copy the ID after `/folders/`.
4. In n8n, open the **Create Google Doc** node and paste the ID into the `Folder ID` field (the workflow is exported with it blank).

## Import steps (first time only)

1. Copy `workflows/video-analysis-agent.n8n.json` to clipboard.
2. In local n8n → **Workflows → Import from File** and pick the JSON.
3. After import, verify:
   - All four credential slots resolved (Telegram, Google Docs, OpenRouter). Bind Gemini API Key manually on the four HTTP nodes that call Gemini (Upload Start, Upload Bytes, Poll, Generate Content).
   - Pasted the Drive folder ID into Create Google Doc.
4. Activate the workflow. The Telegram Trigger webhook will register with Telegram automatically on activation.

## Updating the workflow after first import

Do NOT delete + re-import for every tweak — credential bindings and the Drive folder ID are lost each time. Instead, sync the JSON directly via the n8n REST API:

```bash
node .tmp/sync_workflow.js workflows/video-analysis-agent.n8n.json
```

This reads `N8N_BASE_URL` + `N8N_API_KEY` from `.env`, PUTs the JSON to `/api/v1/workflows/<id>`, and leaves credentials + folder ID bindings intact. Just refresh the n8n canvas in your browser after it reports success.

## Verification

Static verification (safe, no API calls):
```bash
node .tmp/verify_video_workflow.js
```
Expected: `Total: 41 passed, 0 failed` and a sample Telegram message dump at the end.

End-to-end happy paths:
- **YouTube link** — send `https://www.youtube.com/watch?v=dQw4w9WgXcQ` to the bot. Verify a doc appears in the folder and the bot replies with an `Open analysis` button. No Gemini upload/poll path runs.
- **Short video** (under ~30s, under 20MB) — record or share a quick `.mp4`. The upload + first poll iteration completes within ~15s and the rest proceeds identically.
- **Medium/long video** (minutes) — Gemini reports `PROCESSING` on early polls; the loop keeps waiting 10s and re-polling until the state flips to `ACTIVE`, then proceeds. No hard timeout.

End-to-end failure paths:
- Send a **plain text** with no YouTube URL → bot replies `Send a video file or a YouTube link to get an analysis.`
- Send a video **over 20MB** → Telegram's `getFile` returns a "file too big" error at `Telegram Get File`; n8n surfaces the error. Recommended: send the same video as a YouTube link instead, or upload it to YouTube (unlisted) first.
- If Gemini returns `FAILED` for the uploaded file, the loop exits (the gate only loops on `PROCESSING`) and `Gemini Generate Content` surfaces the error downstream instead of spinning forever.

## Known quirks and constraints

- **Telegram Bot API download cap: 20 MB.** This is a hard limit of the cloud Bot API. The workaround built into this workflow is the YouTube branch — for anything larger, upload to YouTube (unlisted is fine) and send the link.
- **Gemini 2.5 Pro accepts YouTube URLs directly** via `file_data.file_uri` with `mime_type: "video/*"`. This skips the Files API entirely.
- **Gemini Files API processing.** After upload, the file sits in `PROCESSING` state for a few seconds to a few minutes depending on length. The workflow polls every 10 seconds in a self-loop (`Wait 10s -> Gemini Poll -> Still Processing?`) and exits the loop only when Gemini reports a **terminal** state — `ACTIVE` (proceeds to Generate Content) or `FAILED` (Generate Content will surface the error). Any other value (`PROCESSING`, `STATE_UNSPECIFIED`, or a missing `state` field) keeps the loop running. Checking for terminal states specifically — rather than "anything not PROCESSING" — is what prevents premature exits that let a still-processing file through to Generate Content (which would error with "The File X is not in an ACTIVE state and usage is not allowed"). No hard timeout — long clips just poll longer.
- **The resumable upload returns its upload URL in a response header**, not a body field. `Gemini Upload Start` has `fullResponse: true` so that the downstream node can read `$json.headers["x-goog-upload-url"]`.
- **HTTP Request nodes drop incoming binary.** After `Gemini Upload Start` runs, the Telegram video binary is no longer on the current item. An `Attach Binary` Code node re-attaches `$('Telegram Get File').first().binary` while preserving the upload-URL headers, so `Gemini Upload Bytes` can find `$binary.data`. Same pattern used by the OCR Invoice Agent's `Attach Binary` node before Google Drive Upload.
- **Gemini sometimes wraps JSON in markdown fences or emits trailing commas.** `Parse Analysis` fence-strips, attempts strict parse, then trailing-comma + smart-quote repair, then brace-match recovery. Mirrors the pattern from OCR Invoice Agent.
- **Google Docs "create" has no body parameter.** The workflow uses two Google Docs nodes: `Create Google Doc` makes an empty doc in the target folder, then `Write To Google Doc` inserts the markdown body via a `text.insert` action.
- **YouTube URLs with timestamps (`&t=120s`) are accepted** but Gemini may ignore the timestamp fragment. The regex in `Parse Input` captures the full URL including any query parameters.

## Edit points

If you need to tweak behavior, these are the most common edits:

- **Change the model** → `Gemini Generate Content` node URL (`gemini-2.5-pro` → `gemini-2.5-flash` for faster/cheaper).
- **Change the OpenRouter model** → `OpenRouter Chat Model` node `model` field.
- **Adjust polling interval** → `Wait 10s` node (the one node in the poll loop).
- **Change the analysis prompt** → `Gemini Generate Content` node `jsonBody` (the `text` part).
- **Change the narrative style** → `Narrative Agent` node `systemMessage`.
- **Change the Telegram reply format** → `Build Telegram Message` node `jsCode`.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `Telegram Get File` errors "file too big" | Video > 20MB | Ask user to send a YouTube link |
| `Gemini Upload Start` 401/403 | Bad API key | Check `Gemini API Key` credential (value: raw key, name: `x-goog-api-key`) |
| `Gemini Upload Bytes` errors "missing upload URL" | `Gemini Upload Start` didn't return `fullResponse` | Check that node's `options.response.response.fullResponse === true` |
| `Gemini Upload Bytes` errors "input data to contain a binary file 'data', but none was found" | HTTP Request node dropped the binary from Telegram Get File | Ensure the `Attach Binary` Code node sits between Gemini Upload Start and Gemini Upload Bytes — its jsCode should be `const gf = $('Telegram Get File').first(); return { json: $input.item.json, binary: gf?.binary || {} };` |
| Gemini upload returns `400 Bad Request` ("please check your parameters") | Content-Length headers built from `$binary.data.fileSize` — that's a formatted **string** like `"10.3 MB"`, not raw bytes. The HTTP library rejects non-numeric header values | The `Prepare Upload` Code node between Telegram Get File and Gemini Upload Start computes the true byte count and exposes it as `$json.file_byte_length`. Upload Start references `$json.file_byte_length`. Upload Bytes has no manual Content-Length header — n8n's binaryData POST computes it automatically |
| Gemini Poll returns `state: "FAILED"` with `error.message: "The file failed to be processed."` and `sizeBytes: "9"` | The local n8n runs in filesystem-v2 binary mode (`N8N_DEFAULT_BINARY_DATA_MODE=filesystem-v2`). In that mode `$binary.data.data` is the literal string `"filesystem-v2"` (13 chars), not base64. Decoding that as base64 yields 9 bogus bytes, which `Prepare Upload` then announces as the Content-Length. Gemini accepts 9 bytes of nothing and fails processing. | `Prepare Upload` must prefer the numeric `bin.bytes` field over decoding `bin.data` as base64. Regression-tested by `[4d2b]` in `.tmp/verify_video_workflow.js`. If you ever see `sizeBytes: "9"` in the Gemini Poll response, it's this bug re-emerging — check `Prepare Upload`'s code. |
| Poll loop runs for a very long time | Large/long video — Gemini may need several minutes | Normal; the loop has no timeout. If you want a ceiling, add a counter or a parallel Wait branch that errors out. Sending a YouTube link is usually faster for very long videos. |
| Poll loop runs forever | Gemini never reached a terminal state (`ACTIVE` / `FAILED`) — often an upload that didn't register, or an authentication issue masking the real error | Open the `Gemini Poll` node's last execution and inspect the response body. If it's not a File resource, the upstream Upload Start/Bytes path didn't actually upload the file. Check credentials and the x-goog-upload-url plumbing. |
| Gemini Generate Content errors "The File X is not in an ACTIVE state and usage is not allowed" | The poll loop exited before Gemini finished processing. Should not happen with the current gate (it waits for `ACTIVE` or `FAILED`), but may reappear if someone flips the gate to check only `state === "PROCESSING"` — then `STATE_UNSPECIFIED` or a missing `state` would falsely exit | Keep the gate as two `notEquals` conditions (state ≠ ACTIVE AND state ≠ FAILED → loop). Verify via `node .tmp/verify_video_workflow.js` — test [3c] dry-runs ACTIVE/PROCESSING/FAILED/STATE_UNSPECIFIED/empty. |
| `Parse Analysis` throws "Could not parse" | Gemini returned text that isn't JSON (often a safety block) | Check `finishReason` in the thrown error; adjust prompt or model |
| `Write To Google Doc` errors "The resource you are requesting could not be found" / "Requested entity was not found" | `documentURL` references the wrong field — Google Docs v2 `create` returns `id` (drive#file shape), NOT `documentId` | Use `={{ $json.id }}`. Downstream consumers (e.g. Code nodes building doc URLs) must also read `.id`, not `.documentId` |
| Google Doc is empty | `Write To Google Doc` didn't run (disconnected) or `documentURL` expression broke | Check connection and that it refers to `$json.id` from Create Google Doc |
| Telegram reply lacks a button | `reply_markup` expression blank | Check `Build Telegram Message` output actually set `reply_markup` to a JSON string |
