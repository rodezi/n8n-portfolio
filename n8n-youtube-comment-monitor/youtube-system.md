# YouTube Comment Monitor & Auto-Reply System

A two-workflow n8n system that monitors YouTube comments across multiple videos, classifies them with AI, and manages team-approved replies via Slack.

---

## Overview

The system is split into two independent n8n workflows:

| Workflow | File | Purpose |
|---|---|---|
| Monitor & Classify | `youtube-comment-editor.json` | Fetches new comments, classifies with GPT, notifies Slack |
| Post Approved Reply | `youtube-post-approved.json` | Receives approved reply via webhook, posts to YouTube |

---

## Workflow 1: YouTube Comment Monitor & AI Classifier

**File:** `youtube-comment-editor.json`

### What it does

Runs on a schedule, reads the full seen-comments list once, then fetches recent comments from every video in the list, deduplicates against the seen list, classifies each new comment with GPT-4o-mini, and sends a Slack notification with the classification and a suggested reply draft.

### Flow

```
Schedule (every 15 min)
  --> Read Seen Comments Sheet      (once per run)
  --> Build Seen IDs Set            (aggregate all rows into 1 item)
  --> Video ID List                 (fan out: 1 item per video)
  --> Fetch YouTube Comments        (per video)
  --> Extract & Filter New Comments (dedup + flatten, all videos)
  --> Mark as Seen in Sheet         (per new comment)
  --> Classify & Draft Reply (GPT)  (per new comment)
  --> Parse AI Response
  --> Log to CommentLog Sheet
  --> Skip Spam/Hate
  --> Notify Slack
```

### Node-by-node breakdown

#### 1. Every 15 Minutes (`schedule-trigger`)
Triggers the workflow every 15 minutes using n8n's Schedule Trigger node.

#### 2. Read Seen Comments Sheet (`read-seen-sheet`)
Reads the entire `SeenComments` tab once at the start of each run. Returns all rows as individual items.

> This is the first node after the trigger — it runs **once per execution**, not once per comment. This was a key architectural fix over the original design.

#### 3. Build Seen IDs Set (`build-seen-ids`)
Aggregates all sheet rows into a **single output item** containing an array of seen comment IDs:

```js
const seenIds = rows.map(r => r.json.commentId).filter(Boolean);
return [{ json: { seenIds } }];
```

This single item is referenced globally downstream via `$('Build Seen IDs Set').first()`. Because this node always outputs exactly one item, `.first()` is safe and correct here.

#### 4. Video ID List (`video-list`)
A Code node that holds the list of YouTube video IDs to monitor. Each ID is emitted as a separate item so the fetch step runs once per video.

```js
const videoIds = ['VIDEO_ID_1', 'VIDEO_ID_2', 'VIDEO_ID_3'];
return videoIds.map(id => ({ json: { videoId: id } }));
```

> **To configure:** Replace the placeholder IDs with your actual YouTube video IDs.

#### 5. Fetch YouTube Comments (`fetch-comments`)
HTTP GET to the YouTube Data API v3 `commentThreads` endpoint. Fetches up to 50 top-level comments per video, ordered by most recent.

- **API Key required:** `YOUR_YOUTUBE_API_KEY`
- **Parameters:** `part=snippet`, `maxResults=50`, `order=time`

#### 6. Extract & Filter New Comments (`extract-filter-new`)
This single node replaces what was originally four separate nodes (Extract Comments, Read Seen Sheet per-comment, Check If Already Seen, Is New Comment? filter). It:

1. Reads the seen IDs built at step 3 via `$('Build Seen IDs Set').first().json.seenIds`
2. Iterates over all video fetch results (`$input.all()`)
3. Flattens each video's comment list
4. Skips any comment whose ID is already in the seen set
5. Returns only new comments as individual items

```js
const seenIds = new Set($('Build Seen IDs Set').first().json.seenIds || []);
const results = [];
for (const item of $input.all()) {
  const comments = item.json.items || [];
  for (const comment of comments) {
    const commentId = comment.snippet.topLevelComment.id;
    if (!seenIds.has(commentId)) {
      results.push({ json: { commentId, videoId, authorName, text, publishedAt, likeCount } });
    }
  }
}
return results; // empty array = no new comments, execution stops cleanly
```

Fields extracted per comment:

| Field | Source |
|---|---|
| `commentId` | `topLevelComment.id` |
| `videoId` | `snippet.videoId` |
| `authorName` | `snippet.authorDisplayName` |
| `text` | `snippet.textDisplay` |
| `publishedAt` | `snippet.publishedAt` |
| `likeCount` | `snippet.likeCount` |

#### 7. Mark as Seen in Sheet (`mark-seen`)
Appends the new comment to the `SeenComments` tab **before** the AI classification step. This is intentional: if GPT or a later step fails, the comment is already recorded as seen and won't be re-processed on the next run.

Columns written: `commentId`, `videoId`, `authorName`, `seenAt`

#### 8. Classify & Draft Reply (GPT) (`classify-comment`)
Sends the comment to GPT-4o-mini with a structured system prompt. The model is instructed to respond in strict JSON only.

**Categories:**
- `QUESTION`
- `POSITIVE_FEEDBACK`
- `NEGATIVE_FEEDBACK`
- `SPAM`
- `COLLABORATION`
- `HATE_SPEECH`
- `OTHER`

**AI response format:**
```json
{
  "category": "QUESTION",
  "confidence": 0.95,
  "reason": "Brief reason",
  "draft_reply": "Your suggested reply"
}
```

- **Model:** `gpt-4o-mini`
- **Temperature:** `0.3` (low, for consistent classification)
- **Credential:** OpenAI API

#### 9. Parse AI Response (`parse-ai`)
Safely parses the raw JSON string from the GPT response. Falls back to `category: OTHER` with a generic reply if parsing fails — the workflow never hard-crashes on a malformed GPT response. Also sets a `skip: true` flag for `SPAM` and `HATE_SPEECH`.

#### 10. Log to CommentLog Sheet (`log-to-sheet`)
Appends a full record to the `CommentLog` tab. Initial status is `PENDING`.

Columns written: `commentId`, `videoId`, `authorName`, `text`, `publishedAt`, `category`, `confidence`, `reason`, `draft_reply`, `status`, `processedAt`

#### 11. Skip Spam/Hate (`skip-spam`)
Filter node. Drops any comment where `skip === true` (SPAM or HATE_SPEECH). These are logged to the sheet but never sent to Slack.

#### 12. Notify Slack (`slack-notify`)
Posts a formatted message to a Slack channel with:
- Video ID and author
- AI category + confidence percentage
- Original comment text
- Suggested reply draft
- Comment ID
- Instructions for approving/editing via the reply webhook

**Credential:** Slack API

---

## Workflow 2: YouTube Post Approved Reply

**File:** `youtube-post-approved.json`

### What it does

A webhook-triggered workflow. When the team approves or edits a reply in Slack, they POST to this workflow's webhook URL. It posts the reply to YouTube, updates the sheet log, and confirms in Slack.

### Flow

```
Webhook (POST /youtube-post-reply)
  --> Validate Input          (validates fields + pre-serializes JSON body)
  --> Post Reply to YouTube   (raw JSON body via OAuth2)
  --> Update Status in Sheet  (PENDING -> POSTED)
  --> Slack: Confirm Posted
  --> Respond OK (HTTP 200)
```

### Node-by-node breakdown

#### 1. Webhook: Approved Reply (`webhook-trigger`)
Listens for a `POST` request at the path `/youtube-post-reply`. Waits for the Respond OK node before closing the connection.

#### 2. Validate Input (`validate-input`)
Validates the request body and pre-serializes the YouTube API payload using `JSON.stringify`. This ensures special characters in the reply text (quotes, newlines, emoji) are properly escaped before the HTTP call.

**Expected request body:**
```json
{
  "commentId": "Ugx...",
  "reply": "Thanks for your question! ..."
}
```

Outputs three fields: `commentId`, `reply`, and `serializedBody` (the pre-built YouTube API JSON string).

#### 3. Post Reply to YouTube (`post-reply`)
HTTP POST to the YouTube Data API v3 `comments` endpoint. Uses `contentType: raw` with `rawContentType: application/json` and passes `$json.serializedBody` as the body. This ensures the `snippet` field is sent as a **nested JSON object**, not a string.

- **Auth:** YouTube OAuth2 (required — API key is not sufficient for write operations)
- **Credential:** `YOUR_YOUTUBE_OAUTH_CREDENTIAL_ID`

> The original implementation sent `snippet` as a JSON string inside a parameters object, which caused a `400 Bad Request` from the YouTube API. The fix is to send the body as raw JSON.

#### 4. Update Status in Sheet (`update-log`)
Updates the matching row in the `CommentLog` tab where `commentId` matches. Sets:
- `status` → `POSTED`
- `postedAt` → current timestamp
- `finalReply` → the actual reply text sent

#### 5. Slack: Confirm Posted (`slack-confirm`)
Posts a confirmation to the Slack channel:
```
Reply posted to YouTube!
Comment ID: `Ugx...`
Reply: Thanks for your question! ...
```

#### 6. Respond OK (`respond-ok`)
Returns a `200 OK` JSON response to the caller:
```json
{ "success": true, "commentId": "Ugx..." }
```

---

## Google Sheets Structure

Create a single Google Sheet with two tabs:

### Tab 1: `SeenComments`

| Column | Description |
|---|---|
| `commentId` | YouTube comment ID (dedup key) |
| `videoId` | YouTube video ID |
| `authorName` | Display name of commenter |
| `seenAt` | ISO timestamp when first processed |

### Tab 2: `CommentLog`

| Column | Description |
|---|---|
| `commentId` | YouTube comment ID |
| `videoId` | YouTube video ID |
| `authorName` | Display name of commenter |
| `text` | Full comment text |
| `publishedAt` | When the comment was posted on YouTube |
| `category` | AI classification |
| `confidence` | AI confidence score (0–1) |
| `reason` | AI reasoning |
| `draft_reply` | AI-suggested reply |
| `status` | `PENDING` or `POSTED` |
| `processedAt` | When the workflow processed it |
| `postedAt` | When the reply was posted (set by Workflow 2) |
| `finalReply` | Actual reply text posted (set by Workflow 2) |

---

## Setup Checklist

### Credentials required

| Service | Type | Used in |
|---|---|---|
| YouTube Data API v3 | API Key | Workflow 1 — fetching comments |
| YouTube Data API v3 | OAuth2 | Workflow 2 — posting replies |
| Google Sheets | OAuth2 | Both workflows |
| OpenAI | API Key | Workflow 1 — GPT classification |
| Slack | OAuth / Bot Token | Both workflows |

### Placeholders to replace

| Placeholder | Where | What to replace with |
|---|---|---|
| `YOUR_YOUTUBE_API_KEY` | Workflow 1, Fetch Comments node | Your YouTube Data API v3 key |
| `YOUR_GOOGLE_SHEET_ID` | Both workflows, all Sheets nodes | The ID from your Google Sheet URL |
| `YOUR_GSHEETS_CREDENTIAL_ID` | Both workflows | Your n8n Google Sheets credential ID |
| `YOUR_OPENAI_CREDENTIAL_ID` | Workflow 1, GPT node | Your n8n OpenAI credential ID |
| `YOUR_SLACK_CREDENTIAL_ID` | Both workflows | Your n8n Slack credential ID |
| `YOUR_SLACK_CHANNEL_ID` | Both workflows | Target Slack channel ID |
| `YOUR_YOUTUBE_OAUTH_CREDENTIAL_ID` | Workflow 2, Post Reply node | Your n8n YouTube OAuth2 credential ID |
| `YOUR_ERROR_WORKFLOW_ID` | Both workflows settings | ID of your n8n error-handling workflow |
| `VIDEO_ID_1`, `VIDEO_ID_2`, ... | Workflow 1, Video ID List node | Your actual YouTube video IDs |

### Step-by-step

1. Create the Google Sheet with the two tabs described above. Note the Sheet ID from the URL.
2. Set up credentials in n8n: YouTube API Key, YouTube OAuth2, Google Sheets OAuth2, OpenAI, Slack.
3. Import `youtube-comment-editor.json` into n8n. Replace all placeholders. Add your video IDs to the Video ID List node. Activate the workflow.
4. Import `youtube-post-approved.json` into n8n. Replace all placeholders. Note the webhook URL generated by n8n (copy it from the Webhook node).
5. In Slack, when a notification arrives, approve a reply by sending a POST to the webhook URL:
   ```bash
   curl -X POST https://your-n8n-instance/webhook/youtube-post-reply \
     -H "Content-Type: application/json" \
     -d '{"commentId": "Ugx...", "reply": "Thanks for commenting!"}'
   ```

---

## Error Handling

- **Both workflows** have `errorWorkflow` configured in settings. Point `YOUR_ERROR_WORKFLOW_ID` to an n8n error-handling workflow (e.g., one that sends a Slack alert on failure).
- **Dedup safety:** Comments are written to `SeenComments` *before* the AI classification step. If GPT or Sheets fails downstream, the comment won't be re-fetched and double-processed on the next run.
- **AI parse failures:** The Parse AI Response node catches JSON parse errors and falls back to `category: OTHER` with a generic reply, so the workflow never hard-crashes on a malformed GPT response.
- **Spam/Hate filtering:** SPAM and HATE_SPEECH comments are logged to the sheet but silently dropped before Slack notification.
- **Webhook validation + body safety:** Workflow 2 throws an explicit error if `commentId` or `reply` is missing. The reply text is pre-serialized with `JSON.stringify` before being sent to YouTube, so special characters never break the API request.
- **No new comments:** If `Extract & Filter New Comments` finds nothing new, it returns an empty array and the workflow stops cleanly — no errors, no notifications.

---

## How the approval loop works

```
[n8n Workflow 1 — runs every 15 min]
  Detects new comment --> Classifies with GPT --> Sends Slack message
                                                         |
                                            Team reads notification,
                                            edits reply if needed,
                                            sends POST to webhook
                                                         |
                                              [n8n Workflow 2]
                                           Posts reply to YouTube
                                           Updates sheet to POSTED
                                           Confirms in Slack
```

The two workflows are decoupled by design. Workflow 1 runs continuously on schedule. Workflow 2 only runs when explicitly triggered by a human action, ensuring no reply is ever posted without team approval.

---

## Bugs fixed from original implementation

| # | Workflow | Severity | Issue | Fix |
|---|---|---|---|---|
| 1 | Monitor | Critical | `$('Extract Comments').first()` always referenced comment #1 — all other comments were evaluated with the wrong ID | Replaced with `$('Build Seen IDs Set').first()` referenced inside one combined node that processes all comments at once |
| 2 | Monitor | Architecture | `Read Seen Comments Sheet` ran once per comment (N API calls), causing item-count mismatch in the dedup check | Moved sheet read to before the video loop; added `Build Seen IDs Set` to aggregate rows into 1 item read globally |
| 3 | Monitor | Minor | No per-comment dedup filter node; architecture depended on broken item-pairing across mismatched item counts | Merged Extract, Read, Check, and Filter into single `Extract & Filter New Comments` node with correct logic |
| 4 | Post Reply | Critical | `snippet` was sent as a JSON string in the request body — YouTube API returned `400 Bad Request` | Changed body mode to `contentType: raw` with pre-serialized JSON via `JSON.stringify` in the Validate Input node |
| 5 | Post Reply | Minor | No `errorWorkflow` configured — failures were silent | Added `errorWorkflow` to settings |
