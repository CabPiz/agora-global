# Architecture — Ágora Global

**Version:** 1.0
**Date:** 2026-08-14
**Stack:** Next.js 15 · TypeScript · Python 3.11 · scikit-learn · pandas · Supabase · Inngest · Anthropic SDK · next-intl · Docker · Fly.io
**License:** AGPL-3.0

---

## 1. Architecture Principles

**1. Score, never accusation.**
The system never makes statements about intent or illegality. It emits objective atypicity indexes based on documented mathematical rules. All AI-generated language is reviewed before publication.

**2. Full parameterization — nothing hardcoded.**
Municipality, country, score threshold, active algorithms — all configurable via environment variables or a configuration table in the database. The code doesn't know "São Paulo" exists; it knows `MUNICIPALITY_IBGE_CODE` exists.

**3. Responsibility separation by language.**
Python handles statistical batch analysis (Benford, Isolation Forest, Z-score, CNPJ risk). TypeScript handles UI, API, streaming and the Claude agent loop. They communicate exclusively via Supabase — no process coupling.

**4. Human-in-the-Loop is mandatory, not optional.**
No alert is ever published automatically. The flow `DRAFT → REVIEW → PUBLIC_AUDITED / REJECTED` with a quorum of 2 independent validators is inviolable — it cannot be bypassed by any code path.

**5. Upstream-first.**
All generic code produced in a national fork is submitted via Pull Request to agora-global before any other instance uses it. Forks do not silently evolve the core.

---

## 2. Technical Stack

| Layer | Technology | Rationale |
|---|---|---|
| Frontend / UI | Next.js 15 (App Router) + TypeScript + Tailwind | SSR for SEO of public dashboard; type-safe throughout |
| Internationalization | next-intl | Global project from day 1; retroactive i18n is expensive technical debt |
| Statistical agents | Python 3.11 + pandas + scikit-learn | Mature Isolation Forest, Benford test and Z-score implementations; no equivalent in TypeScript for large-scale analysis |
| HTTP client (Python) | httpx (async) | Async-native; used for PNCP and Portal da Transparência APIs |
| DB client (Python) | supabase-py | Direct write to Supabase without FastAPI intermediary in MVP |
| ORM / Database | Supabase (PostgreSQL) + pgvector (V2) | ACID transactions, RLS, pgvector for legislative RAG in V2 |
| Async queues | Inngest | Weekly cron to trigger Python batch; event-driven report generation |
| LLM / Agent | Anthropic SDK (claude-haiku) | Citizen Report Writer — Structured Output; Haiku for cost efficiency |
| Observability | `agent_runs` table + `tracedLLMCall` | Every Claude call logged with tokens, latency, estimated cost |
| Frontend deploy | Vercel | Free tier sufficient for MVP |
| Python deploy | Fly.io (Docker container) | Self-hostable; free tier sufficient for weekly batch |
| Audit trail | Hash Chain in PostgreSQL (SHA256) | Immutability without blockchain cost in MVP |
| Tests | Playwright (E2E) + Jest (unit) + pytest (Python) | Full coverage across both runtimes |
| CI/CD | GitHub Actions + SonarCloud | Quality gate on every PR |

---

## 3. Version Roadmap

### V1 — The Municipal Auditor (MVP)

**Scope:** São Paulo-SP as pilot municipality; parametrized structure for any municipality.
**Active AI pillars:** Tool Calling, Async/Batch, Observability, Human-in-the-Loop, Guardrails, Arch Base, Multi-Agent, Structured Outputs

```
Python Ingestor (PNCP) ──→ OCDS Normalizer ──→ Statistical Analyzers
                                                  ├── Benford Test
                                                  ├── Isolation Forest
                                                  ├── CNPJ Temporal Risk
                                                  └── Price Z-Score
                                                        │
                                             Score Aggregator
                                                        │
                                             PostgreSQL (DRAFT_PENDING_REVIEW)
                                                        │
                                             Inngest: generate citizen report
                                                        │
                                             Claude Haiku (Structured Output)
                                             → AnomalyReport validated by Zod
                                                        │
                                             HitL Panel (Next.js)
                                             → 2 auditors validate
                                                        │
                                             Public Dashboard (PUBLIC_AUDITED)
```

**Fixed cost:** $0/month (Supabase free + Vercel hobby + Fly.io free tier + minimal Claude API batch cost)

---

### V2 — The Federated Network

**Scope:** All municipalities in Brazil; State → City selector; legislative RAG; public API for journalists.
**New:** pgvector + Hybrid Search (Law 14133/2021 + SINAPI price index); public FastAPI; deliberation module (Pol.is/Bridging algorithms); multi-tenant; Model Routing (Haiku for reports, Sonnet for legislative analysis)
**Unlocked by:** grant or institutional partnership

---

### V3 — The International Hub

**Scope:** First international fork + Inter-Ágoras protocol.
**New:** Public MCP server for open data; Global Solution Catalog; Evals as CI gate
**Unlocked by:** active contributor community + international funding

---

## 4. Database Schema

### MVP Tables (V1)

```sql
-- Municipalities covered (parameterized — never hardcoded)
CREATE TABLE municipalities (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  ibge_code VARCHAR(7) UNIQUE NOT NULL,
  name VARCHAR(255) NOT NULL,
  state CHAR(2) NOT NULL,
  country_code CHAR(2) NOT NULL DEFAULT 'BR',
  status VARCHAR(20) DEFAULT 'active' CHECK (status IN ('active', 'pending', 'inactive')),
  pncp_coverage_pct NUMERIC(5,2),
  last_ingested_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- OCDS-normalized contracts
CREATE TABLE contracts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  ocid VARCHAR(100) UNIQUE NOT NULL,    -- e.g. ocds-agora-br-sp-2026-004821
  municipality_id UUID REFERENCES municipalities(id),
  contract_number VARCHAR(100),
  supplier_cnpj VARCHAR(14) NOT NULL,
  supplier_name VARCHAR(255) NOT NULL,
  supplier_cnpj_founded_at DATE,
  contract_value NUMERIC(18, 2) NOT NULL,
  procurement_method VARCHAR(50),       -- open, direct, limited
  publication_date TIMESTAMPTZ NOT NULL,
  ocds_payload JSONB NOT NULL,          -- full OCDS release
  atipicity_score NUMERIC(4, 3) DEFAULT 0.000,  -- 0.000 to 1.000
  citizen_report JSONB,                 -- Citizen Report Writer output (Zod-validated)
  status VARCHAR(30) DEFAULT 'DRAFT_PENDING_REVIEW'
    CHECK (status IN ('DRAFT_PENDING_REVIEW', 'UNDER_REVIEW', 'PUBLIC_AUDITED', 'REJECTED')),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Atypicity flags per contract
CREATE TABLE audit_flags (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  contract_id UUID REFERENCES contracts(id) ON DELETE CASCADE,
  flag_code VARCHAR(50) NOT NULL,
    -- RECENT_SUPPLIER | SINGLE_BIDDER | BENFORD_DIVERGENCE | PRICE_OUTLIER | ISOLATION_ANOMALY
  severity VARCHAR(10) CHECK (severity IN ('LOW', 'MEDIUM', 'HIGH', 'CRITICAL')),
  score_weight NUMERIC(4, 3) NOT NULL,
  description TEXT NOT NULL,
  evidence_data JSONB,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Certified citizen auditors (journalists, NGOs, observatories)
CREATE TABLE citizen_auditors (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES auth.users(id),
  display_name VARCHAR(255) NOT NULL,
  organization VARCHAR(255),
  auditor_type VARCHAR(50)
    CHECK (auditor_type IN ('journalist', 'ngo', 'observatory', 'academic', 'founder')),
  verified_at TIMESTAMPTZ,
  reviews_count INT DEFAULT 0,
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- HitL review records
CREATE TABLE auditor_reviews (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  contract_id UUID REFERENCES contracts(id),
  auditor_id UUID REFERENCES citizen_auditors(id),
  decision VARCHAR(20) CHECK (decision IN ('APPROVE', 'REJECT', 'REQUEST_MORE_INFO')),
  notes TEXT,
  reviewed_at TIMESTAMPTZ DEFAULT NOW()
);

-- Cryptographic audit trail (immutable)
CREATE TABLE audit_trail_hash_chain (
  sequence_id BIGSERIAL PRIMARY KEY,
  contract_id UUID REFERENCES contracts(id),
  action VARCHAR(50) NOT NULL,
    -- INGESTED | ANALYZED | DRAFT_CREATED | REVIEW_STARTED |
    -- APPROVED_BY_AUDITOR | REJECTED_BY_AUDITOR | PUBLISHED | REPORT_GENERATED
  actor_id UUID,
  previous_hash CHAR(64) NOT NULL,
  current_hash CHAR(64) NOT NULL,  -- SHA256(previous_hash || contract_id || action || timestamp)
  metadata JSONB DEFAULT '{}',
  timestamp TIMESTAMPTZ DEFAULT NOW()
);

-- AI observability (standard across Ágora ecosystem)
CREATE TABLE agent_runs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id TEXT NOT NULL DEFAULT 'agora-global',
  agent_name TEXT NOT NULL,
  model TEXT NOT NULL,
  input_tokens INT,
  output_tokens INT,
  estimated_cost_usd NUMERIC(10, 6),
  latency_ms INT,
  tools_called TEXT[],
  stop_reason TEXT,
  status TEXT CHECK (status IN ('success', 'error', 'timeout', 'max_iter')),
  error_message TEXT,
  metadata JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### V2 Tables (added in migration)

```sql
CREATE EXTENSION IF NOT EXISTS vector;

-- Legislative embeddings (Law 14133/2021, SINAPI price index)
CREATE TABLE embeddings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  source_id TEXT NOT NULL,
  chunk_index INT,
  content TEXT NOT NULL,
  embedding VECTOR(1536),
  fts TSVECTOR GENERATED ALWAYS AS (to_tsvector('portuguese', content)) STORED,
  metadata JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX ON embeddings USING ivfflat (embedding vector_cosine_ops);
CREATE INDEX ON embeddings USING gin(fts);
```

---

## 5. AI Model — Structured Outputs

### AnomalyReportSchema — Citizen Report Writer Agent

```typescript
// lib/ai/structured.ts
import { z } from "zod";

export const FlagExplanationSchema = z.object({
  flag_code: z.string(),
  what_it_means: z.string().describe("Plain-language explanation for any citizen"),
  evidence: z.string().describe("Objective data that triggered this flag"),
  source_url: z.string().url().optional(),
});

export const AnomalyReportSchema = z.object({
  contract_id: z.string().uuid(),
  severity: z.enum(["LOW", "MEDIUM", "HIGH", "CRITICAL"]),
  citizen_summary: z
    .string()
    .max(400)
    .describe(
      "Plain-language summary, no accusations. Max 3 sentences. " +
      "Never use: fraud, corruption, crime, embezzlement. " +
      "Use: atypical pattern, statistical inconsistency, unusual finding."
    ),
  flags_explained: z.array(FlagExplanationSchema),
  recommended_action: z.enum(["monitor", "investigate", "escalate"]),
  confidence_note: z
    .string()
    .describe(
      "Note on limitations of automated analysis. " +
      "Always mention that analysis is statistical and requires human validation."
    ),
});

export type AnomalyReport = z.infer<typeof AnomalyReportSchema>;
```

### Citizen Report Writer — Agent Call

```typescript
// lib/ai/agent.ts
const CITIZEN_WRITER_SYSTEM = `
You are a public budget audit assistant.
Your function: transform technical contract data into plain language any citizen can understand.

ABSOLUTE RULES:
- NEVER use: fraud, corruption, crime, embezzlement, illegal, criminal
- ALWAYS use: atypical pattern detected, statistical inconsistency, unusual finding identified
- ALWAYS include confidence_note with limitations of automated analysis
- Reports must be factual, objective, and based solely on provided data
`;

export async function generateCitizenReport(
  contractData: ContractWithFlags
): Promise<AnomalyReport> {
  return tracedLLMCall("citizen_report_writer", async () => {
    const response = await anthropic.messages.create({
      model: "claude-haiku-4-5-20251001",
      max_tokens: 1024,
      system: CITIZEN_WRITER_SYSTEM,
      tools: [{
        name: "AnomalyReport",
        description: "Generate structured atypicity report for public display",
        input_schema: zodToJsonSchema(AnomalyReportSchema),
      }],
      tool_choice: { type: "tool", name: "AnomalyReport" },
      messages: [{
        role: "user",
        content: `Analyze this contract and generate the report:\n\n${JSON.stringify(contractData, null, 2)}`
      }],
    });

    const toolUse = response.content.find(b => b.type === "tool_use");
    if (!toolUse || toolUse.type !== "tool_use") throw new Error("Agent did not return structured output");
    return AnomalyReportSchema.parse(toolUse.input);
  });
}
```

---

## 6. Folder Structure — V1

```
agora-global/
│
├── app/                                      # Next.js 15 App Router
│   ├── [locale]/                             # next-intl: pt-BR, en
│   │   ├── page.tsx                          # Public landing
│   │   ├── dashboard/
│   │   │   └── [municipalityId]/
│   │   │       └── page.tsx                  # Public citizen dashboard
│   │   └── (auditor)/                        # Protected — certified auditors only
│   │       └── review/
│   │           ├── page.tsx                  # HitL review queue
│   │           └── [contractId]/
│   │               └── page.tsx              # Contract detail for approval
│   └── api/
│       ├── municipalities/route.ts           # GET active municipalities
│       ├── alerts/route.ts                   # GET PUBLIC_AUDITED alerts
│       ├── review/[contractId]/route.ts      # POST approve/reject
│       └── inngest/route.ts                  # Inngest endpoint
│
├── lib/
│   ├── ai/
│   │   ├── agent.ts                          # Citizen Report Writer agent
│   │   ├── tools.ts                          # Tool definitions
│   │   ├── observe.ts                        # tracedLLMCall + logRun
│   │   └── structured.ts                     # AnomalyReportSchema (Zod)
│   └── supabase/
│       └── server.ts
│
├── inngest/
│   ├── client.ts
│   └── functions/
│       ├── trigger-batch.ts                  # Weekly cron → Python container
│       └── generate-reports.ts               # Event-driven: batch-completed → Claude
│
├── messages/                                 # next-intl
│   ├── pt-BR.json
│   └── en.json
│
├── supabase/
│   └── migrations/
│       ├── 001_municipalities.sql
│       ├── 002_contracts_and_flags.sql
│       ├── 003_auditors_and_reviews.sql
│       ├── 004_hash_chain.sql
│       └── 005_agent_runs.sql
│
├── agents/                                   # Python — separate Docker container
│   ├── requirements.txt                      # pandas, scikit-learn, httpx, supabase
│   ├── orchestrator.py                       # Entry point: receives ibge_code, runs pipeline
│   ├── ingestor.py                           # PNCP connector + OCDS normalizer
│   ├── anomaly/
│   │   ├── benford.py                        # Benford's Law test on value batches
│   │   ├── isolation_forest.py               # Isolation Forest by category
│   │   ├── cnpj_checker.py                   # Temporal risk: < 90 days old
│   │   └── price_outlier.py                  # Z-score vs PNCP historical average
│   ├── score_aggregator.py                   # Combines flags → atipicity_score
│   ├── hash_chain.py                         # SHA256 chaining
│   └── db.py                                 # supabase-py client
│
├── schemas/
│   └── ocds-release.json                     # OCDS Release Schema v1.1.5
│
├── Dockerfile.agents                          # Python agents container
├── docker-compose.yml                         # Local dev: nextjs + agents
├── ARCHITECTURE.md                            # This file
├── GOVERNANCE.md
├── CONTRIBUTING.md
├── CONSTITUTION.md
└── LICENSE                                    # AGPL-3.0
```

---

## 7. Technical Decisions

| Decision | Discarded alternative | Reason |
|---|---|---|
| Python for statistical agents | TypeScript (simple-statistics) | scikit-learn has production-proven Isolation Forest and Benford analysis; handles global contract volume without performance concerns |
| Weekly batch (cron) | On-demand (user triggers analysis) | Fixed cost = $0; avoids serverless timeouts; cached results load instantly |
| supabase-py direct write (no FastAPI in MVP) | FastAPI as intermediary | FastAPI adds an extra service to operate; unnecessary when Python only writes to the database |
| AGPL-3.0 | MIT / Apache 2.0 | AGPL prevents governments or companies from privatizing forks without contributing back; Decidim (closest benchmark) uses the same license |
| Hash Chain in PostgreSQL | Blockchain | Zero gas cost; minimal operational complexity; provides 95% of immutability guarantees needed for MVP |
| OCDS as canonical schema | Proprietary schema | Interoperability with Open Contracting Partnership; enables V3 global expansion without data normalization rework |
| next-intl from day 1 | Add i18n later | Hardcoded strings in one language are expensive technical debt; project is designed to be global from the start |
| Docker / Fly.io for Python | Vercel serverless | Public infrastructure must be self-hostable; Vercel does not natively support Python or long-running batch jobs |
| Co-evolutionary development (global + brasil in parallel) | Core-first approach | Prevents over-engineering; Brasil validates the core with real data; generic modules flow upstream organically |
| Quorum of 2 auditors to publish | Single auditor / auto-publish | Mitigates legal risk of defamation; provides resilience if one auditor is unavailable; reflects journalistic two-source standard |
| Score-based language (never accusatory) | Direct language ("suspected fraud") | Legal protection; system provides evidence, not judgment — that is the journalist's role |

---

## 8. Implementation Roadmap

See [open issues](https://github.com/CabPiz/agora-global/issues) filtered by milestone for the full sprint-by-sprint breakdown.

### Milestone 1 — MVP: The Municipal Auditor
Sprint 1: Infrastructure & Ingestion (#1–#5)
Sprint 2: Analysis Engine & Flags (#6–#11)
Sprint 3: HitL Panel & AI Agent (#12–#15, #22)
Sprint 4: Public Dashboard & Launch (#16–#21, #23)

### Milestone 2 — V2: The Federated Network
pgvector + Hybrid Search · Legislative RAG · Public FastAPI · Multi-tenant · Deliberation module

### Milestone 3 — V3: The International Hub
First international fork · Inter-Ágoras protocol · Public MCP server · Evals as CI gate

---

## Contributing

Read [CONTRIBUTING.md](CONTRIBUTING.md) to understand how to set up the local environment, coding conventions and the pull request process.

Read [GOVERNANCE.md](GOVERNANCE.md) to understand how decisions are made in the core repository and how improvements from national forks flow upstream.

---

*Ágora Global · ARCHITECTURE.md v1.0 · 2026-08-14 · AGPL-3.0*
