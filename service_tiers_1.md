# SQL Server question — Service Tiers 1

## Statement

NorthPier Logistics is consolidating four workloads onto Azure SQL Database, on the logical server `northpier-sql.database.windows.net`. The company applies **Azure Hybrid Benefit** and **reserved capacity** wherever a tier allows it, so every database must be created in the **vCore purchasing model**. Each database has its own requirements:

| Database | Workload and requirements |
|---|---|
| `PierDispatch` | Truck-dispatch OLTP, 900 GB, thousands of small writes per second. Storage I/O latency must be **consistently below 2 ms**. The dispatch reports must run on a **read-only replica that is included in the price — no additional compute charge** for it. Zone-redundant high availability is required. |
| `PierArchive` | Shipment history, **20 TB today**, growing about 1 TB a year, read-mostly. Analysts need **at least two read-only replicas that can be sized independently of the primary** and added or removed without touching it. The company wants to pay only for the storage actually allocated, not for a configured maximum. |
| `PierSandbox` | 40 GB development copy used by contractors for a few hours on weekdays, idle at night and on weekends. **Compute must cost nothing while the database is idle**, the database must come back automatically on the first connection, and a warm-up of about a minute after idle periods is acceptable. |
| `PierBilling` | Invoicing, 250 GB, steady and predictable load 24x7 with occasional peaks. Latency of 5–10 ms is acceptable. The database uses **change data capture** for a downstream feed. The finance team wants the **lowest predictable monthly cost** and no read replica. |

Which combination of service tier and compute tier meets every requirement?

### a.

| Database | Service tier | Compute tier |
|---|---|---|
| `PierDispatch` | Business Critical, zone-redundant, 8 vCores | Provisioned |
| `PierArchive` | Hyperscale, 8 vCores, plus two **named replicas** (4 and 16 vCores) | Provisioned |
| `PierSandbox` | General Purpose, 0.5–4 vCores, auto-pause delay 60 minutes | Serverless |
| `PierBilling` | General Purpose, 4 vCores | Provisioned |

### b.

| Database | Service tier | Compute tier |
|---|---|---|
| `PierDispatch` | Premium P4 (DTU), read scale-out enabled, zone-redundant | Provisioned |
| `PierArchive` | Hyperscale, 8 vCores, plus two named replicas | Provisioned |
| `PierSandbox` | General Purpose, 0.5–4 vCores, auto-pause delay 60 minutes | Serverless |
| `PierBilling` | Standard S3 (DTU), "because CDC needs S3 or higher" | Provisioned |

### c.

| Database | Service tier | Compute tier |
|---|---|---|
| `PierDispatch` | Hyperscale, 8 vCores, one high-availability replica used for reports (`ApplicationIntent=ReadOnly`) | Provisioned |
| `PierArchive` | Hyperscale, 8 vCores, two **high-availability replicas** used for read scale-out | Provisioned |
| `PierSandbox` | Hyperscale, 0.5–4 vCores, auto-pause delay 60 minutes | Serverless |
| `PierBilling` | General Purpose, 4 vCores | Provisioned |

### d.

| Database | Service tier | Compute tier |
|---|---|---|
| `PierDispatch` | Business Critical, zone-redundant, 8 vCores | Provisioned |
| `PierArchive` | General Purpose, 8 vCores, maximum data size set to 20 TB | Provisioned |
| `PierSandbox` | Business Critical, 0.5–4 vCores, auto-pause delay 60 minutes | Serverless |
| `PierBilling` | General Purpose, 0.5–2 vCores, auto-pause disabled | Serverless |

## Correct Answer

**a**

## Explanation

The question is a constraint-matching exercise over the three vCore service tiers (**General Purpose**, **Business Critical**, **Hyperscale**), the two compute tiers (**provisioned**, **serverless**) and the purchasing-model constraint. The decisive facts, all from the Azure SQL Database service-tier documentation:

| Fact | General Purpose | Business Critical | Hyperscale |
|---|---|---|---|
| Storage | Premium remote storage, **5–10 ms** latency | Local SSD, **1–2 ms** average latency | Decoupled storage with local SSD cache; 1–2 ms for the hot part of the data |
| Max data size | **4 TB** | **4 TB** | **128 TB**, not configurable; billed for **allocated** storage only (min 10 GB) |
| Replicas | One replica, **no read scale-out** | Three secondary replicas; **one free read scale-out replica** | 0–4 HA replicas (each replica's vCores are **billed**); up to **30 named replicas** with independent compute |
| Serverless | Yes, **only tier with auto-pause/auto-resume** | **Not supported** | Yes, but **no auto-pause** |
| In-memory OLTP | No | Yes | No |
| Zone redundancy | Yes | Yes | Yes |

### Why option a is correct

- **`PierDispatch` → Business Critical, provisioned.** Business Critical is the tier "designed for applications that require low-latency responses from the underlying SSD storage (1–2 ms in average)" and its cluster has a built-in read scale-out capability that "provides a free-of-charge read-only replica" for reports — the two requirements together identify this tier and no other. Zone redundancy is available. 900 GB is within the 4 TB limit. Provisioned compute is required anyway: Business Critical has no serverless tier.
- **`PierArchive` → Hyperscale, provisioned, named replicas.** 20 TB exceeds the 4 TB ceiling of General Purpose and Business Critical; Hyperscale supports up to 128 TB, and "you're charged only for the allocated data storage, not for maximum data storage". Read scale-out with **independently sized** replicas is exactly what Hyperscale **named replicas** provide ("up to 30 named replicas with independent configurable compute"); they are created and dropped independently of the primary.
- **`PierSandbox` → General Purpose, serverless with auto-pause.** Serverless "automatically pauses databases during inactive periods when only storage is billed and automatically resumes databases when activity returns", and "auto-pause and auto-resume are only supported in the General Purpose service tier". The first connection after a pause receives error 40613 and the client retries; databases generally resume in under a minute, which the team accepts. The auto-pause delay defaults to 60 minutes (allowed range 15 minutes to 7 days, `-1` disables it); compute is billed per second only while the database is running, never below the configured minimum vCores.
- **`PierBilling` → General Purpose, provisioned.** A steady 24x7 workload with moderate latency tolerance is the documented fit for provisioned General Purpose: "Single databases with more regular, predictable usage patterns and higher average compute utilization over time" belong in provisioned compute (fixed hourly price), and General Purpose is "the default service tier ... designed for most of generic workloads" with 5–10 ms storage latency. CDC is supported in every vCore tier — its restriction (S3 or higher) applies only to the DTU model.

### Why option b is wrong

Two of the four databases are placed on **DTU** tiers, which violates the stated purchasing-model constraint: Azure Hybrid Benefit and reserved capacity are **vCore-only** benefits. This is the subtle distractor because, technically, Premium does offer 2 ms I/O latency, a read scale-out replica and zone redundancy, and Standard S3 is indeed the minimum DTU objective for CDC — so both assignments *look* justified. They fail on the purchasing model, not on the feature. (The DTU model also gives no choice of hardware and no serverless or Hyperscale; the documentation recommends vCore for new deployments.)

### Why option c is wrong

- `PierDispatch` on Hyperscale: Hyperscale can give 1–2 ms latency for hot data, but its high-availability replicas are **billed** ("vCore for each replica ... charged"). The requirement was a read replica **included at no additional compute charge** — the free replica exists only in Business Critical (and Premium).
- `PierArchive` with HA replicas for read scale-out: HA replicas follow the primary's compute tier and size ("secondary high-availability compute node replicas in Hyperscale follow the compute tier of the primary"), so they cannot be **sized independently**; that is what named replicas are for.
- `PierSandbox` on Hyperscale serverless: serverless is available in Hyperscale, but **auto-pause is not** — "Currently, auto-pause and auto-resume are only supported in the General Purpose service tier". Compute would keep being billed at the minimum vCores all night and all weekend.

### Why option d is wrong

- `PierArchive` on General Purpose with "maximum data size 20 TB": impossible — General Purpose storage is 1 GB to **4 TB**.
- `PierSandbox` on Business Critical serverless: the serverless compute tier is **not supported** in Business Critical (supported: General Purpose and Hyperscale, on Standard-series hardware only).
- `PierBilling` on serverless with auto-pause disabled: for a database that runs 24x7 at a steady load, serverless never pauses and bills per second at a higher unit price than provisioned; the documentation places "more regular, predictable usage patterns with higher average compute utilization" in the **provisioned** tier, which also gives the *predictable* monthly cost the finance team asked for.

Conceptual question (Azure / tooling); not executed against an engine.

## DP-800 Exam Rule to Remember

```text
Purchasing model   DTU (Basic / Standard / Premium): bundled, no hardware choice, no AHB, no reservations,
                   no serverless, no Hyperscale; CDC needs S3+; Premium = 2 ms + free read replica + in-memory
                   vCore (GP / BC / Hyperscale): AHB, reservations, serverless, hardware choice -> recommended

Service tier       GP  : remote storage 5-10 ms, 4 TB, 1 replica, NO read scale-out, cheapest
                   BC  : local SSD 1-2 ms, 4 TB, 3 secondaries + 1 FREE read-only replica, in-memory OLTP, ~2.7x GP
                   HS  : 128 TB, pay for allocated storage, 0-4 HA replicas (billed), up to 30 NAMED replicas
                         (independent size), fast backup/restore, no in-memory OLTP, 1-2 ms for hot data

Compute tier       Provisioned: fixed vCores, per-hour, predictable steady load
                   Serverless : per-second, min/max vCores, autoscale; auto-pause (15 min - 7 days, default 60)
                                ONLY in General Purpose; Hyperscale serverless never pauses; BC has no serverless;
                                first login after pause -> error 40613, retry, resume in ~1 min
```

Read the requirements for the three discriminators: **latency under 2 ms + free replica → Business Critical**, **more than 4 TB or independently sized read replicas → Hyperscale**, **zero compute while idle → General Purpose serverless with auto-pause**. Everything else steady and cost-driven is General Purpose provisioned — and a DTU tier is a wrong answer whenever the scenario mentions Azure Hybrid Benefit or reservations.
