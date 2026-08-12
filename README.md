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

| Concern | Library / Framework | Why |
|---|---|---|
| Web framework | **FastAPI** | Async, native file-upload support, automatic OpenAPI docs — easy to test from the phone or `curl` while building the client. |
| ASGI server | **uvicorn** | Standard partner for FastAPI; simple to run as a long-lived process. |
| Data validation | **Pydantic** | Ships with FastAPI; validates upload metadata (device, timestamp, GPS, etc.). |
| Database ORM | **SQLModel** (or SQLAlchemy directly) | Pydantic + SQLAlchemy in one; stores photo metadata in SQLite. |
| Database | **SQLite** | Zero-config, file-based — fits a single-machine personal project. |
| Image handling | **Pillow** | Thumbnail generation, format conversion, EXIF reading. |
| iPhone photo support | **pillow-heif** | iPhones shoot HEIC/HEIF by default; Pillow doesn't decode it without this plugin. |
| File-type sniffing | **python-magic** | Verify uploaded bytes are actually images before trusting the extension. |
| Duplicate detection | **imagehash** | Perceptual hashing to skip re-uploading a photo already on disk. |
| Background work | **FastAPI `BackgroundTasks`** (start simple) → **Celery** or **RQ** (if a real queue is needed later) | Thumbnail generation / hashing shouldn't block the upload response. |
| Config / secrets | **pydantic-settings** + `.env` | API key and storage path live outside source control. |
| Testing | **pytest** + **httpx** (`AsyncClient`) | Exercise API endpoints without a real phone. |

### 3.2 Mobile Client

Two Python-first paths, in order of recommendation:

**Option A — Python web app installed to the home screen (recommended to start).**

| Concern | Library / Framework | Why |
|---|---|---|
| UI framework | **Reflex** (formerly Pynecone) | Write the entire front end in Python; it compiles to a real web app. Installable to a phone's home screen as a PWA — no JavaScript required from you. |
| Camera/file access | Browser `<input type="file" capture>` (Reflex wraps this) | Standard way for a mobile browser to open the camera or photo picker. |
| HTTP calls to API | Reflex's built-in state/event handlers, or `httpx` under the hood | Same language, same repo, no separate client codebase. |

**Option B — Native-feeling app, still 100% Python.**

| Concern | Library / Framework | Why |
|---|---|---|
| UI framework | **Kivy** + **KivyMD** | Pure Python UI toolkit with Material Design widgets; runs on Android and iOS. |
| Packaging | **Buildozer** (Android) / **briefcase** (iOS, via BeeWare) | Turns the Kivy app into an installable APK/IPA. |
| HTTP calls | **httpx** or **requests** | Upload photos to the FastAPI server. |

Recommendation: start with **Option A (Reflex)** — it ships faster, updates instantly (just refresh the page, no app rebuild/reinstall), and keeps 100% of the code in Python. Revisit Kivy/BeeWare only if home-screen-web-app limitations (e.g. background upload, native camera controls) become a real blocker.

### 3.3 Networking / Remote Access

| Scenario | Approach |
|---|---|
| Phone on the same home Wi-Fi | Hit the computer's LAN IP directly, e.g. `http://192.168.1.50:8000`. Simplest option, no extra tooling. |
| Upload from outside the house | **Tailscale** — private mesh VPN, free for personal use, no port forwarding, works from Python without extra client code since it's just networking. |
| Quick temporary testing only | **ngrok** or **Cloudflare Tunnel** — fine for a demo, not recommended as the long-term access method. |

### 3.4 Deployment (Windows, since that's the host machine)

| Concern | Tool |
|---|---|
| Keep the API running in the background | **NSSM** (Non-Sucking Service Manager) to run `uvicorn` as a Windows service, or Task Scheduler with "run at logon." |
| Reproducible environment | **Docker Desktop** (optional) — containerize the FastAPI app so the host machine's Python install doesn't drift. |
| TLS | **Caddy** as a reverse proxy in front of uvicorn if HTTPS is needed beyond the local network (Tailscale already encrypts traffic, so this may be unnecessary). |

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
