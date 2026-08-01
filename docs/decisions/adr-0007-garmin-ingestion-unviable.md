# ADR 0007 – Garmin Ingestion Path Renders the Solution Unviable

## Status
Accepted (2026-08-01). Blocks the roadmap Phases 4–5 until scope is redefined.

## Context

The AnalyticsHealth product pitch (see [`docs/index.md`](../index.md)) rests on **silent, automatic ingestion of Garmin data every night** into an AWS-hosted pipeline (S3 → consolidator → Bedrock KB → `/chat` → WhatsApp). Two of the five [guiding principles](../index.md#guiding-principles) are load-bearing here:

- **Idle cost ≈ $0** — pay-per-use, nothing always-on while not in use.
- **Your data, your account** — single-tenant, everything inside the user's AWS account.

Phase 2/3 delivered exactly that: a `fetch_garmin` Lambda calling Garmin Connect via the community library `garminconnect` (which internally uses `garth`), scheduled by EventBridge at 06:00 UTC.

Between May and August 2026 the daily pull began failing silently. The audit on 2026-08-01 (see [status/20260801-status.md](../../status/20260801-status.md), §0.2) established the root cause:

> Garmin Connect blocks authentication attempts originating from datacenter IPs (AWS Lambda source ranges), returning `HTTP 429` (mobile login), `CAPTCHA_REQUIRED` (widget/portal login), and `HTTP 403` (portal fallback) across **all four** login strategies of `garminconnect`. The credentials are valid — the block is IP-based.

This is not a bug in our code, not a library version issue, and not a credentials issue. It is a policy of the data provider.

## Options Evaluated

Four paths were considered to unblock the ingestion:

### A — Upgrade `garminconnect` / switch to `garth` directly
- **Result:** does not help. Both libraries hit the same anti-bot layer. The problem is source-IP reputation, not client behaviour.

### B — Run coleta outside AWS on a residential IP
Variants: Mac + `launchd`, Raspberry Pi at home, small VPS with residential-grade IP.
- **Technically works.** Reuses 100% of the existing Python code, ships in 1–2 days.
- **Violates a core project principle.** Moves the primary data-collection component **outside the AWS account**, breaking "Your data, your account" as originally scoped. The narrative of a fully serverless, single-tenant AWS solution collapses at the first hop.
- **Operational fragility.** Depends on a specific machine being powered on, connected, and unblocked. Introduces a class of failure (host down, ISP outage) that the AWS-native architecture explicitly avoids.

### C — Connect IQ SDK (push from the watch)
A dedicated research memo was produced during the evaluation covering the Monkey C `Toybox.SensorHistory`, `Toybox.Communications` and `Toybox.Background` surfaces, with sources and gaps flagged. Summarised here.
- **Stays inside AWS.** Watch pushes directly to API Gateway.
- **Loses the highest-value data.** `Toybox.SensorHistory` does **not** expose Sleep Score, HRV, or reliable Body Battery/Stress. HRV/Sleep are computed server-side by Garmin (Firstbeat) and are not accessible to Monkey C. Sleep + HRV are precisely the recovery signals the product pitch centres on ("*Ana, how was my recovery this week?*").
- **Requires BLE + phone-with-internet nearby.** Any temporal event firing while the watch is charging, out of range, or offline is lost silently.
- **Doubles the ingestion surface.** Even in the best case, Connect IQ has to be combined with option B for sleep/HRV — so it does not remove the residential-IP dependency, it adds a second pipeline on top of it.
- **Effort:** 2–3 weeks to first end-to-end delivery (Monkey C, sideload, API GW, Lambda receiver, idempotency, in-watch validation).

### D — Garmin Health API (official B2B)
- Enterprise-only, requires Garmin approval with a real product/business case.
- New registrations were reported as suspended in 2026.
- Not viable for personal / learning use.

### E — NAT Gateway with a dedicated Elastic IP
- Would still originate from AWS-owned IP space — the block would recur once the IP is fingerprinted.
- Also fires the "$32/month always-on" cost against the idle-cost principle.
- Not viable.

## Decision

**The AnalyticsHealth solution as originally scoped is not viable.**

There is no path that simultaneously satisfies:

1. **Automatic** ingestion (not manual export via GDPR page).
2. **Complete** ingestion — sleep and HRV included, which are the recovery signals the product depends on.
3. **AWS-native** collection layer — the "your data, your account" principle applied end-to-end.
4. **Idle cost ≈ $0** — no always-on residential infrastructure priced against the project.

Every option evaluated fails at least one of (1)–(4):

| | Automatic | Sleep + HRV | Collection in AWS | Idle ≈ $0 |
|---|---|---|---|---|
| A (`garminconnect` from Lambda) | ✅ | ✅ | ✅ | ✅ but **does not work** |
| B (residential host) | ✅ | ✅ | ❌ | ⚠️ (hidden cost of a Mac/Pi always on) |
| C (Connect IQ) | ⚠️ (BLE-dependent) | ❌ | ✅ | ✅ |
| D (Garmin Health API) | ✅ | ✅ | ✅ | ✅ but **inaccessible** |
| E (NAT + EIP) | ✅ | ✅ | ✅ but **will be re-blocked** | ❌ |

The blocker is not architectural — it is that **the data source refuses programmatic access from the environment the solution was designed for**. No amount of AWS engineering fixes this.

## Consequences

### Immediate
- **Phase 4 (Consolidator + RAG) is blocked** in its intended form. It could still be built and demoed against the historical corpus in S3 (6,317 objects, 2021-12 → 2026-05, loaded via the local `initial_load` script), but the "silently ingest every night" promise is not deliverable.
- **Phase 5 (Scale to family/friends) is blocked absolutely** — none of the options above scale to N users without either N residential hosts, N enterprise approvals, or the same Connect IQ limitations replicated N times.
- **The daily EventBridge schedule remains disabled** (executed 2026-08-01, see [status/20260801-status.md](../../status/20260801-status.md), §0.3). The Lambda, Step Functions state machine, DLQ, and alarms are preserved in AWS but not scheduled.

### Product implications
- The [`docs/index.md`](../index.md) pitch ("*She silently ingests your Garmin data every night*") is currently false as written. The homepage must either be revised to reflect the actual capability or the product must be re-scoped.
- The Textract-based OCR channel (scale photos via WhatsApp) and the manual-log channel remain fully viable and are not affected by this ADR.

### Strategic
- Continuing the project requires an **explicit scope redefinition**. Candidate re-scopings, none accepted here:
  - **Historical-only RAG** — build Phase 4 over the frozen 2021→2026-05 corpus. Ana becomes a "your health history" assistant, not a "your last night's recovery" assistant.
  - **OCR/manual-only** — drop Garmin entirely, lean on WhatsApp photo + text logs.
  - **Accept option B as a pragmatic compromise** — formally amend the "your data, your account" principle to "primary data plane in your account; a thin residential collector is acceptable." This preserves the product but weakens the architectural narrative.
- Choosing among these is a **product decision, not an engineering one**, and is not made in this ADR.

## Revisit Triggers

- Garmin publishes an official personal-use API (unlikely).
- Garmin Health API reopens to non-enterprise applicants.
- The `garminconnect` / `garth` community publishes a reliable bypass for datacenter-IP blocking that survives more than one release cycle.
- Scope is redefined and this ADR is superseded by an ADR that accepts option B/C as an intentional trade-off.
