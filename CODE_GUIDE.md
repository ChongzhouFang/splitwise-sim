# SplitwiseSim Code Guide

This guide summarizes the code layout and the main places to change when extending
SplitwiseSim. It is meant as a quick maintainer map: what each file does, how
requests flow through the simulator, where scheduling decisions live, and where
to find utilization and performance metrics.

## Request Flow

1. `run.py` loads Hydra configuration from `configs/config.yaml`.
2. `initialize.py` builds repositories, the cluster, router, arbiter,
   applications, trace, performance model, and power model.
3. `TraceSimulator` in `simulator.py` schedules one event per request arrival.
4. `router.py` forwards each request to the configured application's scheduler.
5. `scheduler.py` assigns request DAG nodes to instances and may insert KV-cache
   transfer flows.
6. `executor.py` submits tasks to `instance.py` and flows to `interconnect.py`.
7. `instance.py` runs prompt/token tasks using `performance_model.py`.
8. Completed requests are collected by the scheduler and router.
9. `TraceSimulator.save_results()` writes CSV outputs in the Hydra output
   directory, usually under `results/...`.

All simulation time values are in seconds.

## Top-Level Python Files

| File | Purpose |
| --- | --- |
| `run.py` | Main Hydra entry point. Seeds randomness, initializes the simulator through `initialize.py`, deletes old `oom.csv`, and runs the trace simulation. |
| `initialize.py` | Factory helpers for constructing traces, repositories, performance/power models, cluster, router, arbiter, applications, and start state. |
| `simulator.py` | Discrete-event simulator. Defines `Event`, `Simulator`, `TraceSimulator`, and global convenience functions such as `clock()` and `schedule_event()`. |
| `cluster.py` | Cluster object containing servers, interconnects, power budget, and cluster-level run/power plumbing. Builds servers from cluster YAML. |
| `server.py` | Server/SKU runtime object. Expands processor and interconnect configs, tracks instances and server power. |
| `processor.py` | CPU/GPU dataclasses. Tracks memory usage, power, server ownership, and writes `oom.csv` if processor memory is exceeded. |
| `interconnect.py` | Link model for communication flows. Includes PCIe, Ethernet, IB, NVLink, RDMA, and `DummyLink`; executes `Flow` objects with bandwidth-based duration. |
| `application.py` | Application endpoint. Owns model metadata, allocator, scheduler, instances, metrics, and SLO objects. |
| `allocator.py` | Allocator interface and `NoOpAllocator`. Spins up instances onto processors and reports per-instance utilization. |
| `scheduler.py` | Scheduling algorithms. Contains baseline schedulers, KV-transfer schedulers, and `MixedPoolScheduler`; this is the main file for routing requests to instances. |
| `executor.py` | Per-request execution coordinator. Submits request DAG nodes to instances/links and notifies the scheduler when requests complete. |
| `request.py` | Request and generative LLM request definitions. Builds the default prompt-to-token DAG and tracks request-level metrics. |
| `node.py` | Base DAG node and node state machine. Shared by tasks and flows. |
| `task.py` | Compute, prompt, and token task definitions. Handles per-task memory accounting, token progress, completion checks, and TTFT bookkeeping. |
| `flow.py` | Communication flow definitions, especially KV-cache transfer flows between prompt and token instances. |
| `trace.py` | Loads request traces from CSV and converts each row into a `Request`. |
| `metrics.py` | Dataclasses for node, task, flow, request, instance, application, router, arbiter, server metrics, and SLOs. |
| `performance_model.py` | Runtime predictors. Provides `ConstantPerformanceModel` and `DatabasePerformanceModel`, plus global wrappers used by instances. |
| `power_model.py` | Power model interface and constant/database implementations. Currently mostly unused except for idle server power accounting. |
| `model.py` | Model architecture, model size, model parallelism, and generative LLM dataclasses. |
| `model_repo.py` | Loads model architecture and size YAML files and instantiates model objects. |
| `hardware_repo.py` | Loads processor, interconnect, and SKU YAML files and instantiates hardware objects. |
| `orchestrator_repo.py` | Loads scheduler and allocator YAML files and instantiates orchestrators by name. |
| `router.py` | Router interface and `NoOpRouter`, which forwards requests to the target application scheduler. |
| `arbiter.py` | Arbiter interface and `NoOpArbiter`. Intended for future processor/server allocation across applications. |
| `start_state.py` | Creates initial application instances on cluster processors. Supports unallocated, uniform baseline/ORCA, and Splitwise prompt/token placement. |
| `generate_trace.py` | Utilities for generating synthetic traces and downloading Azure LLM trace distributions. |
| `utils.py` | Shared helpers for file logging, YAML loading, summary statistics, and CSV writing. |

## Notebook and Script Files

| Path | Purpose |
| --- | --- |
| `notebooks/utils.py` | Result-loading and plotting helpers used by notebooks. Reads `summary.csv`, `detailed/*.csv`, `request_nodes.csv`, and debug instance logs. |
| `notebooks/perf_model.py` | Notebook-side performance-model exploration code. |
| `notebooks/example.ipynb` | Example result-processing workflow. |
| `notebooks/plots.ipynb` | Plotting workflow for experiment outputs and Gantt-style inspection. |
| `scripts/run_baseline_h_example.sh` | Small baseline H100 run with explicit Hydra overrides. |
| `scripts/run_splitwise_ha_example.sh` | Small Splitwise HA example run. |
| `scripts/run_baseline_a.sh`, `scripts/run_baseline_h.sh` | Baseline A100/H100 experiment scripts. |
| `scripts/run_splitwise_aa.sh`, `scripts/run_splitwise_ha.sh`, `scripts/run_splitwise_hh.sh`, `scripts/run_splitwise_hhcap.sh` | Splitwise experiment scripts for different hardware mixes. |
| `scripts/run_isocost.sh`, `scripts/run_isopower.sh`, `scripts/run_costopt.sh` | Larger experiment sweep scripts. |
| `scripts/run_throughput*.sh`, `scripts/run_traces.sh` | Throughput and trace sweep scripts. |
| `sync_scripts/*.sh` | Helpers for collecting configs, repos, traces, and results across machines after distributed runs. |
| `clean.sh` | Cleanup helper. Inspect before using because cleanup scripts are usually destructive by design. |

## Configuration Layout

| Directory | Purpose |
| --- | --- |
| `configs/config.yaml` | Top-level Hydra config. Selects defaults, output directory template, seed, end time, debug flag, and launcher. |
| `configs/cluster/` | Cluster compositions: server SKU names, counts, interconnect entries, and power budget. |
| `configs/hardware_repo/processors/` | Processor definitions such as A100/H100 memory and power fields. |
| `configs/hardware_repo/skus/` | Server SKU definitions, for example DGX-A100 and DGX-H100 as 8 processors. |
| `configs/hardware_repo/interconnects/` | Link definitions such as NVLink. |
| `configs/model_repo/architectures/` | LLM architecture metadata: layers, hidden size, heads, etc. |
| `configs/model_repo/sizes/` | Model size/precision metadata. |
| `configs/orchestrator_repo/schedulers/` | Scheduler configs. Each YAML maps a scheduler name to a `_target_` class and constructor parameters. |
| `configs/orchestrator_repo/allocators/` | Allocator configs. Currently includes the no-op allocator. |
| `configs/start_state/` | Initial placement of application instances on processors, including baseline and Splitwise prompt/token placements. |
| `configs/applications/` | Application endpoint configs: model, scheduler, allocator, and application ID. |
| `configs/performance_model/` | Selects constant or database-backed performance model. |
| `configs/power_model/` | Selects the power model. |
| `configs/router/`, `configs/arbiter/` | Router and arbiter configs. Currently no-op implementations. |
| `configs/trace/` | Trace config, usually pointing to a CSV under `traces/`. |
| `configs/experiment/` | Hydra sweep definitions used by experiment scripts. |

## Changing the Scheduling Algorithm

Most scheduling changes belong in `scheduler.py`.

For a new non-Splitwise scheduler:

1. Add a subclass of `Scheduler`.
2. Implement `schedule(self, request, *args, **kwargs)`.
3. In `schedule`, get the prompt task from `request.root_node` and token task
   with `next(request.successors(prompt_task))`.
4. Assign every `Task` node to an `Instance` by setting `node.instance`.
5. If prompt and token run on the same instance without an inserted flow, set
   `prompt_task.chain = [token_task]` so the executor submits them back-to-back.
6. Update scheduler-side estimates such as `instance.sched_pending_tokens` or
   `instance.sched_memory` if the algorithm depends on predicted load.
7. Add a YAML file in `configs/orchestrator_repo/schedulers/` with `_target_:
   scheduler.YourScheduler`.
8. Select it with a Hydra override, for example
   `applications.0.scheduler=your_scheduler_name`.

For a new Splitwise/KV-cache scheduler:

1. Subclass `KVScheduler` or one of the existing KV schedulers.
2. Maintain separate prompt/token pools through instance tags or configured
   processor names.
3. Pick `prompt_instance` and `token_instance`.
4. Call `add_kv_cache_transfer(request, prompt_instance, token_instance,
   bandwidth)` when the prompt and token phases run on different instances.
5. Keep `sched_pending_tokens` and `sched_memory` consistent if the scheduler
   uses them for load decisions.

Useful existing examples:

| Scheduler | Behavior |
| --- | --- |
| `RandomScheduler` | Randomly chooses one instance for both phases. |
| `RoundRobinScheduler` | Cycles through all instances. |
| `JSQScheduler` | Chooses the instance with the fewest pending requests. |
| `TokenJSQScheduler` | Chooses the instance with the smallest pending token estimate. |
| `KVRoundRobinScheduler` | Round-robin prompt and token pools with KV transfer. |
| `KVJSQScheduler` | Queue-length JSQ separately for prompt and token pools. |
| `KVTokenJSQScheduler` | Pending-token JSQ separately for prompt and token pools. |
| `OverlapKV*` | Simulates KV-transfer overlap by increasing effective bandwidth. |
| `MixedPoolScheduler` | Allows prompt/token pools to borrow instances under memory/queue pressure. |

If the scheduling change requires different batching, preemption, or iteration
semantics, edit `instance.py` instead of only `scheduler.py`. `ORCAInstance`
implements iteration-level batching without preemption; `SplitwiseInstance`
adds preemption and max batch-token constraints.

## Obtaining Resource Utilization

The built-in utilization output is per instance:

- `Allocator.get_results()` in `allocator.py` computes
  `busy_time / interval_time` for every application instance.
- `TraceSimulator.save_results()` writes allocator results to
  `detailed/{application_id}_alloc.csv`.
- Columns include `instance_names` and `utilizations`.

Example output path:

```text
results/<seed>/<start_state>/<trace>/<cluster>/<model>/<scheduler>/detailed/0_alloc.csv
```

With `debug=True`, each instance also writes iteration-level logs:

```text
instances/<application_id>/<instance_id>.csv
```

These logs include iteration start/end, batch size, memory, pending queues,
batch tokens, and pending tokens. Use `notebooks/utils.py::get_instances_data()`
to load them for Gantt charts or utilization analysis.

For processor memory utilization, inspect or add logging around:

- `Processor.memory_used` and `Processor.memory_free` in `processor.py`.
- `Instance.memory`, `Instance.memory_allocs`, and `Instance.max_memory` in
  `instance.py`.

For network utilization, inspect or add logging around:

- `Link.bandwidth_used`, `Link.bandwidth_free`, and flow queues in
  `interconnect.py`.

Power accounting is only partially wired today. `Cluster.total_power`,
`Server.update_power()`, and `power_model.py` provide the current hooks, but
task-level dynamic power updates are marked unsupported.

## Getting Cluster Performance Metrics

Simulation outputs are written in the Hydra output directory selected by
`configs/config.yaml`:

```text
results/${seed}/${start_state.state_type}/${trace.filename}/${cluster.servers.0.count}_${cluster.servers.1.count}/${applications.0.model_architecture}/${applications.0.scheduler}
```

Important files:

| Output | Meaning |
| --- | --- |
| `summary.csv` | Application-level summary statistics over completed requests. |
| `detailed/{application_id}.csv` | Per-request metrics for completed requests. |
| `detailed/{application_id}_alloc.csv` | Per-instance utilization from the allocator. |
| `request_nodes.csv` | Per-request-node start/completion times, useful for Gantt charts. |
| `instances/{application_id}/{instance_id}.csv` | Iteration-level debug logs when `debug=True`. |
| `schedulers/{application_id}.csv` | Scheduler debug logs when application debug is enabled. |
| `simulator.csv`, `server.csv` | Simulator/server logs. |
| `oom.csv` | Written if a processor exceeds configured memory. |

`summary.csv` is derived from scheduler metrics in `Scheduler.get_results()`.
Current request metrics include:

- response time
- queue time
- TTFT
- time between tokens approximation, `tbt_times`
- nth-token overhead
- prompt size
- token size

The summary statistics are computed by `utils.get_statistics()` and include
mean, standard deviation, min, max, median, p50, p90, p95, p99, p999, and
geomean.

To add a new metric:

1. Add fields to the relevant dataclass in `metrics.py` if the metric needs
   persistent state.
2. Update request/task/flow/instance state transitions where the metric should
   be recorded.
3. Add the metric array in `Scheduler.get_results()` or allocator/router result
   methods.
4. `TraceSimulator.save_results()` will automatically summarize scheduler
   metrics returned from `get_results()`.

## Configuring Clusters

Cluster size and hardware mix are controlled by `configs/cluster/*.yaml`.
A cluster file contains:

```yaml
power_budget: 232000
servers:
  - sku: dgx-a100
    count: 1
  - sku: dgx-h100
    count: 1
interconnects:
  - link: infiniband
    topology: p2p
```

The `sku` names map to files in `configs/hardware_repo/skus/`. A SKU expands
into processors, for example DGX-H100 maps to 8 `h100-80gb` processors. Processor
definitions live in `configs/hardware_repo/processors/`.

To configure an existing cluster from the command line:

```bash
python run.py cluster=half_half cluster.servers.0.count=4 cluster.servers.1.count=4
```

To add a new cluster type:

1. Create a new YAML file under `configs/cluster/`.
2. Reference existing SKU names, or add new SKU files under
   `configs/hardware_repo/skus/`.
3. If adding a new SKU, add or reuse processor configs under
   `configs/hardware_repo/processors/`.
4. Run with `python run.py cluster=<new_cluster_file_stem>`.

Initial model placement is configured separately from cluster composition:

- `configs/start_state/baseline.yaml` and `orca.yaml` place one instance per
  server uniformly.
- `configs/start_state/splitwise.yaml` creates separate prompt and token
  instances.
- `split_type: homogeneous` uses counts such as `prompt.num_instances` and
  `token.num_instances`.
- `split_type: heterogeneous` uses `prompt.instance_names` and
  `token.instance_names` to map prompt/token roles onto SKU names.

The application model, allocator, and scheduler are selected in
`configs/applications/solo.yaml` or overridden from the command line:

```bash
python run.py \
  applications.0.model_architecture=llama2-70b \
  applications.0.model_size=llama2-70b-fp16 \
  applications.0.scheduler=token_jsq \
  start_state=baseline \
  performance_model=db
```

## Configuring Performance and Power Models

Performance model selection is controlled by `configs/performance_model/`:

- `constant.yaml` instantiates `ConstantPerformanceModel`.
- `db.yaml` instantiates `DatabasePerformanceModel` using `data/perf_model.csv`.

`DatabasePerformanceModel` expects profiling rows with:

```text
model, hardware, tensor_parallel, prompt_size, batch_size, token_size, prompt_time, token_time
```

Times in the CSV are converted from milliseconds to seconds. If an exact row is
missing, the model interpolates by total batch tokens for a given
`(model, hardware, tensor_parallel)` key.

Power model selection is controlled by `configs/power_model/`. Dynamic power is
not fully supported yet, but the existing hooks are:

- `power_model.py` for processor and idle server power estimates.
- `server.py::update_power()`.
- `cluster.py::power`, `total_power`, and `power_budget`.

## Common Extension Points

| Goal | Files to change |
| --- | --- |
| New scheduling policy | `scheduler.py`, plus a YAML file under `configs/orchestrator_repo/schedulers/`. |
| New batching or preemption behavior | `instance.py`, and possibly scheduler bookkeeping in `scheduler.py`. |
| New autoscaling policy | `allocator.py`, `arbiter.py`, start-state/config YAML if needed. |
| New routing policy across applications | `router.py`, `configs/router/`, and application configs. |
| New request type or DAG shape | `request.py`, `task.py`, `flow.py`, `executor.py`, and metrics as needed. |
| New hardware type | `processor.py` or `interconnect.py`, plus hardware repo YAML. |
| New model | `configs/model_repo/architectures/`, `configs/model_repo/sizes/`, and possibly `data/perf_model.csv`. |
| New performance predictor | `performance_model.py` and `configs/performance_model/`. |
| New output metric | `metrics.py`, state-transition code, and `Scheduler.get_results()` or `Allocator.get_results()`. |

## Quick Commands

Run the default simulation:

```bash
python run.py
```

Run the small H100 baseline example:

```bash
./scripts/run_baseline_h_example.sh
```

Run with debug logs:

```bash
python run.py debug=True
```

Switch scheduler and performance model:

```bash
python run.py applications.0.scheduler=token_jsq performance_model=db
```

Run a cluster-size override:

```bash
python run.py cluster=half_half cluster.servers.0.count=2 cluster.servers.1.count=2
```
