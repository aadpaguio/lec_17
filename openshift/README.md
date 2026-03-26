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
- **Original flaw: `dead-letter-missing` (events lost permanently)** (Mermaid: flaw at `[B]` and `[D]`). Fix/change modeled: “invalid → logged + discarded” happens at `[B]`, and “write failure → caught + skipped” is modeled at `[D]`, with no DLQ/quarantine outlet.
- **Original flaw: `retry-missing` (transient failures cause permanent loss)** (Mermaid: flaw at `[H]`, plus `[D]`). Fix/change modeled: “no retries” is captured by `[H] PipelineRun · no retries` (mirrored by task `retries: 0`), and transient downstream errors are handled via `[D] write failure → caught + skipped` (catch-and-skip instead of retry).
- **Original flaw: `idempotency-violation` (reprocessing creates duplicates)** (Mermaid: flaw at `[A]` and `[D]`). Fix/change modeled: enrichment logic in `[D]` adds `processed_at` at runtime (so reruns/replays yield different enriched payloads without any dedupe key from `[A]`).
- **Original flaw: `schema-evolution-risk` (hardcoded assumptions break)** (Mermaid: flaw at `[B]` and `[D]`). Fix/change modeled: `[B] validate-order` enforces required fields (`item_id`, `table_number`, `timestamp`) so only the expected subset reaches the routing/enrichment step `[D]`.
- Routing produces simulated downstream outputs through Tekton task results (e.g. `kitchen-order`, `bar-order`, `manager-alert`) instead of real Kafka topics.
