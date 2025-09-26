# Dynamo Actions

GitHub Actions workflows for building container images and deploying [NVIDIA Dynamo](https://github.com/ai-dynamo/dynamo) components to Kubernetes.

## Workflows overview

| Workflow          | Purpose                                                                                                              | Trigger             | Key inputs                                                              | Outputs                               |
| ----------------- | -------------------------------------------------------------------------------------------------------------------- | ------------------- | ----------------------------------------------------------------------- | ------------------------------------- |
| `build-vllm`      | Build the VLLM container via the Dynamo repository and push to NGC.                                                  | `workflow_dispatch` | `dynamo_ref` (optional)                                                 | Image pushed to `nvcr.io`.            |
| `deploy-model`    | Render a deployment template under `deploy/`, substitute env variables, override replicas, and apply with `kubectl`. | `workflow_dispatch` | `runtime`, `deployment_type`, `main_container_image`, `model_name`, JSON config inputs | Applied manifest + uploaded artifact. |
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

Applies one of the templates under `deploy/` to a Kubernetes cluster with your chosen runtime, image, model, and configuration. The workflow now uses **compact JSON inputs** (to stay within GitHub's 10 input limit) that are parsed into environment variables before rendering.

Key capabilities:

- `envsubst` replaces `MAIN_CONTAINER_IMAGE`, `MODEL`, replica placeholders and SGLang-specific values.
- A Python post-processing step coerces replica counts to integers to satisfy the CRD schema.
- The rendered manifest is saved as an artifact for later inspection / rollback.

#### Inputs (deploy-model)

Required:

- `runtime` – `vllm` or `sglang`.
- `deployment_type` – Template filename (without `.yaml`) under `deploy/<runtime>/`.
- `main_container_image` – Image used for all `mainContainer` entries.
- `model_name` – Model identifier passed to the runtime entrypoint.

JSON configuration inputs (all optional; each has defaults if omitted):

| Input name       | Applies to | JSON Keys (examples)                                                                                                                       | Purpose                            |
| ---------------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------- |
| `replicas`       | both       | `frontend`, `decode`, `prefill`, `planner`, `prometheus`                                                                                   | Override service replica counts.   |
| `sglang_scaling` | sglang     | `page_size`, `tp_size`, `dp_size`, `ep_size`, `decode_gpus`, `prefill_gpus`, `multinode_decode_node_count`, `multinode_prefill_node_count` | Parallelism & GPU sizing.          |
| `sglang_flags`   | sglang     | `enable_dp_attention`, `trust_remote_code`, `skip_tokenizer_init`, `mem_fraction_static`                                                   | Boolean / misc tuning flags.       |
| `sglang_disagg`  | sglang     | `disagg_transfer_backend`, `disagg_bootstrap_port`                                                                                         | Disaggregation transport settings. |
| `command_extras` | sglang     | `worker`, `frontend`, `planner`, `prometheus`                                                                                              | Append extra CLI flags per role.   |

##### Defaults

```jsonc
replicas:        { "frontend":1, "decode":1, "prefill":1, "planner":1, "prometheus":1 }
sglang_scaling:  { "page_size":16, "tp_size":1, "dp_size":1, "ep_size":1, "decode_gpus":1, "prefill_gpus":1, "multinode_decode_node_count":1, "multinode_prefill_node_count":1 }
sglang_flags:    { "enable_dp_attention":false, "trust_remote_code":true, "skip_tokenizer_init":true, "mem_fraction_static":"" }
sglang_disagg:   { "disagg_transfer_backend":"nixl", "disagg_bootstrap_port":30001 }
command_extras:  { "worker":"", "frontend":"", "planner":"", "prometheus":"" }
```

You can override only what you need; leaving an input blank (or omitting via CLI) keeps the defaults. For non‑SGLang runtimes the SGLang JSON inputs are ignored.

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

- `HF_TOKEN` – Expose this value on the runner (for example by mapping a repository secret to an environment variable) so the workflow can create the Kubernetes secret. If it is missing, the workflow logs a warning and skips the secret creation step.

---

## Deployment templates (`deploy/`)

Templates are now organized by runtime:

```text
deploy/
  vllm/      # vLLM runtime templates (agg, agg_router, disagg, disagg_router, disagg_planner)
  sglang/    # SGLang runtime templates (agg, agg_logging, agg_router, disagg, disagg_multinode, disagg_planner)
```

When you run the `deploy-model` workflow you must provide:

- `runtime` (currently `vllm` or `sglang`)
- `deployment_type` (template filename without extension in the chosen runtime folder)

The YAML files support the following common environment placeholders:

| Placeholder              | Description                                                     | Defaults (commented in templates)             |
| ------------------------ | --------------------------------------------------------------- | --------------------------------------------- |
| `$MAIN_CONTAINER_IMAGE`  | Runtime container image applied to all `mainContainer` entries. | `nvcr.io/nvidia/ai-dynamo/vllm-runtime:0.5.0` |
| `$MODEL`                 | Model identifier passed to `python3 -m dynamo.vllm`.            | `Qwen/Qwen3-0.6B`                             |
| `${FRONTEND_REPLICAS}`   | Replica count for `Frontend` services.                          | `1`                                           |
| `${DECODE_REPLICAS}`     | Replica count for `VllmDecodeWorker` services.                  | `1` (or `2` in planner/router templates)      |
| `${PREFILL_REPLICAS}`    | Replica count for `VllmPrefillWorker` services.                 | `1` (or `2` in planner template)              |
| `${PLANNER_REPLICAS}`    | Replica count for `Planner` service (planner template only).    | `1`                                           |
| `${PROMETHEUS_REPLICAS}` | Replica count for `Prometheus` service (planner template only). | `1`                                           |

### SGLang-Specific JSON Keys / Placeholders

The SGLang JSON inputs map 1:1 to environment variables consumed by templates and the command builder step:

| JSON Input       | Key                            | Placeholder / Env                 | Notes                                  |
| ---------------- | ------------------------------ | --------------------------------- | -------------------------------------- |
| `sglang_scaling` | `page_size`                    | `${PAGE_SIZE}`                    | Token page size.                       |
|                  | `tp_size`                      | `${TP_SIZE}`                      | Tensor parallel.                       |
|                  | `dp_size`                      | `${DP_SIZE}`                      | Data parallel (omitted in CLI when 1). |
|                  | `ep_size`                      | `${EP_SIZE}`                      | Expert parallel (omitted when 1).      |
|                  | `decode_gpus`                  | `${DECODE_GPUS}`                  | GPUs per decode worker pod.            |
|                  | `prefill_gpus`                 | `${PREFILL_GPUS}`                 | GPUs per prefill worker pod.           |
|                  | `multinode_decode_node_count`  | `${MULTINODE_DECODE_NODE_COUNT}`  | Multi-node template only.              |
|                  | `multinode_prefill_node_count` | `${MULTINODE_PREFILL_NODE_COUNT}` | Multi-node template only.              |
| `sglang_flags`   | `enable_dp_attention`          | `${ENABLE_DP_ATTENTION}`          | Adds flag if true.                     |
|                  | `trust_remote_code`            | `${TRUST_REMOTE_CODE}`            | Adds flag if true.                     |
|                  | `skip_tokenizer_init`          | `${SKIP_TOKENIZER_INIT}`          | Adds flag if true.                     |
|                  | `mem_fraction_static`          | `${MEM_FRACTION_STATIC}`          | Adds flag when non-empty.              |
| `sglang_disagg`  | `disagg_transfer_backend`      | `${DISAGG_TRANSFER_BACKEND}`      | Disaggregation mode templates.         |
|                  | `disagg_bootstrap_port`        | `${DISAGG_BOOTSTRAP_PORT}`        | Port for bootstrap.                    |
| `command_extras` | `worker`                       | `$WORKER_COMMAND_EXTRA`           | Appended to decode/prefill commands.   |
|                  | `frontend`                     | `$FRONTEND_COMMAND_EXTRA`         | Frontend command extension.            |
|                  | `planner`                      | `$PLANNER_COMMAND_EXTRA`          | Planner template only.                 |
|                  | `prometheus`                   | `$PROMETHEUS_COMMAND_EXTRA`       | Planner template only.                 |

Replica environment variables originate from `replicas` JSON keys: `FRONTEND_REPLICAS`, `DECODE_REPLICAS`, `PREFILL_REPLICAS`, `PLANNER_REPLICAS`, `PROMETHEUS_REPLICAS`.

All placeholders are resolved via `envsubst` and replicas are integer-normalized post-render.

---

## Simple Qwen (SGLang, Aggregate) Example

Minimal aggregate deployment (single frontend + single decode worker) with defaults. Because all defaults already match what we need, only the required inputs are necessary.

### Prefilled Dispatch URL (Defaults Only)

```text
https://github.com/sozercan/dynamo-actions/actions/workflows/deploy-model.yaml/dispatches/new?ref=main&inputs[runtime]=sglang&inputs[deployment_type]=agg&inputs[main_container_image]=my-registry/sglang-runtime:latest&inputs[model_name]=Qwen/Qwen3-0.6B
```

### CLI (Explicit Replicas JSON – optional)

```bash
gh workflow run deploy-model.yaml \
  --ref main \
  -f runtime=sglang \
  -f deployment_type=agg \
  -f main_container_image=my-registry/sglang-runtime:latest \
  -f model_name=Qwen/Qwen3-0.6B \
  -f replicas='{"frontend":1,"decode":1,"prefill":1,"planner":1,"prometheus":1}'
```

---

## Simple Qwen (vLLM, Aggregate) Example

Defaults also suffice for vLLM aggregate; only required inputs are needed.

```bash
gh workflow run deploy-model.yaml \
  --ref main \
  -f runtime=vllm \
  -f deployment_type=agg \
  -f main_container_image=my-registry/vllm-runtime:latest \
  -f model_name=Qwen/Qwen3-0.6B
```

---

## DeepSeek (SGLang, Disaggregated) Example

Reference configuration approximating the DeepSeek R1 disaggregated setup (single pod per role with 8 GPUs each, 8‑way TP/DP/EP). Adjust the image as needed.

```bash
gh workflow run deploy-model.yaml \
  --ref main \
  -f runtime=sglang \
  -f deployment_type=disagg \
  -f main_container_image=my-registry/sglang-wideep-runtime:latest \
  -f model_name=deepseek-ai/DeepSeek-R1 \
  -f replicas='{"frontend":1,"decode":1,"prefill":1,"planner":1,"prometheus":1}' \
  -f sglang_scaling='{"page_size":16,"tp_size":8,"dp_size":8,"ep_size":8,"decode_gpus":8,"prefill_gpus":8,"multinode_decode_node_count":1,"multinode_prefill_node_count":1}' \
  -f sglang_flags='{"enable_dp_attention":true,"trust_remote_code":true,"skip_tokenizer_init":true,"mem_fraction_static":"0.82"}' \
  -f sglang_disagg='{"disagg_transfer_backend":"nixl","disagg_bootstrap_port":30001}'
```

URL form (JSON must be URL‑encoded, example shown partially encoded):

```text
https://github.com/sozercan/dynamo-actions/actions/workflows/deploy-model.yaml/dispatches/new?ref=main&inputs[runtime]=sglang&inputs[deployment_type]=disagg&inputs[main_container_image]=my-registry/sglang-wideep-runtime:latest&inputs[model_name]=deepseek-ai/DeepSeek-R1&inputs[sglang_scaling]=%7B%22page_size%22:16,%22tp_size%22:8,%22dp_size%22:8,%22ep_size%22:8,%22decode_gpus%22:8,%22prefill_gpus%22:8,%22multinode_decode_node_count%22:1,%22multinode_prefill_node_count%22:1%7D&inputs[sglang_flags]=%7B%22enable_dp_attention%22:true,%22trust_remote_code%22:true,%22skip_tokenizer_init%22:true,%22mem_fraction_static%22:%220.82%22%7D&inputs[sglang_disagg]=%7B%22disagg_transfer_backend%22:%22nixl%22,%22disagg_bootstrap_port%22:30001%7D
```

### Notes

- Increase `multinode_*_node_count` and reduce `*_gpus` if you want to distribute the total GPU count across multiple nodes instead of concentrating them in a single pod.
- Omit `mem_fraction_static` (leave blank) if unsure; the flag will not be added.
- For planner / prometheus roles use `deployment_type=disagg_planner` and supply meaningful `planner_replicas` / `prometheus_replicas`.
- Ensure the cluster has the required NVIDIA GPU resources and the `hf-token-secret` in the target namespace.

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
