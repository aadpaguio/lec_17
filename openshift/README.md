# OpenShift / Tekton Manifests

This directory contains a runnable Tekton version of the order-processing flow from `mermaid_chart.md`.

## Files

- `02-serviceaccount.yaml`: service account used by the sample `PipelineRun`.
- `10-tasks.yaml`: reusable `Task` resources for validation and routing/enrichment.
- `20-pipeline.yaml`: the `Pipeline` wiring the tasks together with `retries: 0`.
- `30-pipelinerun.yaml`: a sample `PipelineRun` with one valid food order.

## Apply

```bash
oc apply -f openshift/02-serviceaccount.yaml
oc apply -f openshift/10-tasks.yaml
oc apply -f openshift/20-pipeline.yaml
oc apply -f openshift/30-pipelinerun.yaml
```

## Inspect

```bash
oc get pipelineruns -n ds551-2026-spring-a5f0d1
oc describe pipelinerun bistro-order-processing-sample -n ds551-2026-spring-a5f0d1
oc logs -n ds551-2026-spring-a5f0d1 -l tekton.dev/pipelineRun=bistro-order-processing-sample --all-containers
```

## Notes
- The manifests intentionally keep `retries: 0` to match the Mermaid diagram.
- **Original flaw: `dead-letter-missing` (events lost permanently)** (Mermaid: flaw at `[B]` and `[D]`). Fix/change modeled (where in YAML):
  - Invalid JSON / missing required fields are **logged and discarded** by writing `is-valid=false` and an empty `valid-order`, then exiting successfully. See `openshift/10-tasks.yaml:L27-L74` (the `validate-order` step logic at `[B]`).
  - The pipeline **drops invalid events on the floor** by gating `[D]` behind a `when` condition (`is-valid` must be `"true"`). See `openshift/20-pipeline.yaml:L31-L47`.
  - Downstream “write failure → caught + skipped” is modeled by catching exceptions and writing a `route-status` message, then exiting successfully (no retry, no DLQ). See `openshift/10-tasks.yaml:L137-L171` (the `route-and-enrich` step logic at `[D]`).
- **Original flaw: `retry-missing` (transient failures cause permanent loss)** (Mermaid: flaw at `[H]`, plus `[D]`). Fix/change modeled (where in YAML):
  - “No retries” is enforced via `retries: 0` on both pipeline tasks. See `openshift/20-pipeline.yaml:L23-L24` and `openshift/20-pipeline.yaml:L31-L35` (mirrors `[H]`).
  - A transient downstream failure is explicitly **not retried**; it is caught + skipped when `simulate_write_failure=true`. See `openshift/10-tasks.yaml:L145-L171` (mirrors `[D] write failure → caught + skipped`).
- **Original flaw: `idempotency-violation` (reprocessing creates duplicates)** (Mermaid: flaw at `[A]` and `[D]`). Fix/change modeled (where in YAML):
  - There is **no key/dedupe mechanism**; input is treated as an opaque JSON string param (`order-json`) and no dedupe state is kept. See `openshift/10-tasks.yaml:L13-L26` and `openshift/20-pipeline.yaml:L14-L21`.
  - Enrichment adds a runtime timestamp `processed_at`, making reruns produce different outputs for the same input. See `openshift/10-tasks.yaml:L137-L143` (mirrors `[D] adds: processed_at`).
- **Original flaw: `schema-evolution-risk` (hardcoded assumptions break)** (Mermaid: flaw at `[B]` and `[D]`). Fix/change modeled (where in YAML):
  - Validation hardcodes required fields (`item_id`, `table_number`, `timestamp`) and discards events missing them. See `openshift/10-tasks.yaml:L42-L69` (mirrors `[B] checks: ...`).
  - Routing hardcodes known types (`food` and `drink`) and treats everything else as unsupported. See `openshift/10-tasks.yaml:L142-L167` (mirrors `[D] routes order_type: ...`).
- Routing produces simulated downstream outputs through Tekton task results (e.g. `kitchen-order`, `bar-order`, `manager-alert`) instead of real Kafka topics.
