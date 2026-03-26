# OpenShift / Tekton Manifests (lx17a)

This repo contains two related Tekton representations of the Bistro pipeline:

- **`lec-17-bistro-pipeline-flawed.yaml`**: the single YAML file you apply for the lx17a assignment (includes the corrected pipeline and tasks).
- **`openshift/*.yaml`**: an earlier “Mermaid-to-Tekton mapping” version used for discussion; it intentionally preserves flaws (e.g., `retries: 0`, discard-on-failure).

## Files (what to apply)

- **For the assignment**: apply `lec-17-bistro-pipeline-flawed.yaml`.
- **Optional/reference only**: `openshift/02-serviceaccount.yaml`, `openshift/10-tasks.yaml`, `openshift/20-pipeline.yaml`, `openshift/30-pipelinerun.yaml`.

## Apply (assignment commands)

```bash
oc apply -f lec-17-bistro-pipeline-flawed.yaml -n ds551-2026-spring-a5f0d1
oc get pipelines.tekton.dev bistro-pipeline -n ds551-2026-spring-a5f0d1
oc get tasks.tekton.dev validate-order route-and-enrich -n ds551-2026-spring-a5f0d1
```

Note: some clusters also install **Kubeflow Pipelines** (`pipelines.pipelines.kubeflow.org`), which can cause `oc get pipeline ...` to resolve to the wrong API group and fail with RBAC. Using `pipelines.tekton.dev` / `tasks.tekton.dev` avoids that ambiguity.

## What was fixed (2 flaws) and where

These fixes correspond to the flaw types listed in `mermaid_chart.md` and cite the lettered Mermaid boxes.

- **Fix 1: `dead-letter-missing`** (Mermaid flaw location: `[B]` and downstream loss at `[D]`)
  - **Change**: `validate-order` no longer “logs + discards” invalid input; it emits `invalid-order` + `validation-error` results for a DLQ path.
  - **DLQ path**: the pipeline routes invalid events to a dedicated `dead-letter` task (simulated DLQ handler).
  - **YAML references**:
    - `validate-order` results + logic: `lec-17-bistro-pipeline-flawed.yaml:L18-L78`
    - `dead-letter` task: `lec-17-bistro-pipeline-flawed.yaml:L171-L203`
    - pipeline branch to DLQ: `lec-17-bistro-pipeline-flawed.yaml:L240-L253`

- **Fix 2: `retry-missing`** (Mermaid flaw location: `[H]` and error handling at `[D]`)
  - **Change**: `route-and-enrich` no longer catches/skips write failures; it fails on a simulated transient failure so Tekton retries can occur.
  - **Retries enabled**: pipeline sets `retries: 2` for the `route-and-enrich` task.
  - **YAML references**:
    - failure is not swallowed (raises): `lec-17-bistro-pipeline-flawed.yaml:L142-L149`
    - retries enabled: `lec-17-bistro-pipeline-flawed.yaml:L227-L235`

## Inspect (helpful extras)

```bash
oc get pipelines.tekton.dev bistro-pipeline -n ds551-2026-spring-a5f0d1
oc get tasks.tekton.dev validate-order route-and-enrich dead-letter -n ds551-2026-spring-a5f0d1
```
