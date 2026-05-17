# handover-docs

Engineering handover documentation for the Kaha platform.

**Start here:** [`HANDOVER.md`](HANDOVER.md) — the landing page (system map, repo directory, role-based reading path).

## Structure

| Path | Contents |
|---|---|
| `HANDOVER.md` | Master landing page |
| `GLOSSARY.md` | Shared vocabulary |
| `onboarding.md` | Day 1 / Week 1 path per role |
| `kaha-platform/service-architecture.md` | How services connect (read first) |
| `kaha-platform/<service>/` | 5 pages each: overview · architecture · data-model · decisions · runbook |
| `hotel/`, `hostel/` | Hospitality products, 5 pages each |

Standard: C4 model + arc42 + ADRs. Diagrams are Mermaid with prose fallbacks. Confluence-ready.

> No credentials are stored in this repo by design. Secrets live outside version control — request access from the platform owner.
