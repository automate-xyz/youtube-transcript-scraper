# YouTube Transcript Scraper

[Youtube transcript scraper](https://apify.com/supreme_coder/youtube-transcript-scraper?fpr=ve081&fp_sid=gh_ytt) extracts **existing** YouTube captions (manual or auto-generated)—not audio transcription. It accepts a batch of video URLs, fetches caption tracks through YouTube’s innertube API, and writes structured JSON results (one object per URL).

## How to scrape youtube video transcripts

1. Get a free API Key on [apify](https://console.apify.com/settings/integrations)
2. Go to [Youtube transcript API](https://apify.com/supreme_coder/youtube-transcript-scraper/api?fpr=ve081&fp_sid=gh_ytt) page to view API endpoints and examples
3. Use your API key to make API requests and get data
4. Check `OpenAPI.json` for detailed documentation in OpenAPI format

---

## Configuration (input)

Pass a single JSON config object. Only `urls` is required

### `urls` (required)

| Property | Type | Description |
|----------|------|-------------|
| `urls` | `object[]` | List of videos to scrape. |

Each item must include a `url`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `url` | `string` | yes | Full YouTube URL for one video or Short. Query params (e.g. `&t=256s`) do not affect ID extraction. |

**Supported URL patterns**

| Pattern | Example |
|---------|---------|
| Watch | `https://www.youtube.com/watch?v=VIDEO_ID` |
| Short link | `https://youtu.be/VIDEO_ID` |
| Shorts | `https://www.youtube.com/shorts/VIDEO_ID` |

```json
{
  "urls": [
    { "url": "https://www.youtube.com/watch?v=dQw4w9WgXcQ" },
    { "url": "https://youtu.be/dQw4w9WgXcQ" }
  ]
}
```

### `outputFormat` (optional)

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `outputFormat` | `string` | `"json"` | Controls the **shape** of the `transcript` field on success. |

| Value | `transcript` type | Use when |
|-------|-------------------|----------|
| `json` | `object[]` | You need per-line timestamps (`start`, `duration`) for players, search, or alignment. |
| `text` | `string` | You want one block of plain text (LLMs, summarization, keyword extraction). Aliases: `txt`. |
| `srt` | `string` | SubRip subtitles for NLEs (Premiere, DaVinci, etc.). |
| `vtt` | `string` | WebVTT for HTML5 `<track>` or web players. Alias in code: `webvtt`. |

### `languages` (optional)

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `languages` | `string[]` | `["en"]` | Language codes in **priority order**. The scraper uses the first caption track whose `languageCode` matches (manual tracks win over auto-generated when both match). |

Example: `["es", "en"]` tries Spanish first, then English. If none match, the URL gets a `TranscriptNotFound` error object.

### `translateTo` (optional)

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `translateTo` | `string` | — | Target language code via YouTube’s built-in caption translation (same as the “Translate” option in the YouTube UI). Not every language pair is available per video. |

When set, a source track is resolved using `languages`, then the translated XML is fetched. **`videoDetails` is not included** on this path—only transcript-related fields.

### `preserveFormatting` (optional)

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `preserveFormatting` | `boolean` | `false` | If `true`, keeps inline HTML in caption text (`<b>`, `<i>`, etc.). If `false`, tags are stripped. |

---

## Output

The scraper produces **one JSON object per input URL**. Each object is either a **success** or **error** record—not both.

### Successful record

Returned when captions are found and parsed.

| Field | Type | Always present | Description |
|-------|------|----------------|-------------|
| `inputUrl` | `string` | yes | The URL from `urls[].url`. |
| `language` | `string` | yes | Human-readable track name from YouTube (e.g. `"English (auto-generated)"`). |
| `languageCode` | `string` | yes | Code for the track used (e.g. `"en"`). |
| `isGenerated` | `boolean` | yes | `true` for auto-generated captions; `false` for creator-uploaded tracks. |
| `transcript` | varies | yes | Caption payload; shape depends on `outputFormat` (see below). |
| `snippetCount` | `number` | yes | Number of timed segments. |
| `videoDetails` | `object` | no* | Metadata from YouTube’s player response (*omitted when `translateTo` is set). |

#### `transcript` by format

**`outputFormat: "json"`** (default)—array of segments:

| Field | Type | Description |
|-------|------|-------------|
| `text` | `string` | Caption line content. |
| `start` | `number` | Start time in **seconds** (float). |
| `duration` | `number` | Segment length in seconds. End time ≈ `start + duration`. |

```json
{
  "text": "today I'm going to show you",
  "start": 0.04,
  "duration": 3.359
}
```

**`outputFormat: "text"`**—single string: all snippet texts joined with spaces (no timestamps).

**`outputFormat: "srt"`** / **`"vtt"`**—single string in SubRip or WebVTT syntax (SRT includes cue numbers; VTT includes a `WEBVTT` header).

#### `videoDetails` (when present)

Subset of YouTube’s `videoDetails` object—not a fixed schema. Common fields:

| Field | Type | Description |
|-------|------|-------------|
| `videoId` | `string` | 11-character ID. |
| `title` | `string` | Video title. |
| `lengthSeconds` | `string` | Duration as stringified integer seconds. |
| `shortDescription` | `string` | Description text. |
| `keywords` | `string[]` | Tags/keywords when exposed. |
| `channelId` | `string` | Channel ID. |
| `author` | `string` | Channel display name. |
| `viewCount` | `string` | View count as string. |
| `thumbnail` | `object` | `{ thumbnails: [{ url, width, height }, ...] }`. |
| `isLiveContent` | `boolean` | Whether the item is/was live. |

Extra keys may appear as YouTube changes responses; treat unknown fields as optional.

#### Example (success, JSON)

```json
{
  "inputUrl": "https://www.youtube.com/watch?v=neabI31ofMc",
  "language": "English (auto-generated)",
  "languageCode": "en",
  "isGenerated": true,
  "transcript": [
    { "text": "today I'm going to show you all the", "start": 0.04, "duration": 3.359 }
  ],
  "snippetCount": 185,
  "videoDetails": {
    "videoId": "neabI31ofMc",
    "title": "Example title",
    "lengthSeconds": "412",
    "author": "Channel Name",
    "viewCount": "1088282"
  }
}
```

### Error record

Failures are explicit objects—not empty `transcript` fields.

| Field | Type | Description |
|-------|------|-------------|
| `errorCode` | `string` | Machine-readable code (see table). |
| `error` | `string` | Human-readable message. |
| `inputUrl` | `string` | URL that failed (when known). |
| `videoId` | `string` | Extracted ID when applicable; may be absent for invalid URLs. |

| `errorCode` | Meaning |
|-------------|---------|
| `TranscriptNotFound` | No caption track for the requested `languages`, or captions disabled. |
| `URL_NOT_SUPPORTED` | Host/path is not a supported YouTube video URL. |
| `VIDEO_ID_NOT_FOUND` | URL looked like YouTube but no ID could be parsed. |
| `AgeRestricted` | Video requires sign-in (age gate). |
| `VideoUnavailable` | Removed, private, or otherwise unplayable. |
| `IpBlocked` | Request blocked (bot check / rate limit); retry or rotate proxies. |
| `UNKNOWN_ERROR` | Unhandled exception (`error` may include the underlying message). |

```json
{
  "errorCode": "TranscriptNotFound",
  "error": "No transcripts were found for ...",
  "inputUrl": "https://www.youtube.com/watch?v=XXXXXXXXXXX",
  "videoId": "XXXXXXXXXXX"
}
```

---

## Limits

- Does **not** run speech-to-text; only downloads captions YouTube already publishes.
- Age-restricted and login-walled videos return `AgeRestricted` or related errors.
- Translation availability depends on what YouTube offers per video.
- `translateTo` does not attach `videoDetails` in the current implementation.
- Built-in proxy rotation is used to reduce IP blocks; heavy batching may still hit rate limits.
