# Coeus AI — Private Technical Study Pack

This pack is a private study/reference set for understanding the Coeus AI source architecture. It is intended for the project owner and trusted reviewers, not for the public portfolio.

The goal is to make the project easier to re-learn, explain, and defend in technical review conversations. The documents are based on the block-by-block source review and the clean local `scc` scan of the native C/C++ source tree.

## Clean codebase metrics

The clean scan excluded generated Tree-sitter parser code, static assets/fonts, build output, cache directories, vendor/external directories, and environment/secrets files.

| Metric | Value |
|---|---:|
| Files analyzed | 364 |
| Total lines | 181,781 |
| Code lines | 144,110 |
| Reported complexity | 36,717 |
| C++ files | 177 |
| Header files | 187 |

These numbers are codebase-scale evidence. They are not a quality score.

## Recommended reading order

1. `00_START_HERE_SYSTEM_GUIDE.md`
2. `01_SOURCE_MAP_RUNTIME_SURFACES.md`
3. `02_GUI_RENDERING_AND_PRODUCT_SURFACE.md`
4. `03_PROJECT_CONTEXT_RETRIEVAL_LATTICE_SUMMARIES.md`
5. `04_TURN_COMPILER_POLICY_AND_CANONICAL_TRUTH.md`
6. `05_PROMPTING_CONTRACTS_AND_CONTEXT_PACKING.md`
7. `06_AGENT_RUNTIME_VALIDATION_AND_POSTPROCESS.md`
8. `07_TOOLS_COMMANDS_AND_PLANNING.md`
9. `08_TELEMETRY_EVALS_AND_HEADLESS_RUNTIME.md`
10. `09_PRIVATE_REVIEW_BOUNDARY_AND_PUBLIC_SPLIT.md`
11. `10_TECHNICAL_QA_AND_GLOSSARY.md`
12. `11_PERFORMANCE_NOTES_TEMPLATE.md`

For a more readable browser view, open:

```text
coeus_private_study_hub.html
```

## Private/public note

Public material should describe the architecture at a high level: native GUI, project-aware runtime, retrieval, source grounding, tools, telemetry, and private headless evaluation. This study pack can go much deeper: turn compiler details, marker truth, telemetry planes, headless protocol, private file reading path, and internal subsystem relationships.
