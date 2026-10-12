# DataSynk v4 (prototype), archived

> **This repository is archived.** It is the v4 prototype of DataSynk, kept as a record of what was built and learned. Work continues in **[IndicKangaroo/datasynk](https://github.com/IndicKangaroo/datasynk)** (the v5 reboot), which has the current design, plan and decision log. Archived October 2026.

DataSynk is a personal, free-tier file transfer between your own devices. A Python agent on the desktop watches `~/Datasynk` and sends new or changed files to other devices, over the LAN when it can and through a cloud relay when it can't. A Flutter Android client receives from the relay.

## Status

Prototype, built 2026-06-18 to 2026-06-30: 8 commits, 1,000 lines. The commit messages say "untested", and the pieces were never wired together end to end. Nothing here is maintained.

## What's in here

| Path | What it is |
|---|---|
| `backend/main.py` | Desktop agent entry point. Starts the watcher, mDNS announce and browse, the TCP receiver (port + 1), the sync and poll threads, and a FastAPI status API on `0.0.0.0:8765` (`/status`, `/events`, `/devices`). The port is the first command-line argument |
| `backend/watcher.py` | watchdog `PollingObserver` on `~/Datasynk`; records file events in SQLite |
| `backend/discovery.py`, `backend/devices.py` | mDNS (`_datasync._tcp.local.`) announce and browse, and the known-devices table |
| `backend/transfer.py` | LAN transfer over raw TCP: filename length, filename, then bytes until the connection closes |
| `backend/sync_manager.py` | The sync loop (every 5 s) and the relay poll loop (every 10 s) |
| `backend/storgae.py` | Backblaze B2 upload, download and delete helpers via boto3 (the file name is a typo for "storage") |
| `backend/database.py` | SQLite schema: `file_events` and `devices` |
| `backend/relay_server.py` | The cloud relay (FastAPI, meant for Render): `/notify`, `/poll/{device_id}`, `/confirm/{id}`, `/download-url/{key}` (presigned B2 link, 5 minutes). Queue in a local `relay.db` |
| `mobile/lib/` | Flutter app source only: `main.dart` (UI), `api_service.dart` (relay calls), `b2_downloader.dart` (download into a chosen folder) |

## How it works, as coded

1. The watcher records created, modified, deleted and moved events in `file_events` (`database.db`).
2. Every 5 seconds the sync loop goes through unsynced events and, for each known device, either sends the file over TCP (IP starting with `10.`, `192.168.` or `172.`) or uploads it to B2 and tells the relay with `POST /notify`.
3. Devices enter the `devices` table only through mDNS, which is LAN-only.
4. The relay is a small mailbox: `/notify` queues an item, `/poll/{device_id}` lists a device's items, `/confirm/{id}` marks one downloaded, and `/download-url/{key}` returns a short-lived presigned B2 link.
5. The phone polls `/poll/phone-001` when you tap Fetch, asks the relay for a presigned URL per file, downloads it with Dio into the app's documents directory or a folder chosen with `FilePicker`, and confirms.

## Known problems (deliberately not fixed here)

**The desktop never reaches the phone**
- `RELAY_URL` in `sync_manager.py` is still the placeholder `https://your-app.onrender.com`; the phone uses `https://datasynk.onrender.com`.
- The cloud branch only runs for devices with a non-private IP, but devices only enter the table through mDNS, so nothing ever registers `phone-001`.
- The agent's device ID is a new UUID on every launch.
- `poll_loop` fetches the queue and does nothing with it. The desktop's download-from-B2 code was removed in `a8b0c49`.

**Phone**
- `downloadFile` catches every exception and only prints it, so `main.dart` goes on to confirm the transfer, drop it from the list and show a success message. A failed write looks like success.
- Files are written to a path from `FilePicker.getDirectoryPath()` with plain file I/O, which Android's scoped storage generally rejects. There is no Storage Access Framework code in this repo.
- `notifyRelay` is never called, and it would get HTTP 422 anyway: it sends a form body and the relay expects query parameters.
- The phone ID `phone-001` is hard-coded, and files are fetched only when you tap Fetch.

**Delivery**
- A failed LAN send or relay notification is still marked synced, so it is never retried.
- An exception during a B2 upload aborts the whole pass, so one failing file blocks everything after it.
- Deletes are recorded but not propagated (`# V6`). Moves are recorded with the source path and never sent.
- The TCP protocol has no file size or checksum, so a cut-off transfer looks complete. Subfolders are flattened to the base name, and B2 objects are keyed by base name too, so same-named files collide.
- B2 objects are never deleted after `a8b0c49`.

**Watching**
- It uses polling, not native file events.
- There is no rescan on startup, so changes made while the agent was off are never seen.
- Echo suppression is a set of paths (`IGNORE_PATHS`), which is racy when one write fires several events.

**Relay and hosting**
- The relay keeps its queue in a local SQLite file, which Render's free tier discards on spin-down.
- The keep-alive ping goes to `/status`, which the relay doesn't implement.

**Security**
- The relay has no authentication: anyone with its URL can request a download link for any key in the bucket and read or clear any device's queue.
- The TCP receiver accepts files from anyone who can reach the port, and the agent's API binds to `0.0.0.0` with no authentication.
- `is_lan` treats every `172.x` address as private, but the private range is only 172.16 to 172.31.
- No credentials are committed (history checked on 2026-10-11).

**Packaging**
- `requirements.txt` lists only what the relay needs (fastapi, uvicorn, boto3, dotenv). The agent also imports zeroconf, watchdog and requests.
- Three backend files use Windows line endings and the rest use Unix ones.

## History guide

| Commit | Date | What it did |
|---|---|---|
| `a672583` | 2026-06-18 | Scaffolding: agent API, SQLite schema, the relay (`/notify`, `/poll`, `/confirm`) and `requirements.txt`; the other modules created empty |
| `f9f3e5c` | 2026-06-21 | Small change to `main.py` |
| `4e038e6` | 2026-06-21 | The LAN engine lands: mDNS discovery, device table, folder watcher, TCP transfer, sync loop, B2 upload |
| `3a882c2` | 2026-06-23 | No code change despite the message ("empty response handel"): line endings and file modes, plus an empty `api_service.dart` |
| `a8b0c49` | 2026-06-23 | The poll loop gains an empty-response guard, timeouts and a keep-alive ping, and loses the desktop's download-from-B2 code |
| `e591b85` | 2026-06-25 | The relay gains `/download-url` (presigned B2 link); an empty `b2_downloader.dart` is created |
| `8361321` | 2026-06-25 | `boto3` and `dotenv` added to `requirements.txt` |
| `c571acb` | 2026-06-30 | The relay signs with SigV4 (`s3v4`), and the whole Flutter app arrives in one commit |

## What is not in this repo

- **The Flutter project root** (`pubspec.yaml`, `android/`). Android build configuration and any Storage Access Framework experiments lived outside git.
- **V1 to V3.** The first commit is already labelled V4.
- **The original manual** (`DATASYNC_PROJECT.md`). It planned Flutter plus an Electron/React desktop app, hosted on Fly.io with Cloudflare R2 storage. The Electron app was never built, and Backblaze B2 and Render replaced R2 and Fly.io because those two demand a billing account. The manual is kept in the new repo under `docs/archive/`.

## Running it (reference only)

Don't deploy the relay as it is: it is unauthenticated.

- **Relay:** `pip install fastapi uvicorn boto3 python-dotenv`, set `B2_ENDPOINT`, `B2_KEY_ID`, `B2_APPLICATION_KEY` and `B2_BUCKET`, then `python relay_server.py` (listens on `$PORT`, default 8000).
- **Agent:** additionally `pip install zeroconf watchdog requests`, set the same B2 variables, edit `RELAY_URL` in `sync_manager.py`, then `python main.py [port]` from `backend/`.

On 2026-10-11 the relay's routes were exercised locally with FastAPI's test client to confirm the behaviour described above. The rest of the findings come from reading the code.

## Where to go next

The v5 design, plan and decision log are in [IndicKangaroo/datasynk](https://github.com/IndicKangaroo/datasynk): `README.md` (design), `docs/PLAN.md` (milestones), `docs/decisions.md` (why) and `docs/design-review.md`.
