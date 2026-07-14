# Appwrite Setup for ShortTube

Do this once in the Appwrite Console before the JS modules will work.

## 1. Project
Create a project → copy its **Project ID** into `js/appwrite-config.js` (`PROJECT_ID`).

## 2. Auth settings (Auth → Security)
- Password minimum length: **8**
- Session length: **30 days**
- Max sessions per user: **4**
- Enable methods: **Email/Password**, **Email OTP**, **Phone (SMS)** — phone requires an SMS provider (Twilio, MessageBird, etc.) connected under Auth → Settings.

## 3. Database
Create a database with ID `6a69384d083784998fb2`, then these collections:

### `profiles` (Document ID = matching user ID)
| Attribute | Type | Notes |
|---|---|---|
| displayName | string | required |
| avatarFileId | string | nullable |
| bio | string | nullable |
| ageVerified | boolean | nullable, default false — self-attested via checkbox at signup (no date of birth is collected), NOT government-ID verified |

Permissions: Read = Any, Update = User (self only, enforced per-document at creation).

### `videos`
| Attribute | Type | Notes |
|---|---|---|
| title | string | required |
| description | string | |
| thumbnailFileId | string | |
| videoFileId | string | |
| ownerId | string | indexed |
| durationSeconds | integer | |
| engagementScore | integer | default 0 — also shown on cards as the upload's "view count" |
| createdAt | datetime | |
| category | string | optional, nullable — recommended: add this to exactly match onboarding's preference keys (`music`, `gaming`, `comedy`, `tech`, `sports`, `vlogs`, `cooking`, `news`). Without it, `js/feed.js` falls back to keyword-matching the title/description against the selected interests, which is best-effort only. |
| videoHash | string | nullable, indexed — SHA-256 of the uploaded file, used to block exact re-uploads |
| moderationStatus | string | default `'live'`. `'pending'` = held from public feeds pending review (auto-set by a basic upload-time text screen or by report volume); `'removed'` = confirmed violation, permanently hidden. See `js/upload.js` and `netlify/functions/moderate.js` for where a real vision-moderation API would plug in to set this automatically. |

Permissions: Read = Any, Update/Delete = User (owner).
Add an **index** on `ownerId` (for profile stat lookups), `$createdAt` (for feed ordering), and `videoHash` (for duplicate lookups).

### `likes`
| videoId (string, indexed) | userId (string) |

### `comments`
| videoId (string, indexed) | userId (string) | text (string) | createdAt (datetime) |

### `follows`
| followerId (string, indexed) | followingId (string, indexed) |

### `watch_events`
| videoId (string, indexed) | userId (string) | watchedSeconds (integer) | durationSeconds (integer) |

Permissions: Read/Write = User (self only) — this data is per-viewer and shouldn't be public.

### `reports`
| videoId (string, indexed) | reporterId (string) | reason (string) | createdAt (datetime) |

Permissions: Read = Any (or restrict to an admin team, if you have one), Create = Any authenticated user.
Add this collection's ID to `APPWRITE_CONFIG.COLLECTIONS.REPORTS` in `js/appwrite-config.js`. Once a video collects 3+ reports, `js/social.js` automatically flips its `moderationStatus` to `'pending'` so it drops out of Home/Shorts until a human reviews it.

**Note on real automated moderation:** the checks shipped here (exact-file-hash re-upload blocking, a short text denylist, and report-volume auto-hiding) are real and enforced, but they cannot see inside a video's actual frames or audio. Detecting violence or sexual content *in the footage itself* requires connecting a real vision-moderation API (AWS Rekognition, Google Cloud Video Intelligence, Hive Moderation, etc.) — see the detailed wiring notes in `netlify/functions/moderate.js`.

## 4. Storage buckets
Create three buckets with these exact IDs (must match `appwrite-config.js`):
- `avatars` — max file size ~5MB, allowed: jpg/png/webp
- `thumbnails` — max file size ~5MB, allowed: jpg/png/webp
- `videos` — max file size per your plan's limit, allowed: mp4/mov/webm

## 5. Platform
Add your web domain (and `localhost` for dev) under Project Settings → Platforms → Add Web App, or Appwrite will reject requests from origins it doesn't recognize.

That's the whole backend. Nothing else needs a server you manage — Appwrite Cloud handles auth, database, and file storage directly from the browser via the Web SDK.
