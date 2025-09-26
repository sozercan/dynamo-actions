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

### Additional SGLang-Specific Inputs / Placeholders

The SGLang templates (and command builder step) also recognize these inputs (values are passed through as environment variables prior to `envsubst`):

| Input / Placeholder                                                | Purpose / Effect                                   | Typical Default | Example (DeepSeek) |
| ------------------------------------------------------------------ | -------------------------------------------------- | --------------- | ------------------ |
| `page_size` / `${PAGE_SIZE}`                                       | Token page size passed to SGLang                   | 16              | 16                 |
| `tp_size` / `${TP_SIZE}`                                           | Tensor parallel size (`--tp`)                      | 1               | 8                  |
| `dp_size` / `${DP_SIZE}`                                           | Data parallel size (`--dp`, omitted when 1)        | 1               | 8                  |
| `ep_size` / `${EP_SIZE}`                                           | Expert parallel size (`--ep-size`, omitted when 1) | 1               | 8                  |
| `decode_gpus` / `${DECODE_GPUS}`                                   | GPUs per decode worker pod (resource limit)        | 1               | 8                  |
| `prefill_gpus` / `${PREFILL_GPUS}`                                 | GPUs per prefill worker pod                        | 1               | 8                  |
| `enable_dp_attention` / `${ENABLE_DP_ATTENTION}`                   | Adds `--enable-dp-attention` when true             | false           | true               |
| `trust_remote_code` / `${TRUST_REMOTE_CODE}`                       | Adds `--trust-remote-code` when true               | true            | true               |
| `skip_tokenizer_init` / `${SKIP_TOKENIZER_INIT}`                   | Adds `--skip-tokenizer-init` when true             | true            | true               |
| `disagg_transfer_backend` / `${DISAGG_TRANSFER_BACKEND}`           | Backend for disaggregation transfer                | nixl            | nixl               |
| `disagg_bootstrap_port` / `${DISAGG_BOOTSTRAP_PORT}`               | Port used for bootstrap in disagg mode             | 30001           | 30001              |
| `mem_fraction_static` / `${MEM_FRACTION_STATIC}`                   | Adds `--mem-fraction-static <value>` if non-empty  | (empty)         | 0.82               |
| `multinode_decode_node_count` / `${MULTINODE_DECODE_NODE_COUNT}`   | nodeCount for decode (multinode template)          | 1               | 1 (or >1)          |
| `multinode_prefill_node_count` / `${MULTINODE_PREFILL_NODE_COUNT}` | nodeCount for prefill (multinode template)         | 1               | 1 (or >1)          |
| `worker_command_extra` / `$WORKER_COMMAND_EXTRA`                   | Appended verbatim to decode/prefill commands       | (empty)         | (optional)         |
| `frontend_command_extra` / `$FRONTEND_COMMAND_EXTRA`               | Extra flags for frontend (aggregate variants)      | (empty)         | (optional)         |
| `planner_command_extra` / `$PLANNER_COMMAND_EXTRA`                 | Extra flags for planner (planner template)         | (empty)         | (optional)         |
| `prometheus_command_extra` / `$PROMETHEUS_COMMAND_EXTRA`           | Extra flags for prometheus (planner template)      | (empty)         | (optional)         |

The workflow dynamically assembles role-specific commands (Frontend / decode / prefill / planner / prometheus) and injects them as environment variables consumed by the templates.

All placeholders are resolved via `envsubst` in the `deploy-model` workflow, and the workflow will coerce provided values into integers before applying the manifest.

---

## Simple Qwen (SGLang, Aggregate) Example

Minimal aggregate deployment using the `agg` template (single frontend + single decode worker) and the Qwen 0.6B model.

| Input                | Value                               |
| -------------------- | ----------------------------------- |
| runtime              | `sglang`                            |
| deployment_type      | `agg`                               |
| main_container_image | `my-registry/sglang-runtime:latest` |
| model_name           | `Qwen/Qwen3-0.6B`                   |
| frontend_replicas    | `1`                                 |
| decode_replicas      | `1`                                 |
| prefill_replicas     | `1` (unused in agg; safe default)   |
| planner_replicas     | `1` (unused; required input)        |
| prometheus_replicas  | `1` (unused; required input)        |
| page_size            | `16` (default)                      |
| tp_size              | `1` (default)                       |
| dp_size              | `1` (default)                       |
| ep_size              | `1` (default)                       |
| decode_gpus          | `1` (default)                       |
| prefill_gpus         | `1` (default)                       |
| trust_remote_code    | `true` (default)                    |
| skip_tokenizer_init  | `true` (default)                    |

### Prefilled Dispatch URL (Qwen Aggregate)

```text
https://github.com/sozercan/dynamo-actions/actions/workflows/deploy-model.yaml/dispatches/new?ref=main&inputs[runtime]=sglang&inputs[deployment_type]=agg&inputs[main_container_image]=my-registry/sglang-runtime:latest&inputs[model_name]=Qwen/Qwen3-0.6B&inputs[frontend_replicas]=1&inputs[decode_replicas]=1&inputs[prefill_replicas]=1&inputs[planner_replicas]=1&inputs[prometheus_replicas]=1
```

### GitHub CLI Invocation (Qwen Aggregate)

```bash
gh workflow run deploy-model.yaml \
  --ref main \
  -f runtime=sglang \
  -f deployment_type=agg \
  -f main_container_image=my-registry/sglang-runtime:latest \
  -f model_name=Qwen/Qwen3-0.6B \
  -f frontend_replicas=1 \
  -f decode_replicas=1 \
  -f prefill_replicas=1 \
  -f planner_replicas=1 \
  -f prometheus_replicas=1
```

---

## DeepSeek (SGLang, Disaggregated) Example

Below is a reference configuration approximating the DeepSeek R1 disaggregated setup (single pod per role with 8 GPUs each, 8-way TP/DP/EP). Adjust the image to your built SGLang runtime.

| Input                        | Value                                          |
| ---------------------------- | ---------------------------------------------- |
| runtime                      | `sglang`                                       |
| deployment_type              | `disagg`                                       |
| main_container_image         | `my-registry/sglang-wideep-runtime:latest`     |
| model_name                   | `deepseek-ai/DeepSeek-R1`                      |
| frontend_replicas            | `1`                                            |
| decode_replicas              | `1`                                            |
| prefill_replicas             | `1`                                            |
| planner_replicas             | `1` (unused by plain disagg; safe default)     |
| prometheus_replicas          | `1` (unused by plain disagg; safe default)     |
| page_size                    | `16`                                           |
| tp_size                      | `8`                                            |
| dp_size                      | `8`                                            |
| ep_size                      | `8`                                            |
| decode_gpus                  | `8`                                            |
| prefill_gpus                 | `8`                                            |
| enable_dp_attention          | `true`                                         |
| trust_remote_code            | `true`                                         |
| skip_tokenizer_init          | `true`                                         |
| disagg_transfer_backend      | `nixl`                                         |
| disagg_bootstrap_port        | `30001`                                        |
| mem_fraction_static          | `0.82`                                         |
| multinode_decode_node_count  | `1`                                            |
| multinode_prefill_node_count | `1`                                            |
| worker_command_extra         | (leave blank unless adding experimental flags) |

### Prefilled Dispatch URL

Click (or copy) this URL to open the GitHub UI with fields pre-populated:

```text
https://github.com/sozercan/dynamo-actions/actions/workflows/deploy-model.yaml/dispatches/new?ref=main&inputs[runtime]=sglang&inputs[deployment_type]=disagg&inputs[main_container_image]=my-registry/sglang-wideep-runtime:latest&inputs[model_name]=deepseek-ai/DeepSeek-R1&inputs[frontend_replicas]=1&inputs[decode_replicas]=1&inputs[prefill_replicas]=1&inputs[planner_replicas]=1&inputs[prometheus_replicas]=1&inputs[page_size]=16&inputs[tp_size]=8&inputs[dp_size]=8&inputs[ep_size]=8&inputs[decode_gpus]=8&inputs[prefill_gpus]=8&inputs[enable_dp_attention]=true&inputs[trust_remote_code]=true&inputs[skip_tokenizer_init]=true&inputs[disagg_transfer_backend]=nixl&inputs[disagg_bootstrap_port]=30001&inputs[mem_fraction_static]=0.82&inputs[multinode_decode_node_count]=1&inputs[multinode_prefill_node_count]=1
```

If the image name or model include characters that break the query, URL-encode them (`:` → `%3A`, `/` → `%2F`).

### GitHub CLI Invocation

```bash
gh workflow run deploy-model.yaml \
  --ref main \
  -f runtime=sglang \
  -f deployment_type=disagg \
  -f main_container_image=my-registry/sglang-wideep-runtime:latest \
  -f model_name=deepseek-ai/DeepSeek-R1 \
  -f frontend_replicas=1 \
  -f decode_replicas=1 \
  -f prefill_replicas=1 \
  -f planner_replicas=1 \
  -f prometheus_replicas=1 \
  -f page_size=16 \
  -f tp_size=8 \
  -f dp_size=8 \
  -f ep_size=8 \
  -f decode_gpus=8 \
  -f prefill_gpus=8 \
  -f enable_dp_attention=true \
  -f trust_remote_code=true \
  -f skip_tokenizer_init=true \
  -f disagg_transfer_backend=nixl \
  -f disagg_bootstrap_port=30001 \
  -f mem_fraction_static=0.82 \
  -f multinode_decode_node_count=1 \
  -f multinode_prefill_node_count=1
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
