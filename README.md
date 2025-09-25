# Dynamo Actions

GitHub Actions workflows for building container images and deploying [NVIDIA Dynamo](https://github.com/ai-dynamo/dynamo) components to Kubernetes.

## Workflows overview

| Workflow          | Purpose                                                                                                              | Trigger             | Key inputs                                                              | Outputs                               |
| ----------------- | -------------------------------------------------------------------------------------------------------------------- | ------------------- | ----------------------------------------------------------------------- | ------------------------------------- |
| `build-vllm`      | Build the VLLM container via the Dynamo repository and push to NGC.                                                  | `workflow_dispatch` | `dynamo_ref` (optional)                                                 | Image pushed to `nvcr.io`.            |
| `deploy-model`    | Render a deployment template under `deploy/`, substitute env variables, override replicas, and apply with `kubectl`. | `workflow_dispatch` | `deployment_type`, `main_container_image`, `model_name`, replica counts | Applied manifest + uploaded artifact. |
| `deploy-operator` | Install / upgrade the Dynamo platform (CRDs + operator) via Helm.                                                    | `workflow_dispatch` | `dynamo_version` (optional)                                             | Operator installed in target cluster. |

---

## Workflow details

### `build-vllm`

Builds and optionally publishes a container using Dynamo's `./container/build.sh` helper.

- Clones `ai-dynamo/dynamo`, checks out the chosen ref (defaults to `v0.5.0`).
- Logs into `nvcr.io` with the provided credentials.
- Runs `./container/build.sh --framework VLLM --platform linux/arm64`.

#### Inputs (build-vllm)

- `dynamo_ref` (optional) – Tag or branch from the upstream repository; defaults to `v0.5.0`.

### `deploy-model`

Applies one of the templates under `deploy/` to a Kubernetes cluster with your chosen image, model, and replica counts.

Key capabilities:

- `envsubst` replaces `MAIN_CONTAINER_IMAGE`, `MODEL`, and replica placeholders in the selected template.
- A Python post-processing step coerces replica counts to integers to satisfy the CRD schema.
- The rendered manifest is saved as an artifact for later inspection / rollback.

#### Inputs (deploy-model)

- `deployment_type` – Template name (`agg`, `agg_router`, `disagg`, `disagg_router`, `disagg_planner`).
- `main_container_image` – Image to use for all `mainContainer` entries.
- `model_name` – Propagated to `python3 -m dynamo.vllm` calls in the template.
- `frontend_replicas`, `decode_replicas`, `prefill_replicas`, `planner_replicas`, `prometheus_replicas` – Integer replica overrides when the service exists in the chosen YAML.

#### Prerequisites (deploy-model)

- Self-hosted runner with `kubectl`, `envsubst`, and `python3` available.
- Target cluster must already contain the `hf-token-secret` referenced by the templates (or adjust the manifest before applying).

### `deploy-operator`

Installs/updates the Dynamo CRDs and platform Helm charts.

- Verifies `kubectl`, `helm`, and `docker` are present and prints their versions before proceeding.
- Resolves the release version from the `dynamo_version` input; if blank, fetches the latest GitHub release tag automatically.
- Installs CRDs into the `default` namespace and the platform chart into `dynamo-kubernetes` (creating it if needed).
- Creates/updates the `hf-token-secret` using the provided Hugging Face token.

#### Inputs (deploy-operator)

- `dynamo_version` (optional) – Specific release tag to install. Leave empty to auto-detect the latest upstream release.

#### Required environment / secrets (deploy-operator)

- `HF_TOKEN` – Populated on the runner (for example via an Actions secret) so the workflow can create the Kubernetes secret.

---

## Deployment templates (`deploy/`)

The YAML files under `deploy/` support the following environment variables, making it easy to re-use them across environments:

| Placeholder              | Description                                                     | Defaults (commented in templates)             |
| ------------------------ | --------------------------------------------------------------- | --------------------------------------------- |
| `$MAIN_CONTAINER_IMAGE`  | Runtime container image applied to all `mainContainer` entries. | `nvcr.io/nvidia/ai-dynamo/vllm-runtime:0.5.0` |
| `$MODEL`                 | Model identifier passed to `python3 -m dynamo.vllm`.            | `Qwen/Qwen3-0.6B`                             |
| `${FRONTEND_REPLICAS}`   | Replica count for `Frontend` services.                          | `1`                                           |
| `${DECODE_REPLICAS}`     | Replica count for `VllmDecodeWorker` services.                  | `1` (or `2` in planner/router templates)      |
| `${PREFILL_REPLICAS}`    | Replica count for `VllmPrefillWorker` services.                 | `1` (or `2` in planner template)              |
| `${PLANNER_REPLICAS}`    | Replica count for `Planner` service (planner template only).    | `1`                                           |
| `${PROMETHEUS_REPLICAS}` | Replica count for `Prometheus` service (planner template only). | `1`                                           |

All placeholders are resolved via `envsubst` in the `deploy-model` workflow, and the workflow will coerce provided values into integers before applying the manifest.

---

## Runner expectations

- Self-hosted runners require NVIDIA drivers (`nvidia-smi`), Docker, kubectl, helm, and GNU coreutils (for `envsubst`).
- Ensure the runner has network access to `nvcr.io`, `helm.ngc.nvidia.com`, and any target Kubernetes API server.
- Provide `KUBECONFIG` or use in-cluster configuration so `kubectl` commands succeed.

---

## Secrets summary

| Secret / variable | Used by           | Purpose                                        |
| ----------------- | ----------------- | ---------------------------------------------- |
| `HF_TOKEN`        | `deploy-operator` | Populate the `hf-token-secret` in the cluster. |
