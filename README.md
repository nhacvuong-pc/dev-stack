# Ruby Vibe Development Stack

This directory provides local infrastructure and a Linux runtime for testing the Ruby Vibe API with Playwright Chromium.

## Services

| Service | Container | Host port | Purpose |
| --- | --- | --- | --- |
| `ruby-vibe-linux` | `ruby-vibe-linux` | `5000` | .NET 8 API running in Linux with Playwright Chromium |
| `postgres` | `rubys-postgres` | `5432` | PostgreSQL development database |
| `redis` | `rubys-redis` | `6379` | Redis product cache |
| `postgres-restore` | generated name | none | Optional one-time restore from `postgres/backup/rubys.backup` |

The API uses port `8080` inside the container and is exposed at:

```text
http://localhost:5000
```

## Directory layout

```text
dev-stack/
├── docker-compose.yaml
├── Dockerfile.ruby-vibe-linux
├── .env-template
├── postgres/backup/
└── persistence_data/
    ├── postgres/
    ├── redis/
    └── video-downloads/
```

The Docker build context is the parent `D:\00.Dev` directory so the Dockerfile can copy the sibling `ruby-vibe-com-api` repository.

Expected layout:

```text
D:\00.Dev\
├── dev-stack\
└── ruby-vibe-com-api\
```

## Prerequisites

- Docker Desktop with Linux containers enabled.
- The backend repository available at `../ruby-vibe-com-api`.
- Port 5000 available for the Linux API.
- Ports 5432 and 6379 available when PostgreSQL and Redis are exposed to the host.

Stop the backend process in Visual Studio before starting `ruby-vibe-linux`; the default launch profile also uses port 5000.

## Start the Linux API

Build the latest backend source and recreate only the API container:

```powershell
cd D:\00.Dev\dev-stack
docker compose up -d --build --force-recreate ruby-vibe-linux
```

Compose starts the required PostgreSQL and Redis services automatically.

View API logs:

```powershell
docker compose logs -f ruby-vibe-linux
```

Open Swagger:

```text
http://localhost:5000/swagger/index.html
```

## Rebuild after source changes

Normal rebuild:

```powershell
docker compose up -d --build --force-recreate ruby-vibe-linux
```

Force a clean image rebuild when Docker cache is stale:

```powershell
docker compose build --no-cache ruby-vibe-linux
docker compose up -d --force-recreate ruby-vibe-linux
```

## Video downloader

The Linux image is based on the official Playwright .NET `v1.61.0-noble` runtime, matching the backend package version. It includes the Linux browser binaries and operating-system dependencies.

The image also installs FFmpeg and yt-dlp for YouTube downloads. Compose
configures their container paths as:

```text
/usr/bin/ffmpeg
/usr/local/bin/yt-dlp
```

The container configuration enables headless downloads and stores files in:

```text
dev-stack/persistence_data/video-downloads
```

### Download queue workers

Video downloads run in a background channel and do not block the Publish HTTP
request. Compose currently configures:

```yaml
VideoDownloader__ConcurrentDownloads: 2
VideoDownloader__MaxFileSizeBytes: 15728640
```

Two workers allow two different video IDs to download concurrently:

```text
Publish A ─┐             Worker 1 → A
Publish B ─┼── Queue ──→ Worker 2 → B
Publish C ─┘             waits for a free worker
```

A duplicate Publish request for a `videoId` that is already queued or running
does not create another download. The lock for that ID is released after
success, failure, or cancellation.

Each worker owns the resources for one job: a browser session and any yt-dlp or
FFmpeg process required by the platform. These resources and partial files are
cleaned when the job finishes. Raising the worker count increases peak CPU,
RAM, bandwidth, and requests to source platforms. `2` is the recommended
default; the backend accepts values from 1 through 10.

When a job fails, the backend sets `AffiliateProgramVideos.IsPublish=false` and
writes a user-facing failure reason to `AffiliateProgramVideos.Mask`. A new
Publish attempt clears the previous `Mask`; a successful job leaves it empty.

Change the worker count by editing `docker-compose.yaml`, then recreate the API
container:

```powershell
docker compose up -d --build --force-recreate ruby-vibe-linux
```

Confirm the effective value and worker startup log:

```powershell
docker exec ruby-vibe-linux printenv VideoDownloader__ConcurrentDownloads
docker compose logs ruby-vibe-linux | Select-String "concurrent workers"
```

The public URL sent to Shopify comes from the backend `VideoDownloader:StorePath` configuration. The Compose file must not override it with a localhost URL.

Check the effective configuration inside the container:

```powershell
docker exec ruby-vibe-linux sh -lc "grep -A12 VideoDownloader /app/appsettings.json"
```

Run a Chromium smoke test:

```powershell
docker exec --user appuser ruby-vibe-linux sh -lc '/app/.playwright/node/linux-x64/node /app/.playwright/package/cli.js screenshot --browser chromium about:blank /tmp/playwright-smoke.png'
```

Verify the YouTube downloader dependencies:

```powershell
docker exec --user appuser ruby-vibe-linux ffmpeg -version
docker exec --user appuser ruby-vibe-linux yt-dlp --version
```

## PostgreSQL restore

Place a backup at:

```text
dev-stack/postgres/backup/rubys.backup
```

Then run the restore service explicitly:

```powershell
docker compose run --rm postgres-restore
```

The restore uses `--clean --if-exists`, so it replaces objects contained in the backup. Do not run it against data that must be preserved.

## Useful commands

Show service status:

```powershell
docker compose ps
```

Restart only the API:

```powershell
docker compose restart ruby-vibe-linux
```

Stop the API while keeping PostgreSQL and Redis running:

```powershell
docker compose stop ruby-vibe-linux
```

Stop the complete stack without deleting persisted bind-mounted data:

```powershell
docker compose down
```

Validate the Compose file:

```powershell
docker compose config --quiet
```

## Troubleshooting

### Port 5000 is already in use

Stop the API process launched by Visual Studio, or identify it with:

```powershell
Get-NetTCPConnection -LocalPort 5000 -State Listen
```

### Self-health-check targets `[::]`

The backend normalizes wildcard Kestrel addresses to `127.0.0.1`. Rebuild the API image to include the latest backend source.

### Shopify receives a localhost video URL

Remove any `VideoDownloader__StorePath` localhost override and recreate the container. Existing incorrect Shopify links must be removed by unpublishing the affected video.

### Browser fails to launch

Confirm the driver and browser directory exist:

```powershell
docker exec ruby-vibe-linux sh -lc 'test -x /app/.playwright/node/linux-x64/node && echo driver-ok; test -d /ms-playwright && echo browsers-ok'
```
