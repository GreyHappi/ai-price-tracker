# Runbook — Windows primary server

> Operating the **primary lane** (D-02) on the Windows 11 PC: Docker Compose runs `apps/api`
> (NestJS + grammY bot module, **long-polling**) and `apps/worker` (scheduler → `run-due-checks`).
> The runtime database is **Neon** — the local Docker Postgres is dev / integration-test only
> (D-01) and never the runtime store. Remote access is Tailscale + Tailscale Serve (D-09). The
> GitHub Actions backup lane is not configured from here; it only has to keep pinging its own
> healthcheck (D-02). Secrets live in a **gitignored `.env`** beside `docker-compose.yml`.

Paths assume `C:\ops\ai-price-tracker`. Adjust once; the rest is copy-pasteable.

## 1. Power — never sleep while tracking

```powershell
# PowerShell as Administrator
powercfg /change standby-timeout-ac 0      # never sleep on AC
powercfg /change hibernate-timeout-ac 0    # never hibernate on AC
powercfg /change disk-timeout-ac 0         # keep disks spinning
powercfg /change monitor-timeout-ac 10     # the screen may sleep — the machine may not
powercfg /hibernate off                    # frees hiberfil.sys, removes fast-startup surprises
powercfg /query SCHEME_CURRENT SUB_SLEEP   # verify: standby + hibernate read 0x00000000
powercfg /requests                         # what is holding the machine awake right now
```

- Disable **wake timers** (Settings → Power → Sleep) so Windows Update cannot wake-then-sleep the
  box mid-cycle. Laptop: mirror onto `-dc` only if it must track on battery.
- **BIOS/UEFI:** set *Restore on AC Power Loss* → **Power On**; without it a power cut leaves the
  machine off and the backup lane carries everything until you return.

## 2. Auto-start chain

Four rungs; each covers the previous one's failure mode.

**a. Docker Desktop at sign-in.** Settings → General → *Start Docker Desktop when you sign in*.
It is a per-user app, so an unattended reboot only recovers if the machine signs in by itself:
either sign in manually, or use automatic sign-in only on a dedicated, BitLocker-protected,
physically controlled machine. Automatic sign-in weakens local account security; if that trade-off
is unacceptable, let the backup lane cover checks until a human signs in.

**b. Compose restart policies** — the restart-on-failure guarantee (D-29):

```yaml
services:
  api:    { restart: unless-stopped }   # bot long-polling + /api/v1 on loopback
  worker: { restart: unless-stopped }   # scheduler → run-due-checks
```

`unless-stopped` also restarts both when the Docker daemon starts and does not resurrect containers
you stopped on purpose. Leave the dev `postgres` service off it — that service is dev/test (D-01).

**c. Task Scheduler fallback**, for when Docker Desktop starts but the stack does not:

```powershell
# ---- C:\ops\ai-price-tracker\start-primary.ps1 ----
for ($i = 0; $i -lt 60; $i++) {          # wait up to 10 min for the daemon
  docker info *> $null; if ($LASTEXITCODE -eq 0) { break }
  Start-Sleep -Seconds 10
}
docker compose --project-directory C:\ops\ai-price-tracker up -d
```

Register and smoke-test that saved script once; do not put these two commands inside the script:

```powershell
schtasks /Create /TN "AIPT-primary-up" /SC ONLOGON /RU "$env:USERNAME" /DELAY 0001:00 /F `
  /TR "powershell -NoProfile -ExecutionPolicy Bypass -File C:\ops\ai-price-tracker\start-primary.ps1"
schtasks /Run /TN "AIPT-primary-up"      # smoke-test now, not at the next boot
```

With automatic sign-in, `ONLOGON` fires on every boot including unattended Windows Update reboots.
`ONSTART` is avoided on purpose: it runs before a user session exists, where the engine is dead.

**d. Tailscale** must be `Running` / `Automatic`, and Serve must be the persistent (`--bg`) mapping
— a foreground `tailscale serve` dies with its terminal:

```powershell
Get-Service Tailscale | Format-List Name, Status, StartType
Set-Service Tailscale -StartupType Automatic         # only if it is not already
tailscale status
tailscale serve status --json                        # expect / → http://127.0.0.1:3000
tailscale serve --bg 3000                            # current CLI; re-arm if mapping is gone
```

Tailscale changed the Serve CLI in client 1.52; do not copy the older
`tailscale serve --bg https / ...` syntax. Run `tailscale serve --help` after upgrades.

## 3. Backups

History lives in Neon, so the dump runs **against Neon**, not against local dev Postgres (D-01).
Use an unpooled `DATABASE_DIRECT_URL`: Neon recommends direct connections for `pg_dump`.
The script writes a partial archive, validates it, and rotates older dumps **only after success**:

```powershell
# ---- C:\ops\ai-price-tracker\backup-neon.ps1 ----
$ErrorActionPreference = 'Stop'
$opsRoot = 'C:\ops\ai-price-tracker'
$envPath = Join-Path $opsRoot '.env'
$urlLine = Get-Content -LiteralPath $envPath |
  Where-Object { $_ -match '^\s*DATABASE_DIRECT_URL\s*=' } | Select-Object -First 1
if (-not $urlLine) { throw 'DATABASE_DIRECT_URL is missing from .env' }
$dbUrl = ($urlLine -replace '^\s*DATABASE_DIRECT_URL\s*=\s*', '').Trim().Trim('"').Trim("'")
if (-not $dbUrl -or $dbUrl -match '-pooler\.') { throw 'Use a non-pooled Neon URL for pg_dump' }

$backupDir = Join-Path $opsRoot 'backups'
New-Item -ItemType Directory -Force -Path $backupDir | Out-Null
$stamp = Get-Date -Format 'yyyy-MM-dd'
$tmpName = "aipt-$stamp.dump.partial"
$finalName = "aipt-$stamp.dump"
$tmpPath = Join-Path $backupDir $tmpName
$finalPath = Join-Path $backupDir $finalName

docker run --rm -v "${backupDir}:/backups" postgres:17 `
  pg_dump "$dbUrl" --format=custom --no-owner --file "/backups/$tmpName"
if ($LASTEXITCODE -ne 0) {
  Remove-Item -LiteralPath $tmpPath -Force -ErrorAction SilentlyContinue
  throw "pg_dump failed with exit code $LASTEXITCODE"
}

docker run --rm -v "${backupDir}:/backups" postgres:17 `
  pg_restore --list "/backups/$tmpName" *> $null
if ($LASTEXITCODE -ne 0) {
  Remove-Item -LiteralPath $tmpPath -Force -ErrorAction SilentlyContinue
  throw 'Archive validation failed; older backups were not rotated'
}

Move-Item -LiteralPath $tmpPath -Destination $finalPath -Force
Get-ChildItem -LiteralPath $backupDir -Filter 'aipt-*.dump' |
  Sort-Object LastWriteTime -Descending | Select-Object -Skip 7 | Remove-Item -Force
```

After saving the script, register its daily task once; do not include this command in the script:

```powershell
schtasks /Create /TN "AIPT-neon-backup" /SC DAILY /ST 03:30 /RU "$env:USERNAME" /F `
  /TR "powershell -NoProfile -ExecutionPolicy Bypass -File C:\ops\ai-price-tracker\backup-neon.ps1"
```

- `postgres:17` is a throwaway client; match its major to Neon's. The direct connection string is
  read from the gitignored `.env` — never hardcode it into a committable file.
- `backups\aipt-YYYY-MM-DD.dump` — 7 validated daily dumps outside the repo. A same-disk copy is
  not disaster recovery: copy at least one recent dump to an encrypted external/off-device target.
- Full `pg_dump` consumes Neon network transfer. Monitor project usage; reduce frequency or scope
  as data grows rather than silently exhausting the plan allowance.
- `diagnostics\` — sanitized, size-capped failure bodies from the primary lane. They **rotate after
  7 days** (D-12), never enter Neon, and are therefore not part of the dump:

```powershell
Get-ChildItem C:\ops\ai-price-tracker\diagnostics -Recurse -File |
  Where-Object { $_.LastWriteTime -lt (Get-Date).AddDays(-7) } | Remove-Item -Force
```

- `pg_restore --list` catches malformed archives but is not a restore test. The real isolated
  restore drill remains a Phase 4 gate.

## 4. Recovery checklist after a reboot or crash

1. **Containers up?** `docker compose --project-directory C:\ops\ai-price-tracker ps` — `api` and
   `worker` running, no restart loop; then `… logs --tail 50 api worker`. Missing containers →
   `schtasks /Run /TN "AIPT-primary-up"`.
2. **API alive?** It binds loopback and is reached only through Serve (D-10):
   `curl.exe -s -o NUL -w "%{http_code}" http://127.0.0.1:3000/api/v1/health` → `200`.
3. **Bot answers `/status`?** Send it in Telegram. Silence = long-polling did not resume; commands
   being down while the primary is down is an accepted risk, not a bug (D-03, N-01).
4. **Primary healthcheck pinging again?** The healthchecks.io *primary* check must clear its
   late/down state within one cycle (D-02). Still silent after a full cycle → read the worker log.
5. **Serve reachable?** `tailscale status` + `tailscale serve status`, then open
   `https://<host>.<tailnet>.ts.net/api/v1/health` from another tailnet device. 200 on loopback but
   nothing over Serve = the Serve mapping was lost; re-arm it (§2d).

## 5. Verification — did the backup lane cover the gap?

Every attempt is logged to `scrape_runs` with its `lane` (D-12), so downtime is auditable:

```powershell
$urlLine = Get-Content -LiteralPath C:\ops\ai-price-tracker\.env |
  Where-Object { $_ -match '^\s*DATABASE_DIRECT_URL\s*=' } | Select-Object -First 1
$dbUrl = ($urlLine -replace '^\s*DATABASE_DIRECT_URL\s*=\s*', '').Trim().Trim('"').Trim("'")
docker run --rm postgres:17 psql "$dbUrl" -c `
  "select lane, count(*) as runs, max(started_at) as latest from scrape_runs
   where started_at > now() - interval '24 hours' group by lane;"
# per-run detail: add `where lane = 'backup' order by started_at desc limit 20`
```

Zero `backup` rows is not automatically a failure: the lane skips sources the primary refreshed less
than 3 h ago (D-02), so a short outage legitimately produces a correct no-op run.

Then read **both** dead-man checks (D-02) together:

| primary | backup | reading |
|---|---|---|
| green | green | normal operation |
| red | green | the PC was down and the backup lane carried it — expected during the outage |
| green | red | Actions cron disabled or failing (public-repo 60-day inactivity, N-04) → re-arm |
| red | red | both lanes silent; nothing is being checked — treat as an incident |

The backup check pings even on a correct no-op, so a green backup check with zero backup rows still
proves the workflow ran (D-02, N-04). Record any outage that moves the monthly picture: primary
uptime below ~95% is the documented reopen trigger for D-02/D-03.
