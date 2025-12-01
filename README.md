# HostBridge Cron Manager

This directory contains the Fly.io cron-manager configuration for scheduled tasks.

## Overview

Uses [fly-apps/cron-manager](https://github.com/fly-apps/cron-manager) (via our fork at [redblacktree/cron-manager](https://github.com/redblacktree/cron-manager)) to run scheduled tasks by spinning up lightweight curl containers that call the main API.

## Setup

1. **Create the app** (first time only):

   ```bash
   cd cron-manager
   fly apps create hostbridge-cron-manager
   ```

2. **Set secrets**:

   ```bash
   # Generate a secure secret for CRON_SECRET (use the same value in hostbridge-api)
   fly secrets set CRON_SECRET=<your-secure-secret> --app hostbridge-cron-manager

   # Set the Fly API token for machine provisioning
   fly secrets set FLY_API_TOKEN=$(fly auth token) --app hostbridge-cron-manager
   ```

3. **Set CRON_SECRET on the API** (must match):

   ```bash
   fly secrets set CRON_SECRET=<your-secure-secret> --app hostbridge-api
   ```

4. **Deploy**:
   ```bash
   fly deploy --app hostbridge-cron-manager
   ```

## Schedules

Defined in `schedules.json`:

| Name            | Schedule             | Description                                          |
| --------------- | -------------------- | ---------------------------------------------------- |
| `periodic-sync` | `0 * * * *` (hourly) | Triggers periodic sync check for all connected users |

## How It Works

1. Cron-manager runs as a persistent Fly app
2. On schedule, it provisions a temporary machine with the `curlimages/curl` image
3. The machine runs curl to POST to the API's cron endpoint
4. The API authenticates with `CRON_SECRET` and runs `runPeriodicSyncCheck()`
5. Jobs are queued to pg-boss for processing (handles retries, backoff)
6. The temporary machine is destroyed after completion

## Monitoring

View cron-manager logs:

```bash
fly logs --app hostbridge-cron-manager
```

View API logs for sync activity:

```bash
fly logs --app hostbridge-api | grep -E "\[Cron\]|\[Sync\]"
```

## Testing

Manually trigger a sync check:

```bash
curl -X POST \
  -H "Authorization: Bearer $CRON_SECRET" \
  -H "Content-Type: application/json" \
  https://api.hostbridge.ai/api/admin/cron/periodic-sync
```

## Updating from Upstream

This is a fork of `fly-apps/cron-manager`. To sync upstream changes:

```bash
cd cron-manager
git remote add upstream https://github.com/fly-apps/cron-manager.git  # if not already added
git fetch upstream
git merge upstream/main
git push origin main  # push to your fork
cd ..
git add cron-manager
git commit -m "Update cron-manager from upstream"
```
