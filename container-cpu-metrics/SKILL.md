---
name: container-cpu-metrics
description: >
  Cross-technology semantic layer for container CPU metrics. Use this skill whenever working
  with container CPU time, CPU usage, CPU utilization, or CPU mode metrics — regardless of
  the collection source. Covers Docker Stats API (Moby), Kubelet /stats/summary, cAdvisor
  Prometheus, OpenTelemetry Semantic Conventions (container.cpu.time, cpu.mode), cgroups
  v1/v2 kernel sources, and receiver implementations (dockerstats, kubeletstats). Use when
  answering questions about: what container.cpu.time measures, whether user+system equals
  total, what cpu.mode values are valid, how to map between APIs, how cgroup version affects
  CPU metric semantics, how to derive cpu.usage or cpu.utilization from cpu.time, how to
  convert between cpu.time/usage/utilization, or how OTel semconv should model container CPU
  time. Also use when debugging discrepancies between CPU metrics from different collectors,
  when designing metric pipelines that combine Kubernetes and Docker sources, or when
  computing rate/utilization queries from cumulative CPU counters.
---

# Container CPU Metrics — Cross-Technology Semantic Layer

This skill captures the full verified lineage of container CPU time metrics, from the
Linux kernel through every major collection path, to OTel Semantic Conventions. It is the
authoritative reference for reasoning about `container.cpu.time` and `cpu.mode` regardless
of how the data is collected.

---

## Quick Decision Tree

**"Which metric maps to what?"** → See [Source Mapping Table](#source-mapping-table)  
**"Does user + system = total?"** → See [The Inequality](#the-inequality-user--system--total)  
**"What cpu.mode values are valid?"** → See [cpu.mode Semantics](#cpumode-semantics)  
**"How does cgroup version affect things?"** → See [cgroup v1 vs v2](#cgroup-v1-vs-v2-differences)  
**"What does Kubelet expose?"** → See [Kubelet Path](#kubelet--statssummary-path)  
**"How to derive usage or utilization from cpu.time?"** → See [Deriving Usage and Utilization](#deriving-usage-and-utilization-from-cputime)  
**"How to model this in OTel semconv?"** → See [Semconv Design](#otel-semconv-design-guidance)

---

## Kernel Sources (Ground Truth)

Everything originates from two Linux cgroup pseudo-files. This is the ground truth that
all higher-level APIs derive from.

### cgroup v1

| Kernel file | Field | Unit | What it measures |
|---|---|---|---|
| `cpuacct.usage` | total CPU time | nanoseconds (high-res) | All CPU time consumed by cgroup, all cores summed |
| `cpuacct.usage_percpu` | per-CPU breakdown | nanoseconds | Same as above, split by CPU core |
| `cpuacct.stat → user` | user-space CPU time | jiffies (1/100s) → converted to ns | Time processes spent executing user-space code |
| `cpuacct.stat → system` | kernel-space CPU time | jiffies → converted to ns | Time kernel spent executing syscalls on behalf of processes |

**Critical**: `cpuacct.usage` and `cpuacct.stat` are **independent accounting mechanisms**
with different resolutions. `total ≠ user + system` on cgroup v1 because of this.

Kernel docs: https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v1/cpuacct.html

### cgroup v2

| Kernel file | Field | Unit | What it measures |
|---|---|---|---|
| `cpu.stat → usage_usec` | total CPU time | microseconds | All CPU time consumed |
| `cpu.stat → user_usec` | user-space CPU time | microseconds | User-space execution time |
| `cpu.stat → system_usec` | kernel-space CPU time | microseconds | Kernel-mode execution time |

**Critical**: All three come from the same `cpu.stat` file. The kernel guarantees
`usage_usec ≥ user_usec + system_usec` but occasional 1 µs rounding means
`usage_usec` can be 1 µs higher than `user_usec + system_usec`.

Kernel docs: https://docs.kernel.org/admin-guide/cgroup-v2.html#cpu-interface-files

---

## The Inequality: user + system ≠ total

This is **universally true** regardless of cgroup version, verified empirically:

| cgroup version | Max discrepancy | Cause |
|---|---|---|
| **v1** | ~10,000,000 ns (10 ms) | Different kernel files: `cpuacct.usage` (ns) vs `cpuacct.stat` (jiffies, 10ms resolution) |
| **v2** | ~1,000 ns (1 µs) | Same file, but microsecond-level rounding: `usage_usec` occasionally rounds up 1 µs vs `user_usec + system_usec` |

**Empirical evidence (cgroup v2, 100 samples):** 45/100 samples had `total = user + kernel + 1000ns`.
The delta is always exactly +1,000 ns (1 µs), never negative, never larger.

**Semconv implication**: Never define `container.cpu.time` semantics that imply additivity
between the total and the user/system breakdown. They are not additive and come from
different kernel accounting paths.

---

## Source Mapping Table

The definitive cross-API field mapping, verified against source code.

| Concept | Kernel source | opencontainers/cgroups | cAdvisor internal | cAdvisor Prometheus | Docker/Moby API | Kubelet /stats/summary | OTel `container.cpu.time` |
|---|---|---|---|---|---|---|---|
| **Total CPU time** | `cpuacct.usage` / `cpu.stat→usage_usec` | `CpuUsage.TotalUsage` | `Cpu.Usage.Total` | `container_cpu_usage_seconds_total` (÷1e9) | `cpu_usage.total_usage` | `usageCoreNanoSeconds` | `container.cpu.time{}` (no mode) |
| **User-space time** | `cpuacct.stat→user` / `cpu.stat→user_usec` | `CpuUsage.UsageInUsermode` | `Cpu.Usage.User` | `container_cpu_user_seconds_total` (÷1e9) | `cpu_usage.usage_in_usermode` | ❌ **not exposed** | `container.cpu.time{cpu.mode=user}` |
| **Kernel-space time** | `cpuacct.stat→system` / `cpu.stat→system_usec` | `CpuUsage.UsageInKernelmode` | `Cpu.Usage.System` | `container_cpu_system_seconds_total` (÷1e9) | `cpu_usage.usage_in_kernelmode` | ❌ **not exposed** | `container.cpu.time{cpu.mode=system}` |
| **Per-CPU time** | `cpuacct.usage_percpu` | `CpuUsage.PercpuUsage` | `Cpu.Usage.PerCpu` | `container_cpu_usage_seconds_total{cpu="cpuN"}` | `cpu_usage.percpu_usage[]` | ❌ not exposed | not modeled |
| **Host system CPU** | `/proc/stat` cpu line | n/a | n/a | ❌ not per-container | `system_cpu_usage` | ❌ not exposed | not applicable |

---

## cpu.mode Semantics

### Valid values

| `cpu.mode` value | Meaning | Available from |
|---|---|---|
| *(absent)* | Total CPU time (`cpuacct.usage` / `cpu.stat→usage_usec`) | All sources |
| `user` | User-space CPU time | Docker/Moby, cAdvisor only |
| `system` | Kernel-space CPU time | Docker/Moby, cAdvisor only |

### `kernel` is NOT a valid value

`kernel` was a legacy alias for `system`. It should be removed from semconv because:
- The Linux kernel itself names this field `system` in `cpuacct.stat` and `cpu.stat`
- runc uses `systemField = "system"` as the constant name
- cAdvisor exposes `container_cpu_system_seconds_total` (not `kernel`)
- The existing `system.cpu.time` OTel metric uses `system` (not `kernel`)
- Docker/Moby's field is named `UsageInKernelmode` but maps to `cpuacct.stat→system`

The name `kernel` in `UsageInKernelmode` is a historical naming artefact in Moby, not
a semantic distinction. `system` is canonical at every other layer.

---

## Collection Path Details

### opencontainers/cgroups library

The lowest-level Go library that reads cgroup files directly.

- **cgroup v1**: [`fs/cpuacct.go`](https://github.com/opencontainers/cgroups/blob/v0.0.6/fs/cpuacct.go#L37) — reads `cpuacct.usage`, `cpuacct.usage_percpu`, `cpuacct.stat`
- **cgroup v2**: [`fs2/cpu.go#L79-L101`](https://github.com/opencontainers/cgroups/blob/v0.0.6/fs2/cpu.go#L79-L101) — reads `cpu.stat`, multiplies usec fields × 1000 to get nanoseconds
- **Stats struct**: [`stats.go#L22-L41`](https://github.com/opencontainers/cgroups/blob/v0.0.6/stats.go#L22-L41)

**Unit conversion on cgroup v1**: jiffies → nanoseconds via `(ticks × 1e9) / clockTicks`  
**Unit conversion on cgroup v2**: microseconds → nanoseconds via `× 1000`

### cAdvisor path

cAdvisor copies from the opencontainers/cgroups struct into its own internal struct, then
exposes via Prometheus and the Kubelet stats API.

- **Field copy**: [`container/libcontainer/handler.go#L769-L772`](https://github.com/google/cadvisor/blob/v0.56.2/container/libcontainer/handler.go#L769-L772)
  ```go
  ret.Cpu.Usage.User   = s.CpuStats.CpuUsage.UsageInUsermode
  ret.Cpu.Usage.System = s.CpuStats.CpuUsage.UsageInKernelmode
  ret.Cpu.Usage.Total  = s.CpuStats.CpuUsage.TotalUsage
  ```
- **Internal struct**: [`info/v1/container.go#L303-L319`](https://github.com/google/cadvisor/blob/v0.56.2/info/v1/container.go#L303-L319)
- **Prometheus exposition**: [`metrics/prometheus.go#L151-L188`](https://github.com/google/cadvisor/blob/v0.56.2/metrics/prometheus.go#L151-L188) — applies `÷ 1e9` conversion

### Kubelet `/stats/summary` path

Kubelet reads **only `Cpu.Usage.Total`** from cAdvisor. `User` and `System` are never read.

- **Helper**: [`pkg/kubelet/stats/helper.go#L58`](https://github.com/kubernetes/kubernetes/blob/v1.36.0/pkg/kubelet/stats/helper.go#L58)
  ```go
  cpuStats.UsageCoreNanoSeconds = &cstat.Cpu.Usage.Total  // only Total is used
  ```
- `usageNanoCores` (the instantaneous rate) is computed from `CpuInst.Usage.Total`, also Total only
- `UsageInUsermode` and `UsageInKernelmode` from cAdvisor are **never read by kubelet**

**Consequence**: Any receiver using the Kubelet Summary API (e.g., `kubeletstatsreceiver`)
can only produce `container.cpu.time{}` (no mode). It cannot produce `cpu.mode=user` or
`cpu.mode=system`.

### Docker/Moby Stats API path

Moby retrieves metrics from containerd via gRPC (`Task.Metrics()`), not by reading cgroup
files directly.

- **API struct**: [`api/types/container/stats.go#L17-L38`](https://github.com/moby/moby/blob/0fef616242e9d34d4a62c0f13c489bd66bb499bd/api/types/container/stats.go#L17-L38)
- **cgroup v1 population**: via `v1.Metrics` protobuf from containerd → [`containerd/cgroups cgroup1/cpuacct.go`](https://github.com/containerd/cgroups/blob/main/cgroup1/cpuacct.go)
- **cgroup v2 population**: [`daemon/stats_unix.go#L184`](https://github.com/moby/moby/blob/v28.5.2/daemon/stats_unix.go#L184) — reads `UsageUsec`, `UserUsec`, `SystemUsec` × 1000

**Available fields**:
- `cpu_usage.total_usage` → Total (nanoseconds)
- `cpu_usage.usage_in_usermode` → User (nanoseconds)
- `cpu_usage.usage_in_kernelmode` → System/kernel (nanoseconds, despite the name)
- `cpu_usage.percpu_usage[]` → Per-CPU (cgroup v1 only; absent on cgroup v2)
- `system_cpu_usage` → **Host-level** total CPU time from `/proc/stat` — not a container metric

**Latest release**: v28.5.2 (November 2025)

---

## cgroup v1 vs v2 Differences

| Aspect | cgroup v1 | cgroup v2 |
|---|---|---|
| Total source | `cpuacct.usage` (nanoseconds) | `cpu.stat → usage_usec × 1000` |
| User/system source | `cpuacct.stat` (jiffies → ns) | `cpu.stat → user_usec/system_usec × 1000` |
| user+system = total? | **No** — different files, 10ms jiffie resolution | **Almost** — same file, max 1 µs rounding error |
| Per-CPU data | Available (`cpuacct.usage_percpu`) | **Not available** in Moby API |
| Resolution of user/system | 10ms (jiffie-based) | 1 µs |
| Status | Deprecated (systemd v258, Sep 2025) | Default on all modern distros |

---

## Deriving Usage and Utilization from cpu.time

Per OTel semantic conventions, `*.cpu.time` is the **recommended** (canonical) metric.
`*.cpu.usage` and `*.cpu.utilization` are **opt-in** derived metrics. Backends and
dashboards should derive them from `cpu.time` rather than expecting collectors to emit
them directly.

Guidance: https://github.com/open-telemetry/semantic-conventions/blob/main/docs/non-normative/groups/system/cpu-metrics-guidelines.md

### Requirement levels

| Metric | Requirement level | Description |
|---|---|---|
| `*.cpu.time` | **recommended** | Cumulative CPU time in seconds — measured directly from the OS |
| `*.cpu.usage` | **opt-in** | Rate of CPU consumption in core-seconds — derived from cpu.time |
| `*.cpu.utilization` | **opt-in** | CPU usage normalized by limit, range [0, 1] — derived from cpu.time |
| `*.cpu.limit_utilization` | **opt-in** | CPU usage normalized by CPU limit |
| `*.cpu.request_utilization` | **opt-in** | CPU usage normalized by CPU request |

### cpu.time → cpu.usage

**Usage** is the rate of change of cpu.time over a window, measured in core-seconds:

```
rate(container.cpu.time[5m]) / (5 * 60)
```

In PromQL, `rate()` computes the per-second rate of a counter, so the division by
window size converts to core-seconds. The result tells you "how many CPU cores worth
of time this container consumed per second over the window."

### cpu.time → cpu.utilization

**Utilization** is usage normalized by the CPU limit, resulting in a [0, 1] range per core:

```
rate(container.cpu.time[5m]) / (5 * 60) / <cpu_limit>
```

For Kubernetes pods with CPU limits:

```
rate(k8s.pod.cpu.time[5m]) / (5 * 60) / k8s.pod.cpu.limit
```

This gives `k8s.pod.cpu.limit_utilization`. A value of 1.0 means the pod is using 100%
of its CPU limit.

### Utilization excluding idle states (system-level)

For system-level CPU, utilization as percentage of time in non-idle states:

```
sum(rate(system.cpu.time{cpu.mode!="idle"}[5m]) without (cpu.mode)) / (5 * 60)
```

### Total system utilization across all cores

```
avg(sum(rate(system.cpu.time{cpu.mode!="idle"}[5m])) by (cpu.logical_number)) / (5 * 60)
```

### Exception: k8s.*.cpu.usage

`k8s.*.cpu.usage` is a special case — the Kubelet Stats API provides `usageNanoCores`
directly as an instantaneous rate. This is the only `*.cpu.usage` metric that can be
collected directly rather than derived. However, it remains **opt-in** since it is
derived internally from `cpu.time` and is not available from other sources like the
Docker Stats API.

### When to derive vs collect

| Scenario | Approach |
|---|---|
| Backend/dashboard needs usage or utilization | Derive from `cpu.time` using rate queries |
| Kubelet provides `usageNanoCores` and collector wants to emit it | Collect directly as `k8s.*.cpu.usage` (opt-in) |
| Docker Stats API consumer needs utilization | Derive from `container.cpu.time` — Docker does not provide usage/utilization |
| Collector implementation | SHOULD emit `cpu.time` by default; SHOULD gate `cpu.usage` and `cpu.utilization` behind config |

---

## OTel Semconv Design Guidance

### Recommended metric model

```
container.cpu.time{}                     # total — always available from all sources
container.cpu.time{cpu.mode=user}        # user — only from Docker/Moby and cAdvisor
container.cpu.time{cpu.mode=system}      # system — only from Docker/Moby and cAdvisor
```

### Mixed-source scenario

On the same system, it is valid and expected to have:

```
# From kubeletstatsreceiver (total only — from cpuacct.usage via kubelet)
container.cpu.time{}

# From dockerstatsreceiver (all three — from cpu.stat or cpuacct.usage+stat via Moby)
container.cpu.time{}
container.cpu.time{cpu.mode=user}
container.cpu.time{cpu.mode=system}
```

These are distinct series in any time-series backend. Backends that enforce label
consistency per metric name will treat `{}` and `{cpu.mode=user}` as separate series,
which is correct.

### What the semconv notes MUST state

1. `cpu.mode` absent = total CPU time from `cpuacct.usage` / `cpu.stat→usage_usec`
2. `cpu.mode=user` / `cpu.mode=system` from `cpuacct.stat` / `cpu.stat→user_usec|system_usec`
3. **`user + system ≠ total`** — this is a kernel-level property:
   - On cgroup v1: discrepancy up to ~10 ms (different files, different accounting)
   - On cgroup v2: discrepancy up to ~1 µs (same file, microsecond rounding)
4. `cpu.mode=user` and `cpu.mode=system` are **conditionally available** — only when
   collecting from Docker/Moby or cAdvisor directly. Kubelet only exposes total.
5. `kernel` is NOT a valid `cpu.mode` value — use `system` instead.

### Prometheus compatibility

Emitting `container.cpu.time{}` and `container.cpu.time{cpu.mode=user}` from different
receivers is compatible with Prometheus. They are different label-set combinations and
produce distinct series. Users should not attempt to sum them or assume they are
mathematically related.

---

## Pod and Node Level (Kubernetes only)

Container-level CPU time **cannot** be aggregated to pod or node level by simple summation.
Each level is an independent cgroup read:

- **Pod `usageCoreNanoSeconds`** ≠ sum of container values
  - cAdvisor path: read directly from pod's own cgroup via `getCadvisorPodInfoFromPodUID()`
  - CRI path: includes sandbox/pause container CPU, which is absent from `pod.Containers[]`
- **Node `usageCoreNanoSeconds`** ≠ sum of pod values
  - Includes kubelet, container runtime, OS daemons, kernel threads
  - Read from machine-wide root cgroup, not derived from pod stats

Source: [`pkg/kubelet/stats/cadvisor_stats_provider.go`](https://github.com/kubernetes/kubernetes/blob/v1.36.0/pkg/kubelet/stats/cadvisor_stats_provider.go),
[`pkg/kubelet/server/stats/summary_sys_containers.go`](https://github.com/kubernetes/kubernetes/blob/v1.36.0/pkg/kubelet/server/stats/summary_sys_containers.go)

---

## Key Source Code References

For deeper investigation, load [`references/source-links.md`](references/source-links.md)
which contains the full annotated list of all verified source code links organized by layer.
