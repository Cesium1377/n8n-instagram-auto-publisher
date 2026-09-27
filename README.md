# Instagram Folder Auto-Publisher (n8n)

Drop a video or photo into a folder. It gets an AI-written caption, gets published to Instagram, and gets moved out of the way automatically — once a day, or on demand.

An [n8n](https://n8n.io) workflow that watches a local folder, picks the oldest unpublished file, writes a caption for it with Google Gemini, publishes it to an Instagram Professional account via the Meta Graph API, and files it away so it's never posted twice. No database, no external queue — the filesystem itself is the state.

## What it does

1. You copy `.mp4` / `.mov` / `.jpg` / `.jpeg` files into a root folder (**never** into a subfolder — the workflow manages those itself).
2. On each run, the workflow:
   - creates `QUEUE` and `PUBLISHED` subfolders under the root if they don't exist yet
   - moves any new, fully-copied files from the root into `QUEUE`
   - picks exactly **one** file from `QUEUE` — the one that's been waiting longest
   - asks Gemini to look at (or watch/listen to) the file and write an Instagram-appropriate caption
   - uploads the file to a public temporary host (Instagram's servers can't reach a file on your local disk, so it needs a public URL to fetch from)
   - creates an Instagram Reel or Photo container with that URL, polls until Instagram finishes processing it, and publishes it
   - **only after Instagram confirms with a real media ID**, moves the original file into `PUBLISHED`
3. Files in `PUBLISHED` are never looked at again by the workflow. That's the entire duplicate-prevention mechanism — deliberately simple, no extra moving parts.
4. If anything fails, the file is left in `QUEUE` (never deleted, never marked as posted) and gets retried on a later run, behind the rest of the queue.

## Why it's built this way

- **No database.** The folder structure itself (`QUEUE` vs `PUBLISHED`) is the only state that matters. A file can only be published once because published files are physically moved out of the folder the workflow ever looks at.
- **A public temp host, not local file access.** Meta's servers fetch media from a URL — they cannot read a file on your machine. This workflow uploads each file to a public host first and hands Instagram that URL. See [Temporary media hosting](#temporary-media-hosting) below for why, and what to swap it for if you'd rather not depend on a third-party host.
- **Retry-friendly by construction.** Every file's "last touched" time is bumped the moment it's picked, so a file that fails doesn't get retried immediately (and potentially get stuck failing forever) — it goes to the back of the line instead.

## Requirements

- A working [n8n](https://n8n.io) instance (self-hosted or n8n Cloud), version **1.6x or newer**. This was built and tested against a portable Windows self-hosted install running **n8n 2.35.7** — the node types used are stable and should work on other recent versions, but double-check node parameters after import if you're on a much older version.
- An **Instagram Professional account** (Business or Creator) set up for the Instagram API with Instagram Login (see [Instagram / Meta setup](#instagram--meta-setup) below).
- A **Google Gemini API key** (from [Google AI Studio](https://aistudio.google.com/)) — the free tier is enough for captioning.
- Node.js's built-in modules available to n8n's Code nodes (`fs`, `path`) — see [n8n configuration](#n8n-configuration) below.

## Repository contents

```
.
├── README.md                                     <- you are here
├── LICENSE
├── .gitignore
├── .env.example                                   <- n8n process environment variables (not secrets — a template)
└── workflow/
    └── instagram-folder-auto-publisher.json        <- the importable n8n workflow
```

## Quick start

1. **Import the workflow.** In n8n: **Workflows → Import from File** → select `workflow/instagram-folder-auto-publisher.json`.
2. **Create two credentials** in n8n (Settings → Credentials → Create):
   - A **Header Auth** credential (used by the node named `Instagram Access Token (Header Auth)` placeholder in the imported workflow):
     - Header Name: `Authorization`
     - Header Value: `OAuth YOUR_INSTAGRAM_ACCESS_TOKEN`
   - A **Google Gemini(PaLM) Api** credential (used by the `Gemini API Credential` placeholder):
     - API Key: `YOUR_GEMINI_API_KEY`
   - Then open each HTTP Request node and each Gemini node in the imported workflow and assign the matching credential (n8n will flag any node with an unresolved credential — it's normal right after import).
3. **Open the `Config` node** and fill in every value — see [Configuration](#configuration) below. At minimum you must set `ROOT_DIR` and `IG_ACCOUNT_ID`.
4. **Create the root folder** on disk (e.g. `C:\Instagram-Automation`) if it doesn't already exist. You don't need to create `QUEUE` or `PUBLISHED` — the workflow creates those itself on first run.
5. **Test it**: drop one file into the root folder, wait about a minute, then run the workflow manually (click the `Test: Run Now` node, or use n8n's "Execute workflow" button). Watch the execution log — a full run typically takes 1.5–3 minutes, most of it spent uploading to the temp host and waiting for Instagram to finish processing.
6. **Activate the workflow** once a manual test succeeds, so the `Daily Schedule` trigger takes over.

## Instagram / Meta setup

This workflow uses **Instagram API with Instagram Login** (an "IGAA…"-prefixed access token obtained directly from Instagram's own API setup / business login flow), talking to `graph.instagram.com` — **not** the older Facebook-Page-linked Graph API flow. You'll need:

1. A Meta developer account and an app set up for the Instagram API (see [Meta's Instagram API documentation](https://developers.facebook.com/docs/instagram-platform)).
2. An Instagram **Professional** account (Business or Creator) connected through Instagram's own login-based authorization flow.
3. The resulting **access token** (starts with `IGAA`) and your account's **Instagram Professional Account ID** (a numeric ID, shown during that setup flow).
4. Enough permissions/scopes granted to create media containers and publish (`instagram_business_content_publish` and related — Meta's setup flow will walk you through this).

Two things worth knowing before you rely on this in production:

- **Token expiry.** Access tokens issued this way are typically short- or medium-lived and need periodic refreshing. Budget for regenerating the token occasionally and updating the Header Auth credential's value.
- **`video_url` may be required even for the "resumable" upload flow.** Meta's documentation describes `video_url` as optional when using `upload_type=resumable` (direct binary upload). In practice, some Instagram Login tokens reject that flow with `"The parameter video_url is required"`. This workflow sidesteps the issue entirely by always uploading to a public URL first and passing that — for both photos and videos — so it works regardless of which behavior your token has.

If you're using a classic Facebook Page-linked token (`EAA…`-prefixed) instead, you'll likely need a different Graph host (`graph.facebook.com`) and possibly different parameters — this workflow was built and tested specifically against the Instagram Login flow.

## Gemini configuration

- Model is set via `Config.GEMINI_MODEL` (default: `models/gemini-3.5-flash-lite`) — not hardcoded in the Gemini nodes, so you can swap models by changing one value.
- Both Gemini nodes send the raw media file as binary input and ask for a caption based only on what's actually in the media (see the full prompt rules in the `Config` node's `CAPTION_INSTRUCTIONS` field — edit that field directly to change tone, length, hashtag rules, language, etc.).
- `CAPTION_LANGUAGE` and optional `ACCOUNT_CONTEXT` (freeform tone/background notes for Gemini) are also set in `Config`.

## Temporary media hosting

Instagram's Graph API needs a **public** URL to fetch media from — it can't reach a file on your local disk, and (per the note above) some tokens reject the alternative direct-upload flow entirely. So every file is uploaded to a public host first, and that URL is handed to Instagram.

This template is configured (via `Config.TEMP_MEDIA_HOST_URL`) to use **catbox.moe's anonymous upload API** by default:

- No account or API key required.
- `POST` to the URL with a `multipart/form-data` body containing `reqtype=fileupload` and the file itself as `fileToUpload`.
- The response is the raw, directly-fetchable file URL as plain text — no JSON wrapper, no landing page to work around.

**Known limitation:** anonymous catbox.moe uploads aren't automatically deleted, so each published file's temporary copy stays reachable at its (unguessable) catbox.moe URL indefinitely unless you delete it yourself.

**If you'd rather not depend on a third-party host**, swap `TEMP_MEDIA_HOST_URL` (and, if the response format differs, the expression in the `Build Public Media URL` node) for:
- your own S3/R2/Cloud Storage bucket with public read access and a short lifecycle policy, or
- a cloud tunnel (e.g. Cloudflare Tunnel, ngrok) exposing the file directly from your own machine, or
- any other host that returns a directly-fetchable URL from an upload API.

Whatever you choose, confirm it supports HTTP range requests — some hosts that don't can cause Instagram's video processing to fail with a generic error even though the upload itself succeeds.

## Configuration

Every tunable value lives in the **`Config`** node (a Set node) — you should not need to edit any other node's parameters for normal use.

| Field | What it's for | Example / default |
|---|---|---|
| `ROOT_DIR` | The folder you drop files into. Must exist; `QUEUE`/`PUBLISHED` are created under it automatically. | `C:\Instagram-Automation` |
| `IG_ACCOUNT_ID` | Your Instagram Professional Account ID. | `YOUR_INSTAGRAM_ACCOUNT_ID` |
| `GRAPH_HOST` | Graph API host. | `graph.instagram.com` |
| `GRAPH_VERSION` | Graph API version. | `v25.0` |
| `SHARE_TO_FEED` | Whether Reels also post to the main feed. | `true` |
| `VIDEO_EXTENSIONS` | Comma-separated extensions treated as video. | `.mp4,.mov` |
| `IMAGE_EXTENSIONS` | Comma-separated extensions treated as photo. | `.jpg,.jpeg` |
| `MIN_FILE_AGE_SECONDS` | How long a file must sit untouched in the root before it's considered "done copying" and safe to move into `QUEUE`. | `60` |
| `STATUS_CHECK_SECONDS` | How long to wait between Instagram processing-status checks. | `30` |
| `MAX_STATUS_CHECKS` | How many times to check before giving up on a stuck upload. | `20` (10 minutes total) |
| `TEMP_MEDIA_HOST_URL` | Upload endpoint for the public temp host — see [above](#temporary-media-hosting). | `https://catbox.moe/user/api.php` |
| `GEMINI_MODEL` | Gemini model used for captioning. | `models/gemini-3.5-flash-lite` |
| `CAPTION_LANGUAGE` | Language Gemini writes the caption in. | `English` |
| `ACCOUNT_CONTEXT` | Optional freeform notes to steer tone (not used for facts). | *(empty)* |
| `CAPTION_INSTRUCTIONS` | The full caption-writing prompt/rules. Edit freely. | *(see the node)* |

The daily posting time itself is **not** in `Config` — it's set directly on the **`Daily Schedule`** trigger node (default: 19:00 in the workflow's configured timezone; set your own timezone under Workflow Settings → Timezone).

## Folder structure

```
<ROOT_DIR>\                 e.g. C:\Instagram-Automation\
├── (drop new files here — nowhere else)
├── QUEUE\                  auto-created; files waiting to be posted
└── PUBLISHED\              auto-created; posted files land here and are never looked at again
```

There is intentionally **no `FAILED` folder**. A file that fails to publish simply stays in `QUEUE` and is retried on a later run (see below) — kept deliberately simple, no extra state to manage.

## File movement, retries, and failure behavior

- **Root → QUEUE**: on every run, any matching file sitting directly in the root that's non-empty and hasn't been modified in the last `MIN_FILE_AGE_SECONDS` gets moved into `QUEUE`, with automatic `(2)`, `(3)`… renaming on name collisions.
- **One file per run**: the oldest file (by modified time) currently in `QUEUE` is picked.
- **Retry rotation**: the instant a file is picked, its modified time is bumped to "now" — so if this run fails, the file goes to the *back* of the queue rather than being retried immediately (and potentially failing the same way forever). The rest of the queue gets a turn first.
- **Never republished**: only `QUEUE` is ever scanned for candidates. `PUBLISHED` is never scanned, so nothing that's ever successfully reached it can be picked again.
- **Move to `PUBLISHED` only on confirmed success**: the file is moved only after Instagram actually returns a media ID from the publish call. If publish fails or returns nothing, the file is left exactly where it was.
- **If the move itself fails** after a successful post (e.g. a brief file lock), it's retried a few times; if it still can't move, the error explicitly says the post is already live and that you need to move the file by hand — this exists specifically so a stuck lock can never cause a double-post.
- **If Instagram never finishes processing** (explicit error/expired status, or still processing after `MAX_STATUS_CHECKS`), the file is left untouched in `QUEUE` and retried on a later run.
- **Transient failures** (network blips, rate limits) on the Gemini and HTTP Request nodes retry automatically a few times before the run is allowed to fail.

## n8n configuration

The Code nodes in this workflow use Node.js's built-in `fs` and `path` modules directly (not the n8n sandbox's restricted expression syntax), so your n8n instance needs to allow that:

```
NODE_FUNCTION_ALLOW_BUILTIN=fs,path
```

If you self-host and set `N8N_RESTRICT_FILE_ACCESS_TO` (recommended — it limits which folders n8n's file nodes can touch), make sure your chosen root folder is included:

```
N8N_RESTRICT_FILE_ACCESS_TO=C:\Instagram-Automation
```

See `.env.example` for a template of these and other relevant n8n environment variables. **Note:** these are n8n's own *process* environment variables, set wherever you start n8n (a `.env` file, your shell, a Docker Compose file, etc.) — n8n workflows don't read `.env` files themselves, which is why the workflow's own settings live in the `Config` node instead.

## Testing

1. Drop one file into the root folder and wait at least `MIN_FILE_AGE_SECONDS` after the copy finishes.
2. Run the workflow manually (`Test: Run Now`).
3. Watch the execution log — each node is listed as it runs.
4. Confirm success: the final node (`Move File to PUBLISHED`) should output `status: "published"` with a real Instagram media ID, and the file should have physically moved from `QUEUE` to `PUBLISHED`.
5. If it fails, the log names the exact node and error — most issues trace back to either an unresolved credential, a Gemini API error, a temp-host upload timeout, or Instagram rejecting the media (the `Instagram Rejected Media` node's error message echoes Instagram's own status).

## Known limitations

- Every post's media briefly transits a third-party public host — unavoidable given how the Graph API fetches media (see [Temporary media hosting](#temporary-media-hosting)).
- No dead-letter/failed-folder handling beyond "stays in QUEUE and cycles to the back of the line" — a file that will genuinely never succeed keeps getting retried indefinitely rather than being flagged and set aside. Add a failure-count mechanism yourself if you need that.
- Exactly one post per run — there's no backlog catch-up logic. If the schedule doesn't run for several days, the next run still only posts one file.
- Access tokens expire; this is a platform limitation, not something the workflow can work around.

## License

MIT — see [`LICENSE`](./LICENSE). Use, modify, and redistribute freely.
