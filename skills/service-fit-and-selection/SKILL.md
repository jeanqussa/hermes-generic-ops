---
name: service-fit-and-selection
description: "Use when deciding whether to self-host or buy a service."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [self-hosting, vendor-selection, tco, backup, security, decision]
    category: software-development
    related_skills: [docker-test-runner, cloudflare-tunnel-subdomains]
---

# Should we run this, or buy it?

## When to Use

- "Can we self-host X?" / "Is X any good?" / "Which of these three should we use?"
- Before deploying a third-party service onto a machine you are responsible for.
- Before recommending a paid product to someone else — a client, a colleague, a non-technical
  owner who will act on the answer.

The output is a recommendation with reasons and a cost, not a feature list.

## 1. Clarify the job first

Write one sentence: *"<who> needs to <do what> <how often>, and today they <current
workaround>."* If that sentence needs three clauses, the requirement is probably two services.

Then classify:

| Question | Consequence |
|---|---|
| One user or many? | Auth, permissions, audit trail |
| Internal only or public? | Exposure model, WAF, rate limits |
| Data sensitivity (personal, financial, health)? | Encryption, retention, deletion, legal basis |
| Tolerable downtime? | Backup strategy, HA budget |
| Data must stay in a jurisdiction? | Hosting choice narrows fast |
| Who maintains it in 12 months? | The real cost driver |

If "nobody" is the answer to the last one, prefer a managed service.

## 2. Self-hosting fit

Score each; any "no" is a reason to stop and reconsider, not a detail to solve later.

- **Resources.** Documented RAM/CPU/disk requirement, plus ~30% headroom for the database and
  the build. Add up against what the host actually has (`free -h`, `nproc`, `df -h`), not what
  the box was bought with.
- **State.** Where does it keep data? A single volume, a database, object storage? Is there a
  documented backup/restore procedure, and does export work without the app (a dump you cannot
  restore elsewhere is not portable)?
- **Restore rehearsal.** A backup that has never been restored is a hypothesis. Restore into a
  throwaway container **before** committing to the service.
- **Auth.** LDAP/OIDC/SAML support, or its own users? A second password store is a support
  burden and a security surface.
- **Upgrade path.** Versioned releases with notes, or "pull and hope"? Breaking-change cadence?
  Can it run one version behind safely?
- **Exposure.** Does it need to be public, or is a tunnel + access policy enough? Does it ship
  an admin panel on the same port?
- **Maintenance reality.** Release cadence, open security issues, last commit, number of
  maintainers, whether the last two CVEs were handled in days or months.
- **Licence.** Source-available is not open source. Check the actual terms for the features you
  need and for commercial/hosted use — some licences restrict exactly the use being planned.
- **Exit.** How do you get the data out and into the alternative? Time-box it: if migration is
  unproven, the vendor owns you.

Then the honest arithmetic:

```
self-hosted cost = hardware/VM + disk + backup target + your hours/month × rate + upgrade risk
```

A weekend of setup and an hour a month is not free; write the number down so the comparison
with a subscription is at least arithmetic.

## 3. Buying fit

- **Real price.** Per seat? Per user? Per GB? Per environment? Minimum seats? Annual-only?
  Verify on the vendor's current pricing page — not a blog post, not a reseller, not a
  comparison site. Quote the date and the tier.
- **Limits.** What happens at the free-tier boundary: hard stop, throttling, or a bill? Is
  there an overage price, and is it published?
- **Data location.** Region options, subprocessors, retention after cancellation, export
  format. If personal data is involved, a processing agreement is usually required and its
  absence is a blocker, not a detail.
- **Lock-in.** Export is one thing; whether the export contains what you need to reconstruct
  the workflow in a competitor is another. Test the export before the trial ends.
- **Support and SLA.** Response times, channels, whether a human answers. For a client-facing
  deployment this is part of the product.
- **Stability.** Funding, age, customer references, changelog activity, status page history.
  A product that pivots or disappears takes the workflow with it.

## 4. Comparing candidates

Build the table before reading marketing pages, so the criteria come from the requirement
rather than from whichever product you looked at first:

| Criterion (weight) | Option A | Option B | Option C |
|---|---|---|---|
| Meets the core job | | | |
| Data location / export | | | |
| Auth integration | | | |
| 12-month cost | | | |
| Ops burden per month | | | |
| Exit cost | | | |

Run one real task in each trial — the same task, with the same data. Demos are rehearsed;
the trial is not. Note setup time, not just feature coverage.

## 5. Recommending

State: the requirement in one line, the recommendation, the two or three reasons that decided
it, the recurring cost, and what would change the answer ("if the team grows past N" or "if
data must stay on-premise"). Give the runner-up and why it lost — that is what makes the
recommendation reviewable.

For a non-technical audience, lead with what changes for them (time saved, risk removed, who
to contact when it breaks) and keep the technical reasoning available but secondary.

## Pitfalls

- **Marketing numbers are not measurements.** Published benchmarks use ideal hardware and
  cherry-picked workloads. Size from the documentation's minimums plus headroom, then measure.
- **"Free" tiers are acquisition pricing.** Check the paid tier you will actually need at the
  scale you expect in a year, including seats you will add.
- **Self-hosted is not free**, and open source is not support. The recurring cost is the
  person who upgrades it.
- **Single-maintainer projects** can be excellent and still be a risk: one person's burnout,
  one abandoned repository. Look at the issue response time, not the star count.
- **A service that needs its own database, queue and cache** has quietly become three
  services. Count containers before agreeing.
- **Backups that live on the same disk** are not backups. Check where the copy goes.
- **Ports and hostnames are decisions too** — record the port, the hostname and the compose
  project name before deploying, or the second service on the box will collide with the first.
- **The demo instance is not the product.** Trial with your own data, your own SSO, and at
  least one failure (restart a container mid-task) to see how it recovers.
- If the decision is reversible in a day, stop analysing and try it. The framework is for
  decisions that are expensive to undo.
