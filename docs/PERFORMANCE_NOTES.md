# Coeus AI — Performance Notes

Performance measurements should be added only when tied to a specific build, machine, and scenario.

## Planned measurement scenarios

1. App launched, no project loaded.
2. Demo project loaded.
3. Project scan/index completed.
4. Several repository QA turns completed.
5. Headless eval scenario run.

## Suggested metrics

- Executable/release package size.
- Startup time.
- Idle memory after launch.
- Memory after project load.
- Memory after indexing.
- Memory after several QA turns.
- CPU idle behaviour.
- Headless scenario runtime.

## Suggested PowerShell command

```powershell
Get-Process AgentGui_skia | Select-Object ProcessName, Id, CPU,
@{Name="WorkingSetMB";Expression={[math]::Round($_.WorkingSet64/1MB,2)}},
@{Name="PrivateMemoryMB";Expression={[math]::Round($_.PrivateMemorySize64/1MB,2)}},
@{Name="PagedMemoryMB";Expression={[math]::Round($_.PagedMemorySize64/1MB,2)}}
```

## Reporting format

| Scenario | Build | Machine | Working Set MB | Private MB | Notes |
|---|---|---|---:|---:|---|
| App launched idle | TBD | TBD | TBD | TBD | TBD |
| Project loaded | TBD | TBD | TBD | TBD | TBD |
| Project indexed | TBD | TBD | TBD | TBD | TBD |
| After QA turns | TBD | TBD | TBD | TBD | TBD |

## Notes

Do not infer performance from line count or complexity metrics. Use measured values only.
