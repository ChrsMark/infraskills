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
