# Coeus AI — Telemetry and Evaluation Notes

## Purpose

Telemetry supports debugging, traceability, private evaluation, and report generation. It is also important for preventing internal artifacts from leaking into user-facing answers.

## Main areas

```text
core/telemetry/
core/telemetry/schema/
headless/
```

## Telemetry responsibilities

- activity feed,
- artifact store,
- CLE exporter,
- marker truth export,
- run index,
- run trace,
- tool export,
- canonical JSON,
- schema objects for traces and bundles.

Representative files:

```text
core/telemetry/cle_exporter.cpp
core/telemetry/cle_exporter_marker_truth.cpp
core/telemetry/run_trace.cpp
core/telemetry/run_index.cpp
core/telemetry/tool_exporter.cpp
core/telemetry/trace.cpp
core/telemetry/schema/cle_bundle_v1.h
core/telemetry/schema/run_trace_v1.h
core/telemetry/schema/tool_trace_v1.h
```

## Headless runtime

The headless runtime provides a non-GUI execution surface for automation and private evaluation.

Representative files:

```text
headless/headless_main.cpp
headless/headless_protocol.cpp
headless/headless_runtime_bridge.cpp
headless/headless_session.cpp
```

## Relation to public reports

The public RepoGrounding HTML reports are generated from a private headless evaluation harness. The reports are public-facing artifacts, while raw traces and private harness internals remain private.

## Evaluation behaviours

The public reports test:

- repository bootstrap,
- source-grounded QA,
- source-flow tracing,
- semantic comparison,
- negative grounding,
- multi-turn continuity,
- diff-route discipline,
- route retention,
- internal artifact suppression.

## Private boundary

Raw telemetry, local trace paths, private run artifacts, provider configuration, and private project workspaces should not be published publicly.
