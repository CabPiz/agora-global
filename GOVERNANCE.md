# Governance — Ágora Global

This document describes how decisions are made in the core repository, how improvements from national forks flow upstream, and how contributors can become committers over time.

---

## Maintainers

The project is currently maintained by its founder, [@CabPiz](https://github.com/CabPiz). All PRs to `main` require approval from at least one maintainer.

As the contributor community grows, committer rights are granted to individuals who have demonstrated sustained, high-quality contributions over time (see [Becoming a Committer](#becoming-a-committer)).

---

## Decision-Making

### Small changes (bug fixes, documentation, minor improvements)
Open a PR directly. No prior discussion required. A maintainer reviews and merges.

### Significant changes (new algorithms, schema changes, new AI agents, architecture decisions)
Open an issue first and label it `RFC` (Request for Comments). Describe the problem, your proposed solution, and the trade-offs. Allow at least 5 business days for community feedback before opening a PR. The maintainer has final decision authority during the early project phase.

### Breaking changes (changes that affect national forks or the OCDS schema contract)
Require an RFC with a minimum 14-day comment period and explicit sign-off from the maintainer. The change must be documented in `ARCHITECTURE.md` before the PR is merged.

---

## Upstream-First Policy

Ágora Global is the upstream core. National implementations (e.g., `agora-brasil`, `agora-colombia`) are downstream forks.

**Rule:** Any improvement that is not country-specific must be submitted to this repository and merged here before it can be used by any national fork.

**Why this matters:** Without this rule, forks silently diverge. A bug fix in `agora-brasil` would never reach `agora-colombia`. A security patch in the core would need to be manually applied to every fork. Upstream-first keeps the ecosystem coherent without central coordination overhead.

**How to apply it:**
1. Identify whether your change is country-specific (data source connectors, local regulatory rules, language assets) or generic (detection algorithms, UI components, database schema, AI agents).
2. If generic: open the PR against `CabPiz/agora-global` first.
3. After it is merged upstream, the national fork can pull from the core via the standard fork sync flow.
4. If you are unsure: open an issue here and ask. Default to upstream.

---

## Branch Strategy

| Branch | Purpose |
|---|---|
| `main` | Stable. Protected. Merged via squash-and-merge. |
| `feat/*` | Feature branches. Short-lived. PR against `main`. |
| `fix/*` | Bug fix branches. PR against `main`. |
| `release/*` | Release preparation. Created by maintainers only. |

Direct pushes to `main` are disabled. All changes go through a PR with at least one approval.

---

## National Forks

A national fork is a repository that implements Ágora Global for a specific country's data sources and regulatory context. Current forks:

| Fork | Country | Status |
|---|---|---|
| [agora-brasil](https://github.com/CabPiz/agora-brasil) | Brazil | Planned |

To start a new national fork:
1. Open an issue in this repository labeled `national-fork` describing the country, the primary open contracting data source, and the team.
2. The maintainer will provide guidance on the recommended fork structure.
3. The fork must inherit the AGPL-3.0 license and reference this repository as the upstream core in its README.

---

## Becoming a Committer

Committer rights are granted to contributors who have:
- Merged at least 5 substantive PRs (not documentation-only)
- Demonstrated understanding of the system architecture and the upstream-first policy
- Been active in the community for at least 3 months

To be nominated, open an issue labeled `committer-nomination`. The existing maintainers discuss and decide within 14 days.

Committer rights can be revoked for sustained inactivity (12+ months without contribution) or for conduct that violates the project's values of civic responsibility and public accountability.

---

## Releases

Ágora Global follows [Semantic Versioning](https://semver.org/): `MAJOR.MINOR.PATCH`.

- **PATCH**: bug fixes and security patches. Released as needed.
- **MINOR**: new features, new detection algorithms, new UI components. Aligned with milestone completions (M1, M2, M3).
- **MAJOR**: breaking changes to the OCDS schema contract, the fork API, or the database schema. Requires community RFC.

Release notes are published as GitHub Releases and include a migration guide for national forks when the schema changes.

---

## Code of Conduct

Contributors are expected to act with the same civic responsibility the project promotes. Discussions must remain constructive and focused on the public good. Harassment, bad faith, or coordinated attempts to discredit audit findings through the contribution process will result in immediate removal.

---

*Ágora Global · GOVERNANCE.md · AGPL-3.0*
