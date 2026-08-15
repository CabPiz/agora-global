# Ágora Global

**Open-source civic infrastructure for public budget transparency.**

Ágora Global detects statistical anomalies in public procurement contracts and surfaces them for review by certified citizen auditors — journalists, NGOs, and observatories. No algorithm ever accuses anyone of anything. The system emits an **Atypicity Score** (0.000–1.000) backed by documented mathematical evidence. Humans publish the findings.

> "Give citizens the same tools governments use to hide misuse of public funds."

---

## How it works

```
Public procurement data (PNCP / OCDS standard)
        ↓
Python statistical agents
  · Benford's Law test
  · Isolation Forest
  · CNPJ temporal risk (supplier age < 90 days)
  · Price Z-score vs. historical average
        ↓
Atypicity Score aggregated per contract
        ↓
Claude AI writes a plain-language Citizen Report
  (validated by Zod schema — score language only, never accusations)
        ↓
Human-in-the-Loop: 2 certified auditors must approve before publication
        ↓
Public dashboard — audited findings visible to any citizen
```

The system is **fully parametrized**: any municipality, any country. The pilot is São Paulo-SP, Brazil (IBGE 3550308) — the richest open contracting dataset in the country.

---

## Principles

1. **Score, never accusation.** The system never makes statements about intent or illegality. It provides evidence; journalists and auditors draw conclusions.

2. **Human-in-the-Loop is mandatory.** No alert is ever published automatically. The flow `DRAFT → UNDER_REVIEW → PUBLIC_AUDITED / REJECTED` with a quorum of 2 independent validators is inviolable.

3. **Nothing hardcoded.** Municipality, threshold, active algorithms — all configurable. The code doesn't know "São Paulo" exists.

4. **Upstream-first.** Every generic improvement produced in a national fork flows back to this core before any other fork uses it.

5. **Self-hostable.** Any government, NGO, or journalist organization can run a full instance on their own infrastructure with `docker compose up`.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | Next.js 15 (App Router) + TypeScript + Tailwind + next-intl |
| Statistical agents | Python 3.11 + pandas + scikit-learn |
| Database | Supabase (PostgreSQL) + pgvector (V2) |
| Async queues | Inngest (weekly batch cron) |
| AI | Anthropic SDK — claude-haiku (Citizen Report Writer, Structured Outputs) |
| Deploy | Vercel (frontend) + Fly.io (Python container) |
| License | AGPL-3.0 |

---

## Status

**V1 — The Municipal Auditor** is currently in development.

| Component | Status |
|---|---|
| PNCP ingestor + OCDS normalizer | 🔲 Planned — [issue #1](https://github.com/CabPiz/agora-global/issues/1) |
| Statistical analysis engine | 🔲 Planned — [issue #6](https://github.com/CabPiz/agora-global/issues/6) |
| Citizen Report Writer (Claude AI) | 🔲 Planned — [issue #12](https://github.com/CabPiz/agora-global/issues/12) |
| HitL auditor panel | 🔲 Planned — [issue #13](https://github.com/CabPiz/agora-global/issues/13) |
| Public citizen dashboard | 🔲 Planned — [issue #16](https://github.com/CabPiz/agora-global/issues/16) |

See [open issues](https://github.com/CabPiz/agora-global/issues) for the full sprint breakdown.

---

## Join the project

Ágora Global is built by a community of developers, journalists, data scientists, and civic organizations. There are three ways to contribute:

### For developers
Read [CONTRIBUTING.md](CONTRIBUTING.md) to set up the local environment and open your first PR. The system is split between a Next.js frontend and Python statistical agents — contributions are welcome in both. Good first issues are labeled [`good first issue`](https://github.com/CabPiz/agora-global/issues?q=label%3A%22good+first+issue%22).

### For journalists and auditors
We are building a **Validating Community of Certified Citizen Auditors**. If you are a journalist, observatory, or NGO working on public accountability and want to become a certified auditor, open an issue with the label [`area: community`](https://github.com/CabPiz/agora-global/issues?q=label%3A%22area%3A+community%22) or contact us at [cab.pizarro@gmail.com](mailto:cab.pizarro@gmail.com).

Target organizations for V1 launch: Agência Pública, The Intercept Brasil, Transparência Brasil, Contas Abertas, Poder360.

### For national implementations
Want to run Ágora Global for your country? The architecture supports country-specific forks (`agora-brasil`, `agora-colombia`, etc.) via the **upstream-first governance model**. Read [ARCHITECTURE.md](ARCHITECTURE.md) § Upstream-First Policy before starting.

---

## Getting Started (developers)

```bash
git clone https://github.com/CabPiz/agora-global.git
cd agora-global
npm install
supabase start
supabase db push
npm run dev
```

Full setup instructions: [CONTRIBUTING.md → Local Setup](CONTRIBUTING.md#local-setup).

---

## License

AGPL-3.0 — see [LICENSE](LICENSE).

The AGPL-3.0 license means: you can use, modify, and distribute this software freely, including for government deployments, but any modifications must also be released under AGPL-3.0. This prevents governments or companies from privatizing forks without contributing improvements back to the public.

---

*Ágora Global — civic infrastructure for the next century of democracies.*
