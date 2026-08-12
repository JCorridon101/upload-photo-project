# upload-photo-project

A self-hosted system for uploading photos from a phone directly to a personal computer, without relying on a third-party cloud service (iCloud, Google Photos, etc.). The backend is Python end-to-end; the mobile client is also Python where feasible.

## 1. Goals

- Upload photos (and eventually video) from a phone to a home computer in a couple of taps.
- No third-party cloud storage — files land on disk under the user's control.
- Keep as much of the stack in Python as possible, front end included.
- Work over the home network by default, with an option to upload from anywhere.

### Non-goals (for now)

- Multi-user / account system — this is a single-owner personal tool.
- Photo editing, albums, sharing links, or a Google-Photos-style gallery UI.
- iOS App Store / Google Play distribution.

## 2. High-Level Architecture

```
 ┌────────────┐        HTTPS (multipart upload)       ┌───────────────────────┐
 │   Phone    │  ───────────────────────────────────▶ │   Personal Computer    │
 │ (client)   │                                        │                        │
 │            │ ◀───────────────────────────────────  │  FastAPI app (uvicorn) │
 └────────────┘        JSON status / thumbnails        │  ├─ Auth (API key)    │
                                                         │  ├─ Upload endpoint   │
        Tailscale / home Wi-Fi for connectivity         │  ├─ SQLite metadata   │
                                                         │  └─ Photos on disk    │
                                                         └───────────────────────┘
```

The phone is a thin client that authenticates with an API key and POSTs photos as multipart form data. The computer runs a Python API server that validates, stores, and indexes each file.

## 3. Tech Stack

### 3.1 Backend API (runs on the personal computer)

| Concern | What it is / why the project needs it | Library / Framework | Why this pick |
|---|---|---|---|
| Web framework | The core server that listens for HTTP requests from the phone and turns them into Python function calls. Every other backend piece plugs into this. | **FastAPI** | Async, native file-upload support, automatic OpenAPI docs — easy to test from the phone or `curl` while building the client. |
| ASGI server | FastAPI defines the app but doesn't run it — something has to actually open a socket, accept connections, and hand requests to the app. | **uvicorn** | Standard partner for FastAPI; simple to run as a long-lived process. |
| Data validation | Every upload arrives as untrusted bytes/text from the phone; something must check the shape of the metadata (timestamp, device name, etc.) before the app trusts it. | **Pydantic** | Ships with FastAPI; validates upload metadata (device, timestamp, GPS, etc.) with almost no boilerplate. |
| Database ORM | The app needs to read/write photo metadata as Python objects instead of hand-writing SQL for every query. | **SQLModel** (or SQLAlchemy directly) | Combines Pydantic + SQLAlchemy in one model definition; fits naturally with FastAPI's request/response models. |
| Database | Metadata (filenames, timestamps, hashes) needs to persist across server restarts and be queryable, unlike files on disk alone. | **SQLite** | Zero-config, file-based — fits a single-machine personal project; no separate DB server to run. |
| Image handling | Uploaded photos need thumbnails for quick browsing, and EXIF data (capture time, orientation) needs to be read out of the file. | **Pillow** | The standard Python imaging library; thumbnail generation, format conversion, EXIF reading all in one package. |
| iPhone photo support | Most of what actually lands in this system will be iPhone photos, and iPhones shoot HEIC/HEIF by default. | **pillow-heif** | Pillow can't decode HEIC/HEIF out of the box; this plugin adds that codec so iPhone photos don't fail to process. |
| File-type sniffing | A client could send anything with a `.jpg` extension; the server shouldn't trust the filename when deciding what a file actually is. | **python-magic** | Verifies uploaded bytes are actually image data (via file signature/magic bytes) before storing or processing them. |
| Duplicate detection | Phones re-sync and retry uploads; without dedup, the same photo can pile up multiple times on disk. | **imagehash** | Perceptual hashing detects "visually the same photo" even if the file bytes differ slightly, so re-uploads can be skipped. |
| Background work | Thumbnailing and hashing take real time; doing them inline would make the phone wait longer than necessary for an upload to "finish." | **FastAPI `BackgroundTasks`** (start simple) → **Celery** or **RQ** (if a real queue is needed later) | Lets the API respond "upload received" immediately while the slower processing happens after the response is sent. |
| Config / secrets | The API key and photo storage path shouldn't be hard-coded or committed to git, since this repo will be pushed to GitHub. | **pydantic-settings** + `.env` | Loads settings from environment variables/`.env` with validation, keeping secrets out of source control. |
| Testing | Bugs in the upload path are much cheaper to catch by hitting the API directly than by testing through an actual phone every time. | **pytest** + **httpx** (`AsyncClient`) | Lets you exercise every endpoint (including file uploads) from a script, without a real phone in the loop. |

### 3.2 Mobile Client

Two Python-first paths, in order of recommendation:

**Option A — Python web app installed to the home screen (recommended to start).**

| Concern | What it is / why the project needs it | Library / Framework | Why this pick |
|---|---|---|---|
| UI framework | Something has to render a screen the user can tap "choose photo" / "upload" on — the actual visual app the phone shows. | **Reflex** (formerly Pynecone) | Write the entire front end in Python; it compiles to a real web app. Installable to a phone's home screen as a PWA — no JavaScript required from you. |
| Camera/file access | The app needs a way to pull bytes off the phone's camera roll or camera into the upload flow. | Browser `<input type="file" capture>` (Reflex wraps this) | The standard, cross-platform way for a mobile browser to open the camera or photo picker — no native SDK needed. |
| HTTP calls to API | Once a photo is selected, something has to actually send it over the network to the FastAPI server. | Reflex's built-in state/event handlers, or `httpx` under the hood | Same language, same repo, no separate client codebase to maintain. |

**Option B — Native-feeling app, still 100% Python.**

| Concern | What it is / why the project needs it | Library / Framework | Why this pick |
|---|---|---|---|
| UI framework | The visual layer of a real installable app, as opposed to a browser page. | **Kivy** + **KivyMD** | Pure Python UI toolkit with Material Design widgets; runs on Android and iOS from one codebase. |
| Packaging | A Kivy app is just Python source until something turns it into a file a phone can actually install. | **Buildozer** (Android) / **briefcase** (iOS, via BeeWare) | Turns the Kivy app into an installable APK/IPA. |
| HTTP calls | Same need as Option A — get the selected photo's bytes to the server. | **httpx** or **requests** | Upload photos to the FastAPI server; no browser to lean on here, so the app makes the HTTP call itself. |

Recommendation: start with **Option A (Reflex)** — it ships faster, updates instantly (just refresh the page, no app rebuild/reinstall), and keeps 100% of the code in Python. Revisit Kivy/BeeWare only if home-screen-web-app limitations (e.g. background upload, native camera controls) become a real blocker.

### 3.3 Networking / Remote Access

The phone and the computer need a way to actually reach each other over a network — this isn't optional, since without it the API only works if the phone happens to be sitting on the same Wi-Fi.

| Scenario | Why it matters | Approach |
|---|---|---|
| Phone on the same home Wi-Fi | The baseline case — both devices are already on one local network, so no extra infrastructure is needed. | Hit the computer's LAN IP directly, e.g. `http://192.168.1.50:8000`. Simplest option, no extra tooling. |
| Upload from outside the house | The whole point of a "phone → home computer" app is to also work when out and about, not just at home. | **Tailscale** — private mesh VPN, free for personal use, no port forwarding, no exposed public port. |
| Quick temporary testing only | Occasionally useful to show/test the app from a device not on Tailscale yet, without committing to a permanent access method. | **ngrok** or **Cloudflare Tunnel** — fine for a demo, not recommended as the long-term access method (adds a third party in the middle of every request). |

### 3.4 Deployment (Windows, since that's the host machine)

The API needs to survive the computer being used normally — logins, reboots, sleep — without the user having to manually open a terminal and start it every time.

| Concern | Why it's necessary | Tool |
|---|---|---|
| Keep the API running in the background | If the server only runs while a terminal window is open, a photo upload fails any time that window isn't up. | **NSSM** (Non-Sucking Service Manager) to run `uvicorn` as a Windows service, or Task Scheduler with "run at logon." |
| Reproducible environment | Python/library versions on the host machine can drift over time, causing "works on my machine" breakage. | **Docker Desktop** (optional) — containerizes the FastAPI app so its dependencies are pinned and isolated from the host Python install. |
| TLS | Traffic should be encrypted, especially if the API is ever reachable outside the local network. | **Caddy** as a reverse proxy in front of uvicorn if HTTPS is needed beyond the local network (Tailscale already encrypts traffic, so this may be unnecessary). |

## 4. Project Structure (proposed)

```
upload-photo-project/
├── api/
│   ├── main.py              # FastAPI app entrypoint
│   ├── routers/
│   │   └── uploads.py       # POST /uploads, GET /photos, etc.
│   ├── models.py            # SQLModel table definitions
│   ├── storage.py           # disk read/write, folder-by-date layout
│   ├── auth.py              # API key dependency
│   ├── settings.py          # pydantic-settings config
│   └── tasks.py             # thumbnailing, hashing, dedup
├── client/                  # Reflex app (or Kivy app under client/mobile/)
├── tests/
│   └── test_uploads.py
├── data/
│   ├── photos/              # YYYY/MM/DD/ on-disk storage
│   └── app.db               # SQLite database
├── .env.example
├── pyproject.toml
└── README.md
```

## 5. API Design (initial)

| Method | Endpoint | Purpose |
|---|---|---|
| `POST` | `/api/uploads` | Multipart upload of one photo + metadata (taken-at timestamp, device name). Returns the stored record. |
| `GET` | `/api/photos` | List uploaded photos (paginated), for a simple "did it arrive?" check. |
| `GET` | `/api/photos/{id}/thumbnail` | Return a generated thumbnail. |
| `GET` | `/api/health` | Liveness check the phone client can ping before uploading. |

Auth: a single long-lived API key sent as a header (`X-API-Key`), checked via a FastAPI dependency. Sufficient for a single-owner tool — no need for full OAuth/user accounts.

## 6. Data Model (photos table)

| Field | Type | Notes |
|---|---|---|
| `id` | UUID | Primary key |
| `filename` | str | Original filename from the phone |
| `stored_path` | str | Path on disk, relative to `data/photos/` |
| `content_hash` | str | For dedup (via `imagehash` + a plain SHA-256 for exact-duplicate checks) |
| `taken_at` | datetime | From EXIF if present, else upload time |
| `uploaded_at` | datetime | Server-side timestamp |
| `device` | str | Client-supplied label, e.g. "iPhone 15" |
| `width` / `height` | int | From Pillow |
| `size_bytes` | int | File size |

## 7. Security Considerations

- Never expose the raw API port to the open internet; use Tailscale instead of port forwarding.
- Validate uploaded content with `python-magic`, not just the file extension.
- Cap upload size (FastAPI/uvicorn setting) to avoid disk exhaustion.
- Store the API key in `.env`, excluded via `.gitignore` — never commit it.

## 8. Roadmap

1. **MVP**: FastAPI server with a single `/api/uploads` endpoint, saves to disk, SQLite metadata row, API-key auth. Test via `curl`/Postman.
2. **Client v1**: Reflex web app — pick/take a photo, upload, show success state. Installed to phone home screen.
3. **Remote access**: Set up Tailscale so uploads work away from home Wi-Fi.
4. **Quality of life**: thumbnails, duplicate detection, EXIF-based sorting into date folders, upload progress/retry.
5. **Stretch**: background/auto-upload of new camera-roll photos, video support, native Kivy app if the PWA proves limiting.

## 9. Alternatives Considered

- **Django + DRF** instead of FastAPI — more batteries-included (admin panel is genuinely useful for browsing uploaded photos), but heavier for a single-endpoint personal API. Worth reconsidering if an admin UI becomes a priority.
- **Flask** instead of FastAPI — simpler mental model, but FastAPI's async support and automatic docs are a better fit here with negligible extra complexity.
- **React Native / Flutter client** — more mature mobile tooling, but abandons the "as much Python as possible" goal; only reconsider if Reflex/Kivy prove insufficient.
