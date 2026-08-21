# M2 Godot demo runbook

This walkthrough runs the real InstLeague1 M2 slice on a developer machine. It uses a locally built private content pack and locally converted assets. Do not copy either artifact into a public repository.

M2 connects the Godot client directly to the shard over QUIC. The auth service handles accounts and tickets over HTTP. Do not start `cmd/gateway`; it is not in the M2 player path. [ADR 0039](../../adr/0039-gateway-and-session-model.md) moves the public session edge to the gateway in M3.

## Requirements

- Go 1.27
- golangci-lint 2.13.0
- .NET 10 SDK
- Godot 4.7.2 .NET
- Docker Desktop with Compose `--wait` support
- side-by-side `server`, `client`, and `infra` checkouts
- the classic conversion mounted at `client/converted/assets/classic-1.1`
- a private M2 pack built from the developer's own data and the M2 overlays

Open every PowerShell terminal in the directory that contains the checkouts. Paste this bootstrap block into each terminal; PowerShell variables do not carry into a new terminal:

```powershell
$Server = (Resolve-Path './server').Path
$Client = (Resolve-Path './client').Path
$Infra = (Resolve-Path './infra').Path
$Pack = (Resolve-Path (Read-Host 'Private M2 pack directory')).Path
$Godot = (Resolve-Path (Read-Host 'Godot .NET console executable')).Path

function Assert-NativeSuccess([string]$Step) {
    if ($LASTEXITCODE -ne 0) { throw "$Step failed with exit code $LASTEXITCODE" }
}
```

The private pack directory must contain `manifest.json` and `tables/`. Keep it outside every public worktree.

## Check the code and protocol copies

Run these commands from the server checkout:

```powershell
Set-Location $Server
pwsh -NoProfile -File ./scripts/proto-lock.ps1 -Check
Assert-NativeSuccess 'Server protocol lock'
pwsh -NoProfile -File ./scripts/test.ps1
Assert-NativeSuccess 'Server tests'
pwsh -NoProfile -File ./scripts/lint.ps1
Assert-NativeSuccess 'Server lint'
pwsh -NoProfile -File ./scripts/build.ps1
Assert-NativeSuccess 'Server build'
```

Run these commands from the client checkout:

```powershell
Set-Location $Client
pwsh -NoProfile -File ./scripts/verify-proto-lock.ps1
Assert-NativeSuccess 'Client protocol lock'
pwsh -NoProfile -File ./scripts/sync-proto.ps1 -Check -ServerRepo $Server
Assert-NativeSuccess 'Cross-repository protocol copy'
dotnet build SarnautCore.sln
Assert-NativeSuccess 'Client build'
dotnet test SarnautCore.sln
Assert-NativeSuccess 'Client tests'
```

Stop if either protocol check fails. `server/proto` is canonical; repair the client copy with `client/scripts/sync-proto.ps1` before continuing.

## Prove the headless slice

Run the committed synthetic fixture first. This is the same private-data-free chain used by server CI:

```powershell
Set-Location $Server
pwsh -NoProfile -File ./scripts/m2-slice-smoke.ps1
if ($LASTEXITCODE -ne 0) { throw "Synthetic M2 slice failed with exit code $LASTEXITCODE" }
```

Then run the identical gameplay chain against the private InstLeague1 pack:

```powershell
pwsh -NoProfile -File ./scripts/m2-slice-smoke.ps1 -RealPack $Pack
if ($LASTEXITCODE -ne 0) { throw "Real-pack M2 slice failed with exit code $LASTEXITCODE" }
```

Each successful run prints one `PASS` line per step, including host, connect, enter, accept, move, cast, kill, loot, turn-in, logout, and reconnect. The real-pack run must identify pack `aeb47a3e008b5b863ef9c11be5c7d011fa63533c8780d353d49ee6686e6d7a64`, report nine deliberately skipped `count-special` quests, complete three zombie-warrior kills and the starting-inventory item objective, and restore the moved position and turned-in quest after reconnect.

Confirm that the harness also exposes a broken module instead of producing a false green:

```powershell
pwsh -NoProfile -File ./scripts/m2-slice-smoke.ps1 -BreakModule combat -Timeout 20s
if ($LASTEXITCODE -eq 0) { throw 'Broken combat module unexpectedly passed' }
pwsh -NoProfile -File ./scripts/m2-slice-smoke.ps1 -RealPack $Pack -BreakModule combat -Timeout 20s
if ($LASTEXITCODE -eq 0) { throw 'Broken real-pack combat module unexpectedly passed' }
```

Both negative runs must exit non-zero with a `FAIL cast` line. A failure at any other step needs diagnosis; do not treat it as proof of the intended guard.

Finally, run the five-minute capacity gate:

```powershell
pwsh -NoProfile -File ./scripts/m2-slice-smoke.ps1 -Soak -Clients 20 -Duration 5m -TickOverrunThreshold 33.333333ms
if ($LASTEXITCODE -ne 0) { throw "M2 soak failed with exit code $LASTEXITCODE" }
```

The soak prints one machine-readable `SOAK` line followed by `PASS soak`. It must report 20 clients, zero tick overruns beyond 33.333333 ms, zero save-worker drops, start/peak/end goroutine counts, and an end-to-start goroutine delta no greater than two. Preserve that line with the demo evidence.

## Check the local visual build

The client checkout owns the probe list, and `scripts/visual-gate.ps1` runs all of it. The gate starts one probe scene at a time, because concurrent Godot processes share the converted-scene cache and can corrupt one another's results, and it fails a probe that exits non-zero, exceeds its time budget, leaks instances, or writes an `ERROR` line. Import first, then run the gate:

```powershell
Set-Location $Client
& $Godot --headless --import --path $Client
Assert-NativeSuccess 'Godot import'
$VisualGate = Join-Path $env:TEMP 'sarnaut-m2-visual-gate'
$env:SARNAUT_LIGHTING_PROBE_PREFIX = Join-Path $VisualGate 'lighting'
pwsh -NoProfile -File ./scripts/visual-gate.ps1 -Godot $Godot -OutputDirectory $VisualGate
Assert-NativeSuccess 'Visual gate'
Get-Item (Join-Path $VisualGate 'lighting-shadowed.png'),
    (Join-Path $VisualGate 'lighting-unshadowed.png'),
    (Join-Path $VisualGate 'lighting-ambient-only.png'),
    (Join-Path $VisualGate 'player-appearance-front.png'),
    (Join-Path $VisualGate 'player-appearance-back.png')
```

The gate prints one `PASS` line per probe, ends with a `visual-gate: OK` line, and leaves every probe's stdout and stderr in the output directory for diagnosis.

Do not begin the human walkthrough unless every command exits zero. Inspect the three lighting PNGs: the shadowed frame must remain readable, the unshadowed frame must visibly remove cast shadows, and the ambient-only frame must visibly remove direct sunlight. Inspect the two player PNGs too; both the front and back must show the equipped converted character without missing surfaces or placeholder geometry. The gate also requires four terrain tiles, layered Up and Down terrain, 41 authored placements, 36 rendered statics, zero unresolved visual placements, converted player and creature models, playing animations, a grounded player, directional-light coverage, cast shadows, and non-empty rendered zone and player frames. Capsule fallbacks, a T-pose, black terrain, missing textures, or an empty sky fail the gate.

## Start infrastructure

```powershell
docker compose -f (Join-Path $Infra 'compose/docker-compose.yml') up -d --wait
Assert-NativeSuccess 'Infrastructure startup'
```

After the bootstrap block, paste this service configuration into the migration, auth, and shard terminals:

```powershell
$env:SARNAUT_CONTENT_PACK = $Pack
$env:SARNAUT_CONTENT_SKIP_UNSUPPORTED_QUESTS = 'true'
$env:SARNAUT_POSTGRES_DSN = 'postgres://sarnaut:sarnaut_dev@127.0.0.1:5433/sarnaut?sslmode=disable'
$env:SARNAUT_VALKEY_ADDRESS = '127.0.0.1:6379'
$env:SARNAUT_NATS_URL = 'nats://127.0.0.1:4222'
$env:SARNAUT_OTEL_ENDPOINT = ''
```

Apply the schema once from the server checkout:

```powershell
Set-Location $Server
go run ./cmd/migrate up
Assert-NativeSuccess 'Database migration'
```

In a second PowerShell terminal, paste the bootstrap block and service configuration, then start auth:

```powershell
Set-Location $Server
go run ./cmd/auth
```

In a third PowerShell terminal, paste the bootstrap block and service configuration, then start the shard in the explicit M2 demo composition:

```powershell
Set-Location $Server
go run ./cmd/shard -m2-demo
```

`-m2-demo` is a local-development composition, not a content fallback. It refuses to start if the real pack lacks the enabled chargen option, its terrain-tile origin, quest giver, quest targets, ability, or objective item. It derives the canonical wire spawn from the chargen row and its tile origin, places three runtime target spawns on that starting floor, and grants the objective item's required quantity to a fresh demo character. It does not edit the pack or write setup rows directly to PostgreSQL.

Wait for auth and shard readiness:

```powershell
Invoke-WebRequest http://127.0.0.1:8082/readyz
Invoke-WebRequest http://127.0.0.1:8081/readyz
```

Both commands must return HTTP 200.

## Prove the live spawn and grounding

In a fourth PowerShell terminal, paste the bootstrap block, then run the production auth, ticket, session, coordinate-mapping, and rendering path through the production Forward+ renderer. Keep every other Godot process stopped until this probe exits:

```powershell
Set-Location $Client
$env:SARNAUT_AUTH_ADDRESS = 'http://127.0.0.1:8083'
$env:SARNAUT_SERVER_ADDRESS = '127.0.0.1:4242'
$env:SARNAUT_CONTENT_PACK = $Pack
$env:SARNAUT_PROBE_ZONE = 'inst-league1'
$env:SARNAUT_PROBE_MAP = 'Inst_LeagueStart'
$ProbeId = -join ((1..10) | ForEach-Object { [char](97 + (Get-Random -Maximum 6)) })
$env:SARNAUT_PROBE_EMAIL = "ground-$ProbeId@example.invalid"
$env:SARNAUT_PROBE_PASSWORD = "ground-password-$ProbeId"
$env:SARNAUT_PROBE_CHARACTER = "Ground$ProbeId"
$env:SARNAUT_PROBE_SCREENSHOT = Join-Path $env:TEMP 'sarnaut-m2-live-grounding.png'
& $Godot --display-driver windows --path $Client --scene res://tests/zone_online_probe.tscn
Assert-NativeSuccess 'Live auth, spawn, grounding, and rendering probe'
Get-Item $env:SARNAUT_PROBE_SCREENSHOT
```

The probe must request the zone supplied by auth, retain the actual `EnterZoneResponse.spawn_position`, map that wire value through the production coordinate frame, and find authored support immediately below the rendered player's feet. It must not compare against a hardcoded pre-globalized point. It also rejects missing terrain tiles, statics, converted models, animations, lighting, nameplates, health bars, or targeting. Inspect the saved screenshot before continuing.

## Start Godot

In the same fourth terminal, run:

```powershell
& $Godot --path $Client
```

Do not pipe Godot output. On Windows, piping can leave the .NET editor process running after PowerShell believes it stopped.

## Play the slice

1. Choose **Play**.
2. Enter a new email address and a password of at least eight characters, then choose **Create account**.
3. Choose **Create a character**. Keep the server-provided League Warrior option selected, enter a unique 3–16-letter name with one uppercase initial and lowercase remaining letters, and choose **Create**. Avoid the reserved substrings `gm`, `admin`, and `sarnaut`.
4. Select the new character and choose **Enter the world**.
5. Confirm that InstLeague1 renders with textured two-sided terrain, lit statics, a grounded equipped player, converted creatures, and playing idle animations.
6. Press `Tab` until the nearby quest giver is selected. Press `E`, then choose **Accept**. The objective tracker must show one item already held and three living targets remaining.
7. Press `Tab` to select a living target. Press `1` until it is defeated. The target frame must count health down without jumping back to an older value.
8. Keep the corpse selected, press `E`, then choose **Take all**. The loot window must close after the authoritative bag update.
9. Repeat the target, cast, kill, and loot steps for the other two targets. The quest tracker must become complete after the third kill.
10. Return to the quest giver, select it with `Tab`, press `E`, then choose **Complete**. The quest must leave the tracker and the reward items must appear in the bags opened with `I`.
11. Press `Esc` until the client leaves the zone. Select the same character and choose **Enter the world** again. Open bags with `I` and quests with `J`. The position and reward items must persist, and the completed quest must not return to the active quest list.

The controls used above are `W`, `A`, `S`, `D` to move, `Tab` to select, `1` to cast, `E` to interact, `I` for bags, `J` for the quest log, and `Esc` to close the top window, release the pointer, or leave the zone.

## Stop the local stack

Stop the auth and shard commands with `Ctrl+C`. If the infrastructure was not already running before this walkthrough, stop it too:

```powershell
docker compose -f (Join-Path $Infra 'compose/docker-compose.yml') down
```
