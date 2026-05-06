# Verified Source Code References

All links verified against specific tagged releases. Last verified: May 2026.

---

## Linux Kernel Documentation

- **cgroup v1 cpuacct**: https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v1/cpuacct.html
  - Defines `cpuacct.usage` (nanoseconds), `cpuacct.stat` (user/system in jiffies)
- **cgroup v2 cpu interface**: https://docs.kernel.org/admin-guide/cgroup-v2.html#cpu-interface-files
  - Defines `cpu.stat` fields: `usage_usec`, `user_usec`, `system_usec` (microseconds)

---

## opencontainers/cgroups (v0.0.6)

The lowest-level library. Reads cgroup files and populates the CpuStats struct.

- **Stats struct** (CpuUsage fields): https://github.com/opencontainers/cgroups/blob/v0.0.6/stats.go#L22-L41
- **cgroup v1 CPU reading** (`cpuacct.usage` + `cpuacct.stat`): https://github.com/opencontainers/cgroups/blob/v0.0.6/fs/cpuacct.go#L37
- **cgroup v2 CPU reading** (`cpu.stat → usage_usec/user_usec/system_usec`): https://github.com/opencontainers/cgroups/blob/v0.0.6/fs2/cpu.go#L79-L101

Unit conversions:
- cgroup v1: jiffies → nanoseconds: `(ticks × 1_000_000_000) / clockTicks` (clockTicks = 100 on x86)
- cgroup v2: microseconds → nanoseconds: `× 1000`

---

## opencontainers/runc

runc's cgroup library (predecessor to opencontainers/cgroups, same logic):

- **cgroup v1 cpuacct reading**: https://github.com/opencontainers/runc/blob/main/libcontainer/cgroups/fs/cpuacct.go
  - `getCpuUsageBreakdown()` reads `cpuacct.stat` and parses `user`/`system` lines
  - Uses constant `systemField = "system"` — confirming `system` is the canonical name

---

## containerd/cgroups

Used by Moby to get container metrics via the containerd task API:

- **cgroup v1 cpuacct** (`cpuacct.usage` + `cpuacct.stat`): https://github.com/containerd/cgroups/blob/main/cgroup1/cpuacct.go
- **v1 protobuf** (`CPUStat.usage.total`, `.user`, `.system`): https://github.com/containerd/cgroups/blob/main/cgroup1/stats/v1.proto
- **v2 protobuf** (`CPUStat.usage_usec`, `.user_usec`, `.system_usec`): https://github.com/containerd/cgroups/blob/main/cgroup2/stats/v2.proto

---

## google/cAdvisor (v0.56.2)

- **Field copy from opencontainers/cgroups into cAdvisor struct**:
  https://github.com/google/cadvisor/blob/v0.56.2/container/libcontainer/handler.go#L769-L772
  ```go
  ret.Cpu.Usage.User   = s.CpuStats.CpuUsage.UsageInUsermode
  ret.Cpu.Usage.System = s.CpuStats.CpuUsage.UsageInKernelmode
  ret.Cpu.Usage.Total  = s.CpuStats.CpuUsage.TotalUsage
  ```
- **cAdvisor internal CPU struct** (`CpuUsage` with Total/User/System/PerCpu):
  https://github.com/google/cadvisor/blob/v0.56.2/info/v1/container.go#L303-L319
- **Prometheus exposition** (applies `÷ 1e9`, defines metric names):
  https://github.com/google/cadvisor/blob/v0.56.2/metrics/prometheus.go#L151-L188
  - `container_cpu_usage_seconds_total` ← `Cpu.Usage.Total ÷ 1e9`
  - `container_cpu_user_seconds_total` ← `Cpu.Usage.User ÷ 1e9`
  - `container_cpu_system_seconds_total` ← `Cpu.Usage.System ÷ 1e9`
  - `asNanosecondsToSeconds` helper: `float64(v) / 1e9`

---

## kubernetes/kubernetes (v1.36.0)

- **Only `Cpu.Usage.Total` is read by kubelet** (User and System are never touched):
  https://github.com/kubernetes/kubernetes/blob/v1.36.0/pkg/kubelet/stats/helper.go#L58
  ```go
  cpuStats.UsageCoreNanoSeconds = &cstat.Cpu.Usage.Total
  ```
- **cAdvisor stats provider** (pod cgroup read — NOT a sum of containers):
  https://github.com/kubernetes/kubernetes/blob/v1.36.0/pkg/kubelet/stats/cadvisor_stats_provider.go
- **CRI stats provider** (pod = sandbox + container sum):
  https://github.com/kubernetes/kubernetes/blob/v1.36.0/pkg/kubelet/stats/cri_stats_provider.go
- **System containers** (node stats — kubelet, runtime, pods cgroup root):
  https://github.com/kubernetes/kubernetes/blob/v1.36.0/pkg/kubelet/server/stats/summary_sys_containers.go
- **Summary handler** (`GET /stats/summary` → `ListPodStats`, `POST` → `ListPodStatsAndUpdateCPUNanoCoreUsage`):
  https://github.com/kubernetes/kubernetes/blob/v1.36.0/pkg/kubelet/server/stats/summary.go

---

## moby/moby (v28.5.2)

- **CPUUsage API struct** (field definitions with units):
  https://github.com/moby/moby/blob/0fef616242e9d34d4a62c0f13c489bd66bb499bd/api/types/container/stats.go#L17-L38
- **cgroup v2 stats population** (`UsageUsec × 1000`, `UserUsec × 1000`, `SystemUsec × 1000`):
  https://github.com/moby/moby/blob/v28.5.2/daemon/stats_unix.go#L184
- **Stats collection** (daemon entry point):
  https://github.com/moby/moby/blob/master/daemon/stats.go
- **`PercpuUsage` absent on cgroup v2** (added in PR #40657):
  https://github.com/moby/moby/pull/40657

---

## OTel Collector Contrib receivers

- **kubeletstatsreceiver CPU** (reads `UsageCoreNanoSeconds` → `container.cpu.time`):
  https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/v0.151.0/receiver/kubeletstatsreceiver/internal/kubelet/cpu.go#L51-L57
- **kubeletstatsreceiver metric definition**:
  https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/v0.151.0/receiver/kubeletstatsreceiver/internal/metadata/generated_metrics.go#L290-L297
- **dockerstatsreceiver** (uses Moby CPUUsage struct):
  https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/v0.151.0/receiver/dockerstatsreceiver/receiver.go#L292

---

## OTel Semantic Conventions

- **`container.cpu.time` metric**: https://opentelemetry.io/docs/specs/semconv/system/container-metrics/#metric-containercputime
- **`cpu.mode` attribute registry**: https://opentelemetry.io/docs/specs/semconv/registry/attributes/cpu/
- **Active discussion**: https://github.com/open-telemetry/semantic-conventions/issues/2418

---

## Empirical Verification

Verified on a cgroup v2 host (Docker Engine, macOS Docker Desktop), 100 consecutive samples:

```
Exact match (user+kernel == total): 55/100 samples
Mismatch (delta = +1,000 ns):       45/100 samples
Max delta observed:                  +1,000 ns (1 µs)
Min delta observed:                  0 ns
```

Delta is always exactly +1,000 ns when non-zero, consistent with a 1 µs rounding
artefact in the kernel's `cpu.stat` accounting where `usage_usec` rounds up 1 µs
relative to `user_usec + system_usec`.
