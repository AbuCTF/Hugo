---
title: "Shipd"
role: "Contributor"
period: "May 2026 – Present"
location: "Remote"
image: "/images/work/eris-harness.png"
draft: false
---

Freelance contributor on Shipd, an ML challenge platform run as competitive quests.

## The quests

**Cipher**: cryptographic and security-focused ML challenges.

**Olympus**: model optimisation and efficiency benchmarks.

**Eris**: time-boxed competitions across whatever domain comes up. RNA sequencing, signal processing, optimisation.

## The harness

Eris is time-sensitive, so I built orchestration for it rather than competing by hand. The diagram above is the architecture.

An orchestrator drives the pipeline. A fleet of scout sub-agents explores problems in parallel. The point of the fleet is that most approaches to a new problem are wrong, and finding that out sequentially costs the whole time budget.

Everything else is there to keep the pipeline honest:

- **GPU orchestrator** provisioning RunPod pods, with warm-pod reuse and dataset caching so a repeat run doesn't pay setup cost again
- **Pre-submit gate**: 13 hard checks, including no-leakage validation, run before anything is submitted
- **SQLite** as the single source of truth for challenges, attempts, leaderboards and knowledge
- **Playwright connector** for the browser side of submission
- **Submission budget** enforced per UTC day, resetting at 00:00

The gate is the part that earns its keep. In a timed competition the tempting failure is to submit something that scores well locally because it leaked, burn a submission slot, and lose both the points and the time. Thirteen checks up front is cheaper than one bad submission.

Reproducibility discipline throughout: trust internal cross-validation over the free-check signal, and ensemble when re-run variance is low.
