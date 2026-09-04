# Instructor-Examiner guide — Service Tiers 1

Companion to [service_tiers_1.md](service_tiers_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.
7. **This is a multiple-choice question with no code.** It is a four-database matching exercise over service tier and compute tier. Read all four options, pieces 6 to 9, before taking an answer. Take one letter as the answer. If the learner prefers, let them first name a tier for each database, then map that to a letter, but the final answer is one letter.
8. Do not forget the purchasing-model constraint in piece 1. It decides one of the options on its own. Make sure the learner heard it.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Secure, optimize, and deploy database solutions (35–40%).
- Skill: Optimize database performance.
- Task bullet: Select the appropriate purchasing model, service tier and compute tier for Azure SQL Database.
- What is tested: matching latency, size, replica, idle-cost and predictability requirements to General Purpose, Business Critical or Hyperscale, provisioned or serverless, within the vCore purchasing model.

## 2. Scenario to read aloud

**Piece 1, the story and the constraint.** "NorthPier Logistics is consolidating four workloads onto Azure SQL Database, on the logical server northpier dash sql dot database dot windows dot net. The company applies Azure Hybrid Benefit and reserved capacity wherever a tier allows it, so every database must be created in the vCore purchasing model. Each database has its own requirements. I will describe the four databases one at a time."

**Piece 2, PierDispatch.** "PierDispatch is truck-dispatch OLTP, nine hundred gigabytes, thousands of small writes per second. Storage I O latency must be consistently below two milliseconds. The dispatch reports must run on a read-only replica that is included in the price, with no additional compute charge for it. Zone-redundant high availability is required."

**Piece 3, PierArchive.** "PierArchive is shipment history, twenty terabytes today, growing about one terabyte a year, read-mostly. Analysts need at least two read-only replicas that can be sized independently of the primary, and added or removed without touching it. The company wants to pay only for the storage actually allocated, not for a configured maximum."

**Piece 4, PierSandbox.** "PierSandbox is a forty gigabyte development copy used by contractors for a few hours on weekdays, idle at night and on weekends. Compute must cost nothing while the database is idle. The database must come back automatically on the first connection. A warm-up of about a minute after idle periods is acceptable."

**Piece 5, PierBilling.** "PierBilling is invoicing, two hundred fifty gigabytes, steady and predictable load twenty-four by seven with occasional peaks. Latency of five to ten milliseconds is acceptable. The database uses change data capture for a downstream feed. The finance team wants the lowest predictable monthly cost and no read replica."

**Piece 6, option a.** "Option a. PierDispatch: Business Critical, zone-redundant, eight vCores, provisioned. PierArchive: Hyperscale, eight vCores, plus two named replicas of four and sixteen vCores, provisioned. PierSandbox: General Purpose, zero point five to four vCores, auto-pause delay sixty minutes, serverless. PierBilling: General Purpose, four vCores, provisioned."

**Piece 7, option b.** "Option b. PierDispatch: Premium P4, which is a DTU tier, with read scale-out enabled and zone-redundant, provisioned. PierArchive: Hyperscale, eight vCores, plus two named replicas, provisioned. PierSandbox: General Purpose, zero point five to four vCores, auto-pause delay sixty minutes, serverless. PierBilling: Standard S3, a DTU tier, quote, because CDC needs S3 or higher, provisioned."

**Piece 8, option c.** "Option c. PierDispatch: Hyperscale, eight vCores, one high-availability replica used for reports with ApplicationIntent equals ReadOnly, provisioned. PierArchive: Hyperscale, eight vCores, two high-availability replicas used for read scale-out, provisioned. PierSandbox: Hyperscale, zero point five to four vCores, auto-pause delay sixty minutes, serverless. PierBilling: General Purpose, four vCores, provisioned."

**Piece 9, option d.** "Option d. PierDispatch: Business Critical, zone-redundant, eight vCores, provisioned. PierArchive: General Purpose, eight vCores, maximum data size set to twenty terabytes, provisioned. PierSandbox: Business Critical, zero point five to four vCores, auto-pause delay sixty minutes, serverless. PierBilling: General Purpose, zero point five to two vCores, auto-pause disabled, serverless."

## 3. Setup script (reference only; do not read verbatim unless asked)

There is no code in this question. The requirements table, verbatim:

| Database | Workload and requirements |
|---|---|
| `PierDispatch` | Truck-dispatch OLTP, 900 GB, thousands of small writes per second. Storage I/O latency must be consistently below 2 ms. The dispatch reports must run on a read-only replica that is included in the price, no additional compute charge for it. Zone-redundant high availability is required. |
| `PierArchive` | Shipment history, 20 TB today, growing about 1 TB a year, read-mostly. Analysts need at least two read-only replicas that can be sized independently of the primary and added or removed without touching it. The company wants to pay only for the storage actually allocated, not for a configured maximum. |
| `PierSandbox` | 40 GB development copy used by contractors for a few hours on weekdays, idle at night and on weekends. Compute must cost nothing while the database is idle, the database must come back automatically on the first connection, and a warm-up of about a minute after idle periods is acceptable. |
| `PierBilling` | Invoicing, 250 GB, steady and predictable load 24x7 with occasional peaks. Latency of 5–10 ms is acceptable. The database uses change data capture for a downstream feed. The finance team wants the lowest predictable monthly cost and no read replica. |

## 4. The question (ask exactly this)

"Which combination of service tier and compute tier meets every requirement? Option a, b, c or d?"

- **a.** PierDispatch: Business Critical, zone-redundant, 8 vCores, Provisioned. PierArchive: Hyperscale, 8 vCores, plus two **named replicas** (4 and 16 vCores), Provisioned. PierSandbox: General Purpose, 0.5–4 vCores, auto-pause delay 60 minutes, Serverless. PierBilling: General Purpose, 4 vCores, Provisioned.
- **b.** PierDispatch: Premium P4 (DTU), read scale-out enabled, zone-redundant, Provisioned. PierArchive: Hyperscale, 8 vCores, plus two named replicas, Provisioned. PierSandbox: General Purpose, 0.5–4 vCores, auto-pause delay 60 minutes, Serverless. PierBilling: Standard S3 (DTU), "because CDC needs S3 or higher", Provisioned.
- **c.** PierDispatch: Hyperscale, 8 vCores, one high-availability replica used for reports (`ApplicationIntent=ReadOnly`), Provisioned. PierArchive: Hyperscale, 8 vCores, two **high-availability replicas** used for read scale-out, Provisioned. PierSandbox: Hyperscale, 0.5–4 vCores, auto-pause delay 60 minutes, Serverless. PierBilling: General Purpose, 4 vCores, Provisioned.
- **d.** PierDispatch: Business Critical, zone-redundant, 8 vCores, Provisioned. PierArchive: General Purpose, 8 vCores, maximum data size set to 20 TB, Provisioned. PierSandbox: Business Critical, 0.5–4 vCores, auto-pause delay 60 minutes, Serverless. PierBilling: General Purpose, 0.5–2 vCores, auto-pause disabled, Serverless.

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: a.**

| Database | Tier and compute | Decisive facts |
|---|---|---|
| PierDispatch | Business Critical, zone-redundant, provisioned | Local SSD, 1 to 2 ms average latency; one free read-only replica via read scale-out; zone redundancy available; 900 GB within 4 TB; no serverless in Business Critical anyway. |
| PierArchive | Hyperscale, provisioned, two named replicas | 20 TB exceeds the 4 TB ceiling of GP and BC; Hyperscale up to 128 TB, billed for allocated storage only; up to 30 named replicas with independently configurable compute, created and dropped without touching the primary. |
| PierSandbox | General Purpose, serverless, auto-pause 60 minutes | Auto-pause and auto-resume are only supported in General Purpose; only storage is billed while paused; first connection after a pause gets error 40613 and the client retries; resume in about a minute. |
| PierBilling | General Purpose, 4 vCores, provisioned | Steady, predictable 24x7 usage belongs in provisioned compute with a fixed hourly price; 5 to 10 ms storage latency is acceptable; CDC works in every vCore tier. |

Why the wrong options are wrong:

- **b.** Premium P4 and Standard S3 are DTU tiers. Azure Hybrid Benefit and reserved capacity are vCore-only, so the purchasing-model constraint is violated. Technically Premium gives 2 ms latency, a free read replica and zone redundancy, and S3 is the CDC minimum in the DTU model, so both rows look justified; they fail on the purchasing model.
- **c.** Hyperscale high-availability replicas are billed per vCore, so PierDispatch's replica is not free. HA replicas follow the primary's compute size, so PierArchive's replicas cannot be sized independently. Hyperscale serverless never auto-pauses, so PierSandbox would bill minimum vCores all night.
- **d.** General Purpose storage is 1 GB to 4 TB, so 20 TB is impossible. Serverless is not supported in Business Critical. PierBilling on serverless with auto-pause disabled runs 24x7 at a higher per-second unit price than provisioned and gives no predictable monthly cost.

## 6. Hint ladder (one hint per attempt, in order)

1. "Piece 1 said every database must be in the vCore purchasing model, because of Azure Hybrid Benefit and reservations. Go through the options and look for tier names that are not vCore tiers. Names like Premium P4 or Standard S3 belong to which model?"
2. "For PierDispatch: under two milliseconds and a read replica at no extra compute charge. Which vCore tier keeps data on local SSD, and which tier gives one read-only replica free of charge through read scale-out? Is it the same tier?"
3. "For PierArchive: twenty terabytes. What is the maximum data size for General Purpose and Business Critical? Which tier goes to one hundred twenty-eight terabytes and bills only allocated storage?"
4. "Still PierArchive: replicas sized independently of the primary. Hyperscale has two kinds of replicas. High-availability replicas follow the primary's size. What is the other kind called?"
5. "For PierSandbox: compute must cost nothing while idle. Serverless can auto-pause, but only in one service tier. Which one? Does Hyperscale serverless pause? Does Business Critical even offer serverless?"
6. "For PierBilling: steady, predictable, twenty-four by seven, lowest predictable cost. Is a per-second serverless bill predictable for a database that never pauses? Which compute tier has a fixed hourly price?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "b, Premium has 2 ms latency and a free replica, and S3 is the CDC minimum" | Judges features and forgets the purchasing model | "Both statements about Premium and S3 are true. Now recall what piece 1 said about Azure Hybrid Benefit and reserved capacity. Which purchasing model do they require?" |
| "c, Hyperscale has SSD cache so it meets 2 ms and the HA replica can serve reports" | Thinks Hyperscale HA replicas are free | "Is the compute of a Hyperscale high-availability replica included in the primary's price, or charged per vCore?" |
| "c, two HA replicas give read scale-out" | Confuses HA replicas with named replicas | "Can an HA replica have a different vCore count from the primary? Which replica type can?" |
| "c, Hyperscale serverless will pause the sandbox" | Assumes serverless always auto-pauses | "Serverless exists in Hyperscale, yes. But which tier is the only one where auto-pause and auto-resume are supported?" |
| "d, set the General Purpose max size to 20 TB" | Does not know the 4 TB ceiling | "What is the largest max data size General Purpose allows?" |
| "d, Business Critical serverless for the sandbox" | Does not know serverless is GP and Hyperscale only | "Which two service tiers support the serverless compute tier?" |
| "d, serverless for billing since it autoscales for the peaks" | Confuses autoscale with predictability | "The workload runs all day at steady load. Is a per-second bill lower or higher than a fixed hourly price in that case, and is it predictable?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain the three discriminators and the purchasing-model rule:

- **Purchasing model.** DTU tiers, Basic, Standard and Premium, are bundled: no hardware choice, no Azure Hybrid Benefit, no reservations, no serverless, no Hyperscale; CDC needs S3 or higher there; Premium gives 2 ms, a free read replica and in-memory OLTP. vCore tiers, General Purpose, Business Critical and Hyperscale, support Azure Hybrid Benefit, reservations, serverless and hardware choice, and are recommended for new deployments. Any scenario that mentions Azure Hybrid Benefit or reservations rules out DTU.
- **General Purpose.** Premium remote storage with 5 to 10 ms latency, up to 4 TB, one replica, no read scale-out, cheapest. Serverless here is the only place with auto-pause and auto-resume. Auto-pause delay defaults to 60 minutes, range 15 minutes to 7 days, minus one disables it. While paused only storage is billed. The first login after a pause gets error 40613, the client retries, and the database resumes in about a minute. Compute is billed per second while running, never below the configured minimum vCores.
- **Business Critical.** Local SSD, 1 to 2 ms average latency, up to 4 TB, three secondary replicas and one free read-only replica through read scale-out, in-memory OLTP, roughly 2.7 times the price of General Purpose. No serverless tier.
- **Hyperscale.** Up to 128 TB, not configurable, billed for allocated storage only, minimum 10 GB. Zero to four high-availability replicas, each billed per vCore and following the primary's compute tier. Up to 30 named replicas with independent, configurable compute, created and dropped without touching the primary. Fast backup and restore. No in-memory OLTP. Serverless is available but never auto-pauses. 1 to 2 ms for the hot part of the data thanks to the local SSD cache.
- **Compute tier.** Provisioned: fixed vCores, per-hour, for regular, predictable usage with higher average utilisation. Serverless: per-second, min and max vCores, autoscale, for intermittent or unpredictable usage. CDC is supported in every vCore tier.
- **Reading the four rows.** Under 2 ms plus a free replica is Business Critical. More than 4 TB or independently sized replicas is Hyperscale with named replicas. Zero compute while idle is General Purpose serverless with auto-pause. Steady and cost-driven is General Purpose provisioned.

Memory hook: "Two milliseconds and a free replica: Business Critical. Over four terabytes or named replicas: Hyperscale. Pause when idle: General Purpose serverless. Steady and cheap: General Purpose provisioned. Hybrid Benefit means vCore."

## 9. Follow-up oral questions (optional)

1. "What error does the first connection receive when a paused serverless database is resuming, and what should the client do?" (Error 40613, database not currently available. Retry; the database resumes in about a minute.)
2. "Could PierSandbox use Hyperscale serverless to get the same idle-cost behaviour?" (No. Hyperscale serverless scales but does not auto-pause, so compute is billed at least at the minimum vCores all the time.)
3. "If PierArchive later needed a second read replica of a different size, what would you do?" (Create another named replica with its own vCore count; it does not touch the primary.)

## 10. References

- vCore purchasing model overview: https://learn.microsoft.com/en-us/azure/azure-sql/database/service-tiers-sql-database-vcore
- Compare vCore and DTU purchasing models: https://learn.microsoft.com/en-us/azure/azure-sql/database/purchasing-models
- General Purpose service tier: https://learn.microsoft.com/en-us/azure/azure-sql/database/service-tier-general-purpose
- Business Critical service tier: https://learn.microsoft.com/en-us/azure/azure-sql/database/service-tier-business-critical
- Hyperscale service tier: https://learn.microsoft.com/en-us/azure/azure-sql/database/service-tier-hyperscale
- Hyperscale named replicas: https://learn.microsoft.com/en-us/azure/azure-sql/database/service-tier-hyperscale-replicas
- Serverless compute tier: https://learn.microsoft.com/en-us/azure/azure-sql/database/serverless-tier-overview
- Read scale-out: https://learn.microsoft.com/en-us/azure/azure-sql/database/read-scale-out
- Azure Hybrid Benefit: https://learn.microsoft.com/en-us/azure/azure-sql/azure-hybrid-benefit
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
