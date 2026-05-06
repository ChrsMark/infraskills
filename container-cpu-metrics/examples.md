# container-cpu-metrics — Skill Usage Examples

---

## Example 1: Docker Stats API

**User prompt:**
> Given the following from docker stats what is the otel compatible cpu time?
> `{ "cpu_stats": { "cpu_usage": { "total_usage": 428232076000, "usage_in_kernelmode": 125821286000, "usage_in_usermode": 302410790000 }, ... } }`

**Skill invoked:** `container-cpu-metrics/SKILL.md`

**Response:**

The skill identifies the three Docker Stats fields and maps them to `container.cpu.time` data points (÷ 1e9 for seconds):

| Metric | `cpu.mode` | Raw (ns) | OTel value (s) | Source field |
|---|---|---|---|---|
| `container.cpu.time` | *(absent — total)* | 428,232,076,000 | 428.232076 | `cpu_usage.total_usage` |
| `container.cpu.time` | `user` | 302,410,790,000 | 302.410790 | `cpu_usage.usage_in_usermode` |
| `container.cpu.time` | `system` | 125,821,286,000 | 125.821286 | `cpu_usage.usage_in_kernelmode` |

Key clarifications surfaced by the skill:
- `system_cpu_usage` is a host-level field from `/proc/stat` — not a container metric.
- The Docker field `usage_in_kernelmode` maps to `cpu.mode=system`, not `cpu.mode=kernel`.
- `user + system ≠ total` is expected behaviour, not a bug.

Full reference for the Docker Stats API response schema:
https://docs.docker.com/reference/api/engine/version/v1.45/#tag/Container/operation/ContainerStats

---

## Example 2: Kubelet Stats API

**User prompt:**
> Here is another sample from kubelet stats API — what is the otel compatible cpu time?
> `{ "containers": [ { "name": "kube-apiserver", "cpu": { "usageCoreNanoSeconds": 45900186687, "usageNanoCores": 46144879 } } ] }`

**Skill invoked:** `container-cpu-metrics/SKILL.md`

**Response:**

The skill identifies `usageCoreNanoSeconds` as the only field that maps to `container.cpu.time`, with no `cpu.mode` breakdown available from this source:

| Metric | `cpu.mode` | Source field |
|---|---|---|
| `container.cpu.time` | *(absent — total only)* | `usageCoreNanoSeconds` ÷ 1e9 |

Key clarifications surfaced by the skill:
- Kubelet forwards only `Cpu.Usage.Total` from cAdvisor — user and system mode breakdowns are never exposed.
- `usageNanoCores` is an instantaneous rate, not a cumulative counter — it does not map to `container.cpu.time`.
- Any receiver using the Kubelet Summary API (e.g., `kubeletstatsreceiver`) can only produce `container.cpu.time{}` with no `cpu.mode` attribute.

Full reference for the kubelet `/stats/summary` JSON structure:
https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/receiver/kubeletstatsreceiver/testdata/stats-summary.json

---

## Example 3: Deriving CPU Usage and Utilization from cpu.time

**User prompt:**
> I have `container.cpu.time` from the dockerstatsreceiver. How do I get CPU usage and utilization from it?

**Skill invoked:** `container-cpu-metrics/SKILL.md`

**Response:**

Per the OTel CPU metrics guidelines, `cpu.time` is the canonical metric. Usage and utilization are derived from it:

**CPU Usage** (core-seconds — "how many cores worth of CPU per second"):

```promql
rate(container.cpu.time[5m]) / (5 * 60)
```

**CPU Utilization** (normalized to [0, 1] — "fraction of CPU limit consumed"):

```promql
rate(container.cpu.time[5m]) / (5 * 60) / <cpu_limit>
```

For a container with a 2-core CPU limit and `container.cpu.time` increasing by 600 seconds over a 5-minute window:

| Step | Formula | Result |
|---|---|---|
| Rate of cpu.time | 600s / 300s | 2.0 core-seconds/s |
| Usage | same as rate | 2.0 (using 2 full cores) |
| Utilization | 2.0 / 2 (limit) | 1.0 (100% of limit) |

Key clarifications:
- Docker Stats API does not provide usage or utilization directly — always derive from `container.cpu.time`.
- `cpu.mode` breakdown (user/system) can also be rated individually: `rate(container.cpu.time{cpu.mode="user"}[5m])`.
- For Kubernetes pods: `rate(k8s.pod.cpu.time[5m]) / (5*60) / k8s.pod.cpu.limit` gives `k8s.pod.cpu.limit_utilization`.
- `usageNanoCores` from the Kubelet Stats API is the only `*.cpu.usage` that can be collected directly (opt-in).

OTel CPU metrics guidelines:
https://github.com/open-telemetry/semantic-conventions/blob/main/docs/non-normative/groups/system/cpu-metrics-guidelines.md
