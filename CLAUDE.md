# Ledger

Working name (placeholder). A self-hosted **evidence plane** for AI agents: it turns what agents actually do into EU AI Act-ready proof — a tamper-evident audit log, human-oversight records, an agent register with risk classification, and exportable evidence packs.

It is a **standalone service**: it runs on its own, ingests agent activity directly over OpenTelemetry, and doesn't need any other platform to work. It can run next to existing observability (Langfuse, Phoenix, WSO2 Agent Manager) but must never depend on it. It is not an agent runtime, a guardrails catalogue, or a tracing tool.

## Status

Early. Private repo. Built slowly, by one person, alongside client work.

**Current milestone: evidence pack spec** (`docs/evidence-pack.md`). No application code until it exists. For each section: what it contains, which AI Act article it answers, and what an auditor would reject.

## Stack

- Python + FastAPI (API, ingest, CLI)
- PostgreSQL
- OpenTelemetry (GenAI semantic conventions) for agent activity

Don't add other services or frameworks (Next.js console, OPA, Keycloak, ClickHouse) until a milestone needs them. Ask first.

## Principles

- **Evidence, never compliance claims.** Copy, docs and output say "evidence and documentation support". Never "makes you compliant".
- **Cite the article.** Every feature or evidence section names the AI Act article it supports. Article references follow Regulation (EU) 2024/1689 as amended by the Digital Omnibus (Regulation (EU) 2026/1744); mark anything not checked against the consolidated text as `[unverified]`.
- **EU-only and self-hosted.** No outbound calls by default, no telemetry home, air-gap friendly.
- **Audit log is append-only.** Records are hash-chained; never write code that updates or deletes audit records.
- **Open-core boundary from day one.** Core and enterprise features stay separable, with no shortcuts across the boundary. The license is not decided yet; don't add a LICENSE file or license headers.
- **Reuse standards, don't rebuild them.** Prefer OTel, existing instrumentors and established libraries over custom code.

## Working rules

- Keep changes small and in scope. Ask when something is unclear instead of guessing.
- Check your work (run it, test it) before calling it done.