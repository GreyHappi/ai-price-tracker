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
# Keep the probe independent of the caller's error preference.
$ErrorActionPreference = 'Continue'
# Fail fast when the CLI itself is missing: without this the loop below never runs docker, and a
# manual run in a shell with a stale $LASTEXITCODE could read as success.
if (-not (Get-Command docker -ErrorAction SilentlyContinue)) {
  throw 'docker is not on PATH - install or repair Docker Desktop, then run again'
}
for ($i = 0; $i -lt 60; $i++) {          # wait up to 10 min for the daemon
  & docker info 2>$null | Out-Null; if ($LASTEXITCODE -eq 0) { break }
  Start-Sleep -Seconds 10
}
if ($LASTEXITCODE -ne 0) {
  throw 'Docker daemon did not come up within 10 min - fix it, then run again'
}
docker compose --project-directory C:\ops\ai-price-tracker up -d
if ($LASTEXITCODE -ne 0) { throw 'docker compose up failed - see the error above' }
```

Register, smoke-test, and inspect that saved script once; do not put these commands inside the
script:

```powershell
schtasks /Create /TN "AIPT-primary-up" /SC ONLOGON /RU "$env:USERNAME" /DELAY 0001:00 /F `
  /TR "powershell -NoProfile -ExecutionPolicy Bypass -File C:\ops\ai-price-tracker\start-primary.ps1"
schtasks /Run /TN "AIPT-primary-up"      # smoke-test now, not at the next boot
# After the smoke run exits, record the XML and confirm Last Run Result is 0:
schtasks /Query /TN "AIPT-primary-up" /XML
schtasks /Query /TN "AIPT-primary-up" /V /FO LIST
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
# This URL -> libpq parsing block is intentionally duplicated in the §5 audit snippet: a runbook
# snippet must stay copy-pasteable on its own. Phase 0 extracts a shared parse-neon-url.ps1 and
# both callers dot-source it instead.
$urlLine = Get-Content -LiteralPath $envPath |
  Where-Object { $_ -match '^\s*DATABASE_DIRECT_URL\s*=' } | Select-Object -First 1
if (-not $urlLine) { throw 'DATABASE_DIRECT_URL is missing from .env' }
$dbUrl = ($urlLine -replace '^\s*DATABASE_DIRECT_URL\s*=\s*', '').Trim().Trim('"').Trim("'")
if (-not $dbUrl -or $dbUrl -match '-pooler\.') { throw 'Use a non-pooled Neon URL for pg_dump' }
$u = $null
$uriValid = [System.Uri]::TryCreate($dbUrl, [UriKind]::Absolute, [ref]$u)
if (-not $uriValid -or $u.Scheme -notin @('postgres', 'postgresql')) {
  throw 'DATABASE_DIRECT_URL must be an absolute postgres:// or postgresql:// URL'
}
$dbUri = $u
$userInfo = $dbUri.UserInfo
$userSeparator = $userInfo.IndexOf(':')
if ($userSeparator -lt 0) { throw 'DATABASE_DIRECT_URL must include a database user and password' }
$pgUser = [System.Uri]::UnescapeDataString($userInfo.Substring(0, $userSeparator))
$pgPassword = [System.Uri]::UnescapeDataString($userInfo.Substring($userSeparator + 1))
$pgDatabase = [System.Uri]::UnescapeDataString($dbUri.AbsolutePath.TrimStart('/'))
if (-not $dbUri.Host -or -not $pgUser -or -not $pgPassword -or -not $pgDatabase) {
  throw 'DATABASE_DIRECT_URL must include a host, user, password, and database'
}
$pgPort = if ($dbUri.Port -lt 0) { '5432' } else { [string]$dbUri.Port }
$pgSslMode = 'require'
$pgChannelBinding = $null
$pgConnectTimeout = $null
$pgAppName = $null
$pgOptions = $null
foreach ($queryPart in ($dbUri.Query.TrimStart('?') -split '&')) {
  if (-not $queryPart) { continue }
  $queryPieces = $queryPart -split '=', 2
  $queryName = [System.Uri]::UnescapeDataString($queryPieces[0])
  $queryValue = if ($queryPieces.Count -eq 2) {
    [System.Uri]::UnescapeDataString($queryPieces[1])
  } else {
    $null
  }
  switch ($queryName.ToLowerInvariant()) {
    'sslmode' {
      if ([string]::IsNullOrEmpty($queryValue)) {
        throw 'DATABASE_DIRECT_URL sslmode query parameter must include a value'
      }
      $pgSslMode = $queryValue
    }
    'channel_binding' {
      if ([string]::IsNullOrEmpty($queryValue)) {
        throw 'DATABASE_DIRECT_URL channel_binding query parameter must include a value'
      }
      $pgChannelBinding = $queryValue
    }
    'connect_timeout' {
      if ([string]::IsNullOrEmpty($queryValue)) {
        throw 'DATABASE_DIRECT_URL connect_timeout query parameter must include a value'
      }
      $pgConnectTimeout = $queryValue
    }
    'application_name' {
      if ([string]::IsNullOrEmpty($queryValue)) {
        throw 'DATABASE_DIRECT_URL application_name query parameter must include a value'
      }
      $pgAppName = $queryValue
    }
    'options' {
      if ([string]::IsNullOrEmpty($queryValue)) {
        throw 'DATABASE_DIRECT_URL options query parameter must include a value'
      }
      $pgOptions = $queryValue
    }
    default {
      throw "Unsupported DATABASE_DIRECT_URL query parameter '$queryName'"
    }
  }
}
# Escalate to verified TLS (see the note below this script): never weaken, never skip.
$pgSslMode = $pgSslMode.ToLowerInvariant()
if ($pgSslMode -in @('disable', 'allow', 'prefer')) {
  throw "DATABASE_DIRECT_URL sslmode '$pgSslMode' is weaker than require and is not accepted"
}
if ($pgSslMode -notin @('require', 'verify-ca', 'verify-full')) {
  throw "DATABASE_DIRECT_URL sslmode '$pgSslMode' is not recognized"
}
$pgSslMode = 'verify-full' # require and verify-ca are both upgraded; hostname verification is mandatory.
$caCertDir = Join-Path $opsRoot 'certs'
$caCertPath = Join-Path $caCertDir 'isrgrootx1.pem'
if (-not (Test-Path -LiteralPath $caCertPath)) {
  throw "Missing $caCertPath - download it once (see the registration step below)"
}
$expectedCaFileSha256 = '22B557A27055B33606B6559F37703928D3E4AD79F110B407D04986E1843543D1'
$actualCaFileSha256 = (Get-FileHash -LiteralPath $caCertPath -Algorithm SHA256).Hash
if ($actualCaFileSha256 -ne $expectedCaFileSha256) {
  throw "Unexpected ISRG Root X1 file SHA-256 at $caCertPath; do not use this trust anchor"
}
$pgEnvNames = @(
  'PGHOST', 'PGPORT', 'PGDATABASE', 'PGUSER', 'PGPASSWORD', 'PGSSLMODE',
  'PGCHANNELBINDING', 'PGCONNECT_TIMEOUT', 'PGAPPNAME', 'PGOPTIONS', 'PGSSLROOTCERT'
)
$priorPgEnv = @{}
foreach ($pgEnvName in $pgEnvNames) {
  $priorValue = Get-Item -LiteralPath "Env:$pgEnvName" -ErrorAction SilentlyContinue
  $priorPgEnv[$pgEnvName] = @{ Present = $false; Value = $null }
  if ($null -ne $priorValue) {
    $priorPgEnv[$pgEnvName].Present = $true
    $priorPgEnv[$pgEnvName].Value = $priorValue.Value
  }
}
try {
  $env:PGHOST = $dbUri.Host
  $env:PGPORT = $pgPort
  $env:PGDATABASE = $pgDatabase
  $env:PGUSER = $pgUser
  $env:PGPASSWORD = $pgPassword
  $env:PGSSLMODE = $pgSslMode
  $env:PGSSLROOTCERT = '/certs/isrgrootx1.pem'
  $dockerEnvArgs = @(
    '--env', 'PGHOST', '--env', 'PGPORT', '--env', 'PGDATABASE', '--env', 'PGUSER',
    '--env', 'PGPASSWORD', '--env', 'PGSSLMODE', '--env', 'PGSSLROOTCERT'
  )
  if ($null -ne $pgChannelBinding) {
    $env:PGCHANNELBINDING = $pgChannelBinding
    $dockerEnvArgs += @('--env', 'PGCHANNELBINDING')
  }
  if ($null -ne $pgConnectTimeout) {
    $env:PGCONNECT_TIMEOUT = $pgConnectTimeout
    $dockerEnvArgs += @('--env', 'PGCONNECT_TIMEOUT')
  }
  if ($null -ne $pgAppName) {
    $env:PGAPPNAME = $pgAppName
    $dockerEnvArgs += @('--env', 'PGAPPNAME')
  }
  if ($null -ne $pgOptions) {
    $env:PGOPTIONS = $pgOptions
    $dockerEnvArgs += @('--env', 'PGOPTIONS')
  }

  $backupDir = Join-Path $opsRoot 'backups'
  New-Item -ItemType Directory -Force -Path $backupDir | Out-Null
  $stamp = Get-Date -Format 'yyyy-MM-dd'
  $tmpName = "aipt-$stamp.dump.partial"
  $finalName = "aipt-$stamp.dump"
  $tmpPath = Join-Path $backupDir $tmpName
  $finalPath = Join-Path $backupDir $finalName

  try {
    docker run --rm @dockerEnvArgs `
      -v "${backupDir}:/backups" -v "${caCertDir}:/certs:ro" postgres:17 `
      pg_dump --format=custom --no-owner --file "/backups/$tmpName"
    if ($LASTEXITCODE -ne 0) { throw "pg_dump failed with exit code $LASTEXITCODE" }
  }
  catch {
    Remove-Item -LiteralPath $tmpPath -Force -ErrorAction SilentlyContinue
    throw
  }

  try {
    docker run --rm -v "${backupDir}:/backups" postgres:17 `
      pg_restore --list "/backups/$tmpName" | Out-Null
    if ($LASTEXITCODE -ne 0) { throw 'Archive validation failed; older backups were not rotated' }
  }
  catch {
    Remove-Item -LiteralPath $tmpPath -Force -ErrorAction SilentlyContinue
    throw
  }

  Move-Item -LiteralPath $tmpPath -Destination $finalPath -Force
  Get-ChildItem -LiteralPath $backupDir -Filter 'aipt-*.dump' |
    Sort-Object LastWriteTime -Descending | Select-Object -Skip 7 | Remove-Item -Force
}
finally {
  # Success or throw, no PG* value - PGPASSWORD above all - survives in the calling shell.
  foreach ($pgEnvName in $pgEnvNames) {
    $prior = $priorPgEnv[$pgEnvName]
    if ($prior.Present) {
      Set-Item -LiteralPath "Env:$pgEnvName" -Value $prior.Value
    } else {
      Remove-Item -LiteralPath "Env:$pgEnvName" -ErrorAction SilentlyContinue
    }
  }
}
```

The script works under both Windows PowerShell 5.1 and PowerShell 7 (`pwsh`).

The query-parameter switch is strict on purpose — an unknown parameter throws rather than being
silently dropped — so if Neon ever adds a new benign parameter the task **fails closed**: verify
that parameter against Neon's connection docs, then add a switch arm for it; the failure is visible
in Task Scheduler history and as the missing daily dump (the dead-man signal for backups).
Server identity is verified, not assumed: the scripts normalize `require`, `verify-ca`, and
`verify-full` to `verify-full` (weaker or unknown modes throw) against a pinned **ISRG Root X1** —
the trust anchor used by [Neon's verified-psql guidance](https://neon.com/blog/avoid-mitm-attacks-with-psql-postgres-16).
The exact PEM bytes from [Let's Encrypt's root registry](https://letsencrypt.org/certificates/) are
SHA-256-pinned; the root is mounted read-only into the
client container because the `postgres:17` image ships **no** CA bundle (`ca-certificates` is not
installed; verified empirically 2026-08-24), so libpq-17's `sslrootcert=system` would fail there
even though the client itself is new enough. The stock Neon URL's `channel_binding=require` stays
as defense-in-depth for the credentials. If verification starts failing after a Neon-side CA
change, verify the replacement against Neon's current security guidance and update both pins.

After saving the script, run the one-time setup — download the pinned root CA, register the daily
task, and smoke-test it; none of these commands belongs inside the script:

```powershell
New-Item -ItemType Directory -Force C:\ops\ai-price-tracker\certs | Out-Null
curl.exe -fsSL https://letsencrypt.org/certs/isrgrootx1.pem `
  -o C:\ops\ai-price-tracker\certs\isrgrootx1.pem
$expectedCaFileSha256 = '22B557A27055B33606B6559F37703928D3E4AD79F110B407D04986E1843543D1'
$caCertPath = 'C:\ops\ai-price-tracker\certs\isrgrootx1.pem'
if ((Get-FileHash -LiteralPath $caCertPath -Algorithm SHA256).Hash -ne $expectedCaFileSha256) {
  Remove-Item -LiteralPath $caCertPath -Force
  throw 'Downloaded ISRG Root X1 file failed the pinned SHA-256 check; task was not registered'
}
schtasks /Create /TN "AIPT-neon-backup" /SC DAILY /ST 03:30 /RU "$env:USERNAME" /F `
  /TR "powershell -NoProfile -ExecutionPolicy Bypass -File C:\ops\ai-price-tracker\backup-neon.ps1"
schtasks /Run /TN "AIPT-neon-backup"     # smoke-test the verified-TLS path now, not at 03:30
# After the smoke run exits, record the XML and confirm Last Run Result is 0:
schtasks /Query /TN "AIPT-neon-backup" /XML
schtasks /Query /TN "AIPT-neon-backup" /V /FO LIST
```

- `postgres:17` is a throwaway client; match its major to Neon's. The direct connection string is
  read from the gitignored `.env` — never hardcode it into a committable file and never pass it as
  a command-line argument (it would appear in the process list); the scripts parse it into libpq
  variables and pass those to the throwaway container via value-less `--env` flags only (N-18).
  `PGPASSWORD` passed via `--env` is visible in `docker inspect` output for the container's
  lifetime; these are throwaway (`--rm`), short-lived containers.
- `backups\aipt-YYYY-MM-DD.dump` — 7 validated daily dumps outside the repo. A same-disk copy is
  not disaster recovery: copy at least one recent dump to an encrypted external/off-device target.
- Full `pg_dump` consumes Neon network transfer. Monitor project usage; reduce frequency or scope
  as data grows rather than silently exhausting the plan allowance.
- `diagnostics\failures\` — sanitized, size-capped failure bodies from the primary lane. They
  **rotate after 7 days** (D-12), never enter Neon, and are therefore not part of the dump.
  The initial accepted baseline, every accepted change, and every suspicious/held candidate store a
  sanitized, size-capped page snapshot under primary's separate `snapshots\` directory. Every file
  is retained **at least 35 days** (the Phase-2 two-week verification window plus margin); afterward
  the two newest `accepted` baseline snapshots and every unresolved `held`/`pending` snapshot stay.
  Thus newer held attempts cannot evict the current baseline and predecessor evidence. On the
  Actions lane each accepted or held candidate contributes one file to the workflow execution's
  single artifact, named `snapshots-<github-run-id>-<run-attempt>` with `retention-days: 30`.
  Its archive contains only root files named `snapshot-<scrape-run-id>.json`, each one a
  strict UTF-8 JSON `snapshot-v1` envelope (duplicate/unknown fields fail): schema version, entry
  id, scrape-run id, DB-issued RFC-3339 UTC capture time, lane, candidate disposition, media type,
  the size-capped sanitized payload as canonical base64, and SHA-256 of its decoded bytes. Primary
  snapshots use the same envelope.
- **Primary evidence barrier:** before creating the initial baseline or committing an accepted/held
  candidate, atomically install its local envelope under lifecycle state `pending`; only then may
  semantic effects commit. Reconcile the filename to `accepted`, `held`, or `discardable` from the
  terminal `scrape_runs` state before a healthy ping. A crash leaves `pending` protected; startup
  reconciliation finishes it. A local write failure blocks effects; a post-commit rename failure
  keeps the protected `pending` file and suppresses the ping until reconciliation succeeds.
- **Backup evidence barrier:** the Actions workflow runs the backup command as
  `prepare → actions/upload-artifact@043fb46d1a93c77aae656e7c1c64a875d1fc6a0a → finalize`
  ([reviewed v7.0.1 release](https://github.com/actions/upload-artifact/releases/tag/v7.0.1);
  never a mutable major tag). `prepare` keeps/renews each url-group and entry lease, writes every
  envelope to a dedicated upload directory and sealed continuations outside it, but commits no
  accepted observation, baseline, `change_event`, or `notification_delivery`. One upload of that
  exact directory covers the batch (`if-no-files-found: error`, `overwrite: false`); if prepare
  reports zero envelopes, skip upload and finalize the no-change results. `finalize` receives artifact id
  and digest, then independently revalidates both fencing-token scopes/expiry before each effect.
  Upload failure affects the batch; a candidate whose finalize fence fails is retried from a fresh
  fetch while other valid candidates may finish idempotently. Record the concrete error in each
  affected `scrape_runs`; emit no unfenced alert or baseline move. Bound batch count/duration,
  renew earlier leases during prepare and immediately before upload, and size TTL for the bounded
  upload/finalize budget; finalize still revalidates rather than assuming it remained live. Orphan
  evidence is harmless.
- **Backup-artifact ingest contract (Phase 1 implementation):** before a primary cycle may ping
  healthy, list every non-expired `snapshots-*` artifact for the exact backup workflow and repository
  (do not rely on a last-seen cursor or API ordering). Download with a fine-grained
  `GITHUB_SNAPSHOT_TOKEN` limited to that public repository's **Actions: read**, stored only in the
  gitignored ops `.env` and never placed on a command line. Verify the artifact API's SHA-256 digest
  before extraction; enforce configured entry-count and aggregate expanded-byte caps; accept only
  regular root files matching `snapshot-<uuid>.json` (no directory, symlink, absolute path, `..`,
  alternate stream, duplicate name/id, or extra entry). Validate every envelope id, UTC time, lane
  and decoded payload hash; correlate each `scrape_run_id` with its terminal DB state; atomically
  move each file from a same-volume temp directory to its lifecycle-state destination. A live
  non-terminal run is retried and suppresses the ping. Once its exact workflow run is terminal and
  both DB leases have expired, reconciliation atomically marks a still-prepared run `abandoned`
  (no semantic effects) and installs its evidence as `discardable`; never infer abandonment from a
  process clock or timeout alone. An existing run id with the same origin digest and immutable
  envelope fields is an idempotent success; any mismatch is an incident.
  Wrong workflow/run, expired/missing artifact, download or validation failure is retried on the
  next cycle and suppresses the primary success ping, while scraping itself may continue. Listing
  all still-live artifacts makes duplicate and out-of-order API results harmless. This closes the
  chain only when primary returns inside the 30-day artifact window; a longer outage can lose
  expired evidence and is D-12's accepted risk.
- **Snapshot layout contract:**
  `snapshots\<entry-id>\<yyyyMMddTHHmmss.fffZ>-<scrape-run-id>-<state>.json` — state is `pending`,
  `accepted`, `held`, or `discardable`; lifecycle transitions are same-directory atomic renames.
  There is exactly **one self-contained `snapshot-v1` envelope per snapshot**. The filename time
  must equal its validated UTC `captured_at`; sweep ordering/age uses it rather than filesystem
  `LastWriteTime` (an old artifact ingested today otherwise looks newest). The sweep depends on
  that layout:
  files lying directly under the root or nested deeper are out of contract and it does not manage
  them.
- Both sweeps live in one saved script and are registered as one daily task. Snapshot rotation
  stays a **separate sweep from the 7-day failures sweep** and is never age-only:

```powershell
# ---- C:\ops\ai-price-tracker\retention-sweep.ps1 (1/2) — failure diagnostics ----
$failuresRoot = 'C:\ops\ai-price-tracker\diagnostics\failures'
if (Test-Path -LiteralPath $failuresRoot) {   # nothing to sweep before the first failure
  Get-ChildItem -LiteralPath $failuresRoot -Recurse -File |
    Where-Object { $_.LastWriteTime -lt (Get-Date).AddDays(-7) } | Remove-Item -Force
}

# ---- retention-sweep.ps1 (2/2) — snapshots ----
# Layout: snapshots\<entry-id>\<yyyyMMddTHHmmss.fffZ>-<scrape-run-id>-<state>.json.
$snapshotRoot = 'C:\ops\ai-price-tracker\snapshots'
$snapshotNamePattern = '^(?<stamp>\d{8}T\d{6}\.\d{3}Z)-[0-9A-Fa-f]{8}-[0-9A-Fa-f]{4}-[0-9A-Fa-f]{4}-[0-9A-Fa-f]{4}-[0-9A-Fa-f]{12}-(?<state>pending|accepted|held|discardable)$'
$snapshotTimestampFormat = 'yyyyMMddTHHmmss.fffZ'
$utcStyle = [Globalization.DateTimeStyles]::AssumeUniversal -bor `
  [Globalization.DateTimeStyles]::AdjustToUniversal
if (Test-Path -LiteralPath $snapshotRoot) {   # nothing to sweep before the first snapshot
  Get-ChildItem -LiteralPath $snapshotRoot -Directory | ForEach-Object {
    $snapshotItems = @(Get-ChildItem -LiteralPath $_.FullName -File -Filter '*.json' | ForEach-Object {
      if ($_.BaseName -notmatch $snapshotNamePattern) {
        Write-Warning "Out-of-contract snapshot left unmanaged: $($_.FullName)"
        return
      }
      $capturedAt = [DateTime]::ParseExact(
        $Matches.stamp, $snapshotTimestampFormat,
        [Globalization.CultureInfo]::InvariantCulture, $utcStyle
      )
      [pscustomobject]@{
        File = $_; CapturedAt = $capturedAt; State = $Matches.state.ToLowerInvariant()
      }
    })
    # Held/pending evidence survives until application reconciliation. Held attempts never displace
    # the two accepted baseline snapshots that define the current/predecessor chain.
    $protectedAcceptedPaths = @($snapshotItems |
      Where-Object State -eq 'accepted' | Sort-Object CapturedAt -Descending |
      Select-Object -First 2 | ForEach-Object { $_.File.FullName })
    foreach ($item in $snapshotItems) {
      if ($item.State -in @('held', 'pending')) { continue }
      if ($item.File.FullName -in $protectedAcceptedPaths) { continue }
      if ($item.CapturedAt -lt [DateTime]::UtcNow.AddDays(-35)) {
        Remove-Item -LiteralPath $item.File.FullName -Force
      }
    }
  }
}
```

After saving that script, register, smoke-test, and inspect its daily task once; do not include
these commands in the script:

```powershell
schtasks /Create /TN "AIPT-retention-sweep" /SC DAILY /ST 04:00 /RU "$env:USERNAME" /F `
  /TR "powershell -NoProfile -ExecutionPolicy Bypass -File C:\ops\ai-price-tracker\retention-sweep.ps1"
schtasks /Run /TN "AIPT-retention-sweep"   # smoke-test now; on a fresh tree it must delete nothing
# After the smoke run exits, record the XML and confirm Last Run Result is 0:
schtasks /Query /TN "AIPT-retention-sweep" /XML
schtasks /Query /TN "AIPT-retention-sweep" /V /FO LIST
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
   late/down state within one cycle (D-02). Still silent after a full cycle → read the worker and
   backup-snapshot-ingest logs; an ingest failure deliberately suppresses this ping.
5. **Serve reachable?** `tailscale status` + `tailscale serve status`, then open
   `https://<host>.<tailnet>.ts.net/api/v1/health` from another tailnet device. 200 on loopback but
   nothing over Serve = the Serve mapping was lost; re-arm it (§2d).

## 5. Verification — did the backup lane cover the gap?

Every attempt is logged to `scrape_runs` with its `lane` (D-12), so downtime is auditable:

```powershell
& {
$pgEnvNames = @(
  'PGHOST', 'PGPORT', 'PGDATABASE', 'PGUSER', 'PGPASSWORD', 'PGSSLMODE',
  'PGCHANNELBINDING', 'PGCONNECT_TIMEOUT', 'PGAPPNAME', 'PGOPTIONS', 'PGSSLROOTCERT'
)
$priorPgEnv = @{}
foreach ($pgEnvName in $pgEnvNames) {
  $priorValue = Get-Item -LiteralPath "Env:$pgEnvName" -ErrorAction SilentlyContinue
  $priorPgEnv[$pgEnvName] = @{ Present = $false; Value = $null }
  if ($null -ne $priorValue) {
    $priorPgEnv[$pgEnvName].Present = $true
    $priorPgEnv[$pgEnvName].Value = $priorValue.Value
  }
}
try {
$ErrorActionPreference = 'Stop'
$urlLine = Get-Content -LiteralPath C:\ops\ai-price-tracker\.env |
  Where-Object { $_ -match '^\s*DATABASE_DIRECT_URL\s*=' } | Select-Object -First 1
if (-not $urlLine) { throw 'DATABASE_DIRECT_URL is missing from .env' }
$dbUrl = ($urlLine -replace '^\s*DATABASE_DIRECT_URL\s*=\s*', '').Trim().Trim('"').Trim("'")
if (-not $dbUrl -or $dbUrl -match '-pooler\.') { throw 'Use a non-pooled Neon URL for psql' }
$u = $null
$uriValid = [System.Uri]::TryCreate($dbUrl, [UriKind]::Absolute, [ref]$u)
if (-not $uriValid -or $u.Scheme -notin @('postgres', 'postgresql')) {
  throw 'DATABASE_DIRECT_URL must be an absolute postgres:// or postgresql:// URL'
}
$dbUri = $u
$userInfo = $dbUri.UserInfo
$userSeparator = $userInfo.IndexOf(':')
if ($userSeparator -lt 0) { throw 'DATABASE_DIRECT_URL must include a database user and password' }
$pgUser = [System.Uri]::UnescapeDataString($userInfo.Substring(0, $userSeparator))
$pgPassword = [System.Uri]::UnescapeDataString($userInfo.Substring($userSeparator + 1))
$pgDatabase = [System.Uri]::UnescapeDataString($dbUri.AbsolutePath.TrimStart('/'))
if (-not $dbUri.Host -or -not $pgUser -or -not $pgPassword -or -not $pgDatabase) {
  throw 'DATABASE_DIRECT_URL must include a host, user, password, and database'
}
$pgPort = if ($dbUri.Port -lt 0) { '5432' } else { [string]$dbUri.Port }
$pgSslMode = 'require'
$pgChannelBinding = $null
$pgConnectTimeout = $null
$pgAppName = $null
$pgOptions = $null
foreach ($queryPart in ($dbUri.Query.TrimStart('?') -split '&')) {
  if (-not $queryPart) { continue }
  $queryPieces = $queryPart -split '=', 2
  $queryName = [System.Uri]::UnescapeDataString($queryPieces[0])
  $queryValue = if ($queryPieces.Count -eq 2) {
    [System.Uri]::UnescapeDataString($queryPieces[1])
  } else {
    $null
  }
  switch ($queryName.ToLowerInvariant()) {
    'sslmode' {
      if ([string]::IsNullOrEmpty($queryValue)) {
        throw 'DATABASE_DIRECT_URL sslmode query parameter must include a value'
      }
      $pgSslMode = $queryValue
    }
    'channel_binding' {
      if ([string]::IsNullOrEmpty($queryValue)) {
        throw 'DATABASE_DIRECT_URL channel_binding query parameter must include a value'
      }
      $pgChannelBinding = $queryValue
    }
    'connect_timeout' {
      if ([string]::IsNullOrEmpty($queryValue)) {
        throw 'DATABASE_DIRECT_URL connect_timeout query parameter must include a value'
      }
      $pgConnectTimeout = $queryValue
    }
    'application_name' {
      if ([string]::IsNullOrEmpty($queryValue)) {
        throw 'DATABASE_DIRECT_URL application_name query parameter must include a value'
      }
      $pgAppName = $queryValue
    }
    'options' {
      if ([string]::IsNullOrEmpty($queryValue)) {
        throw 'DATABASE_DIRECT_URL options query parameter must include a value'
      }
      $pgOptions = $queryValue
    }
    default {
      throw "Unsupported DATABASE_DIRECT_URL query parameter '$queryName'"
    }
  }
}
$pgSslMode = $pgSslMode.ToLowerInvariant()
if ($pgSslMode -in @('disable', 'allow', 'prefer')) {
  throw "DATABASE_DIRECT_URL sslmode '$pgSslMode' is weaker than require and is not accepted"
}
if ($pgSslMode -notin @('require', 'verify-ca', 'verify-full')) {
  throw "DATABASE_DIRECT_URL sslmode '$pgSslMode' is not recognized"
}
$pgSslMode = 'verify-full'
$caCertDir = 'C:\ops\ai-price-tracker\certs'
$caCertPath = Join-Path $caCertDir 'isrgrootx1.pem'
if (-not (Test-Path -LiteralPath $caCertPath)) {
  throw "Missing $caCertDir\isrgrootx1.pem - download it once (backup setup, section 3)"
}
$expectedCaFileSha256 = '22B557A27055B33606B6559F37703928D3E4AD79F110B407D04986E1843543D1'
$actualCaFileSha256 = (Get-FileHash -LiteralPath $caCertPath -Algorithm SHA256).Hash
if ($actualCaFileSha256 -ne $expectedCaFileSha256) {
  throw "Unexpected ISRG Root X1 file SHA-256 at $caCertPath; do not use this trust anchor"
}
$env:PGHOST = $dbUri.Host
$env:PGPORT = $pgPort
$env:PGDATABASE = $pgDatabase
$env:PGUSER = $pgUser
$env:PGPASSWORD = $pgPassword
$env:PGSSLMODE = $pgSslMode
$env:PGSSLROOTCERT = '/certs/isrgrootx1.pem'
$dockerEnvArgs = @(
  '--env', 'PGHOST', '--env', 'PGPORT', '--env', 'PGDATABASE', '--env', 'PGUSER',
  '--env', 'PGPASSWORD', '--env', 'PGSSLMODE', '--env', 'PGSSLROOTCERT'
)
if ($null -ne $pgChannelBinding) {
  $env:PGCHANNELBINDING = $pgChannelBinding
  $dockerEnvArgs += @('--env', 'PGCHANNELBINDING')
}
if ($null -ne $pgConnectTimeout) {
  $env:PGCONNECT_TIMEOUT = $pgConnectTimeout
  $dockerEnvArgs += @('--env', 'PGCONNECT_TIMEOUT')
}
if ($null -ne $pgAppName) {
  $env:PGAPPNAME = $pgAppName
  $dockerEnvArgs += @('--env', 'PGAPPNAME')
}
if ($null -ne $pgOptions) {
  $env:PGOPTIONS = $pgOptions
  $dockerEnvArgs += @('--env', 'PGOPTIONS')
}
$auditSql = "select lane, count(*) as runs, max(started_at) as latest from scrape_runs " +
  "where started_at > now() - interval '24 hours' group by lane;"
docker run --rm @dockerEnvArgs `
  -v "${caCertDir}:/certs:ro" postgres:17 `
  psql -v ON_ERROR_STOP=1 -c "$auditSql"
if ($LASTEXITCODE -ne 0) { throw "psql failed with exit code $LASTEXITCODE" }
# per-run detail: add `where lane = 'backup' order by started_at desc limit 20`
}
finally {
  foreach ($pgEnvName in $pgEnvNames) {
    $prior = $priorPgEnv[$pgEnvName]
    if ($prior.Present) {
      Set-Item -LiteralPath "Env:$pgEnvName" -Value $prior.Value
    } else {
      Remove-Item -LiteralPath "Env:$pgEnvName" -ErrorAction SilentlyContinue
    }
  }
}
}
```

Zero `backup` rows is not automatically a failure: the lane skips sources the primary refreshed less
than 3 h ago (D-02), so a short outage legitimately produces a correct no-op run.

Then read **both** dead-man checks (D-02) together:

| primary | backup | reading |
|---|---|---|
| green | green | normal operation |
| red | green | primary unavailable **or** unhealthy — the PC was down, or a cycle withheld its ping (failed snapshot ingest, D-12) — while the backup lane carried the checks |
| green | red | Actions cron disabled or failing (public-repo 60-day inactivity, N-04) → re-arm |
| red | red | no lane is confirming work: primary down or unhealthy **and** the Actions cron silent — treat as an incident |

Row 2 has two causes, so it never proves on its own that the machine was down: separate them with
§4 steps 1–3 (containers, API, bot) plus the worker and backup-snapshot-ingest logs. A ping
suppressed by ingest stays suppressed until ingest succeeds, so an artifact that never validates can
hold the check red for the rest of its 30-day lifetime — an ingest failure is an incident to work,
not a state to wait out. Giving that case a bounded escape is open decision **O-20**.

The backup check pings even on a correct no-op, so a green backup check with zero backup rows still
proves the workflow ran (D-02, N-04). Record any outage that moves the monthly picture: primary
uptime below ~95% is the documented reopen trigger for D-02/D-03.
