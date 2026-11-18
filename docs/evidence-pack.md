# Evidence pack specification

Status: **draft**. Only section 2 (audit log) is written; the other sections are placeholders.

Article references follow Regulation (EU) 2024/1689 as amended by the Digital Omnibus (Regulation (EU) 2026/1744). None have been checked against the consolidated text yet, so every reference is marked `[unverified]` until it is.

## What an evidence pack is

An evidence pack is a bundle that Ledger exports for one or more agents over a fixed time window. A provider or deployer hands it to an auditor, a notified body or a market surveillance authority.

The pack provides evidence and documentation support. It records what happened and how that record can be checked. It never states that a system is compliant.

Each section below answers three questions:

- **Contains**: what is in the section.
- **Answers**: which AI Act article it supports.
- **Rejected if**: what an auditor would not accept.

## Sections

1. Manifest and integrity (placeholder)
2. Audit log (drafted)
3. Human oversight records (placeholder)
4. Agent register and risk classification (placeholder)
5. Technical documentation support (placeholder)

---

## 1. Manifest and integrity

_Not drafted._

- **Contains (sketch):** pack ID, creation time, Ledger version, exporter role (provider or deployer), agents and time window covered, list of files with their hashes, signature over the manifest, the OTel GenAI semantic conventions version used at ingest.
- **Answers:** no article of its own. It makes every other section verifiable.
- **Rejected if:** TBD.

---

## 2. Audit log

A record of what each agent did during the pack window, in an order that can be checked: model calls, tool calls, agent hand-offs, errors and human interventions.

### 2.1 Answers

| Article | What it asks | What the section supplies |
|---|---|---|
| Art. 12(1) `[unverified]` | High-risk systems technically allow automatic recording of events over their lifetime. | The recorded events. The capability itself belongs to the agent: it must be instrumented with OTel, and Ledger is where the records land. Ledger alone does not give an agent this capability. |
| Art. 12(2)(a)–(c) `[unverified]` | Logging enables identifying risk situations or substantial modification, post-market monitoring (Art. 72), and deployer monitoring (Art. 26(5)). | Event types and fields chosen so each purpose can be traced back to records (see 2.2). |
| Art. 19(1) `[unverified]` | Providers keep automatically generated logs under their control for an appropriate period, at least six months unless other Union or national law says otherwise. | Retention statement (2.6). |
| Art. 26(6) `[unverified]` | The same retention duty for deployers. | Retention statement (2.6). |
| Art. 21(2) `[unverified]` | On reasoned request, providers give the competent authority access to the logs. | The pack is the format for handing them over. |

**Agents that are not high-risk.** None of the articles above require this log for an agent the register does not classify as high-risk. The pack still includes the log, but labels it as voluntary evidence and does not cite Art. 12 as an obligation for that agent.

### 2.2 Contains: scope statement

- Agents covered, by register ID, and every agent version active in the window.
- Time window in UTC, and the environment (e.g. production).
- The exporter's role (provider, deployer or both), which decides which retention article applies.

### 2.3 Contains: records

One record per ingested event. Fields:

| Field | Source | Notes |
|---|---|---|
| `seq` | Ledger | Increases by one per record in the chain. No gaps. |
| `recorded_at` | Ledger ingest clock | UTC. |
| `event_start`, `event_end` | OTel span timestamps | UTC. |
| `agent_id`, `agent_version` | `gen_ai.agent.id`, mapped to the register | Must resolve to a register entry (section 4). |
| `trace_id`, `span_id`, `parent_span_id` | OTel | Lets the auditor rebuild the call tree. |
| `event_type` | `gen_ai.operation.name` (e.g. `chat`, `invoke_agent`, `execute_tool`), plus Ledger-native types | Ledger-native types: `human_intervention`, `ingest_gap`, `config_change`. |
| `actor` | Agent ID, or the identified person for human events | Links to section 3. |
| `model` | `gen_ai.request.model`, `gen_ai.response.model` | Both, when they differ. |
| `tool` | `gen_ai.tool.name` | For tool calls. |
| `outcome` | Span status, `error.type` | |
| `content_ref` | Ledger | Hash of the input/output content, plus a pointer to the content store if content was captured. See open question on personal data. |
| `prev_hash`, `record_hash` | Ledger | The hash chain. |

The OTel GenAI semantic conventions are still in development status, and attribute names can change between versions. The manifest records which version was used at ingest.

### 2.4 Contains: chain verification material

- Hash algorithm and the canonical serialisation used to hash a record.
- First and last `seq` and hash in the window, plus the hash of the record just before the window. This lets an auditor verify an exported slice without the full chain.
- Signed checkpoints of the chain head taken during the window, with the signing key ID.
- A verification procedure the auditor can run offline, without a running Ledger instance.

### 2.5 Contains: coverage statement

- **Sampling.** The sampler configuration at the ingest boundary. Agent spans must be recorded at 100%. Any period with lower sampling is listed.
- **Content capture.** Whether prompt and response content was captured, per agent and period. GenAI instrumentors do not capture message content by default.
- **Known gaps.** Every ingest outage or detected loss (e.g. spans whose parent never arrived, collector drop counts), with time range and cause. Each gap is also an `ingest_gap` record in the chain.
- **Clock.** The clock source, and the largest observed difference between event time and `recorded_at`.

### 2.6 Contains: retention statement

- Retention period configured per agent.
- Earliest record still held.
- Whether any part of the window falls outside what is still held.

### 2.7 Rejected if

1. `seq` has gaps, or the chain breaks, and the coverage statement does not explain it.
2. The chain can only be verified with Ledger itself. The auditor needs a documented offline procedure.
3. The chain has no external anchor. Anyone who controls the database can recompute a hash chain from start to end. Without signed checkpoints whose key or head hashes live outside that database, the log is not meaningfully tamper-evident.
4. Sampled telemetry is presented as a complete record.
5. Records don't resolve to a registered agent and version.
6. Only model calls are logged. Agents act through tools; without tool calls the auditor can see what the agent said but not what it did.
7. Timestamps lack a timezone or a stated clock source, or event times come after ingest times with no explanation.
8. A high-risk agent's logs are kept for less than six months, or no retention period is stated `[unverified]`.
9. Art. 12 is cited for an agent the register does not classify as high-risk (overclaiming), or a high-risk agent has no log at all.
10. Human interventions in section 3 have no matching `human_intervention` records here, or the other way round.

### 2.8 Open questions

- **Personal data vs. an append-only log.** Prompts and outputs often contain personal data. GDPR storage limits and erasure requests conflict with a log that never deletes. One option: the chain holds only content hashes, and content sits in a separate store that can expire or be crypto-shredded without breaking the chain. This needs deciding before the record schema is final.
- **Anchoring without outbound calls.** Where the signing key lives (file, HSM) and where checkpoint heads go so they sit outside the operator's database. RFC 3161 timestamping needs an outbound call, so it can only be opt-in.
- **Content capture policy.** Required, recommended or optional, and does that depend on risk class?
- **Chain order.** Proposal: chain order is ingest order, not event order. OTel exports a span when it ends, so a parent `invoke_agent` span arrives after its children. Event order can be rebuilt from timestamps and span IDs.
- **Granularity.** One record per span, or also one per span event / GenAI log event?
- **Hashing.** Proposal: SHA-256 over JSON canonicalised with RFC 8785 (JCS), to reuse a standard instead of inventing a serialisation.
- **Remote biometric identification.** Art. 12(3) `[unverified]` adds minimum logging fields for these systems. Support them, or exclude them from v1 and say so in the pack?

---

## 3. Human oversight records

_Not drafted._

- **Contains (sketch):** who is assigned oversight for each agent, with their competence, training and authority. Each intervention (approve, reject, override, stop): who did it, when, and which audit log records it links to.
- **Answers:** Art. 14 `[unverified]` (system designed for oversight, including override and stop). Art. 26(2) `[unverified]` (deployer assigns oversight to competent people).
- **Rejected if:** TBD.

---

## 4. Agent register and risk classification

_Not drafted._

- **Contains (sketch):** for each agent: ID, name, owner, intended purpose, versions, models and tools used, risk classification with rationale, who classified it and when.
- **Answers:** Art. 5 `[unverified]` (prohibited practices check). Art. 6 and Annex III `[unverified]` (high-risk classification). Art. 6(3)–(4) `[unverified]` (documented assessment when an Annex III system is considered not high-risk). Art. 49 `[unverified]` (registration; check whether the Omnibus changed this duty for Art. 6(3) cases). Art. 50 `[unverified]` (transparency obligations).
- **Rejected if:** TBD.

---

## 5. Technical documentation support

_Not drafted._

- **Contains (sketch):** the parts of Annex IV that Ledger can evidence from data it holds, such as the logging setup, oversight measures and monitoring. Not the full Annex IV.
- **Answers:** Art. 11 and Annex IV `[unverified]`.
- **Open:** whether this section belongs in v1 at all.
