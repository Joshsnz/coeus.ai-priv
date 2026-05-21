# Performance Notes Template

This document is a placeholder for measured runtime numbers. Do not publish performance claims until they are measured under repeatable scenarios.

## Measurement principles

Every measurement should include:

- date
- machine hardware
- OS version
- build type
- executable/version/commit if available
- scenario
- measurement method
- observed values
- notes/limitations

## Suggested scenarios

### Scenario 1 — App launched, idle

Steps:

1. Start release build.
2. Wait 30 seconds.
3. Record memory and CPU.

### Scenario 2 — Demo project loaded

Steps:

1. Start release build.
2. Load demo project.
3. Wait for project state to stabilize.
4. Record memory and CPU.

### Scenario 3 — Project indexed / file context bootstrapped

Steps:

1. Load demo project.
2. Trigger/confirm file context scan/index.
3. Wait for quiescence.
4. Record memory and CPU.

### Scenario 4 — After repository QA turns

Steps:

1. Load demo project.
2. Ask several repo-grounded questions.
3. Record memory and CPU.

### Scenario 5 — Headless eval run

Steps:

1. Run headless benchmark.
2. Capture wall-clock duration.
3. Capture peak memory if possible.
4. Record scenario count and turn count.

## PowerShell process snapshot

Find the process:

```powershell
Get-Process | Where-Object { $_.ProcessName -like "*Coeus*" -or $_.ProcessName -like "*Agent*" -or $_.ProcessName -like "*skia*" }
```

Capture memory:

```powershell
$P = Get-Process <ProcessName>
$P | Select-Object ProcessName, Id, CPU,
@{Name="WorkingSetMB";Expression={[math]::Round($_.WorkingSet64/1MB,2)}},
@{Name="PrivateMemoryMB";Expression={[math]::Round($_.PrivateMemorySize64/1MB,2)}},
@{Name="PagedMemoryMB";Expression={[math]::Round($_.PagedMemorySize64/1MB,2)}}
```

## Result table template

| Scenario | Working set MB | Private memory MB | CPU notes | Build | Notes |
|---|---:|---:|---|---|---|
| App launched, idle | TBD | TBD | TBD | TBD | TBD |
| Demo project loaded | TBD | TBD | TBD | TBD | TBD |
| Project indexed | TBD | TBD | TBD | TBD | TBD |
| After repo QA turns | TBD | TBD | TBD | TBD | TBD |
| Headless eval run | TBD | TBD | TBD | TBD | TBD |

## Public phrasing after measurement

Use cautious wording:

> Local performance notes are available in the private review package. Measurements are scenario-based and include launch, project load/index, repo-QA turns, and headless eval runs.

Avoid broad claims like “fast” or “lightweight” unless the data supports them.
