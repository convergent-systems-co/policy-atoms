# policy-atoms — Goals

> Governance rules as typed, versioned, catalog-driven artifacts — subjects, resources, actions, effects, and conditions — so every organization stops reinventing the same policies.

*This document is derived from `aish/ARCHITECTURE.md` (now `xdao/xdao/ARCHITECTURE.md` §The *-Atoms Catalogs). Sections marked **Generated** are pattern-based and are intended as a starting point for revision, not as decided plan.*

---

## What this catalog makes civilization-grade

OPA is the closest existing answer but isn't catalog-driven — every organization writes its own Rego, every team reinvents the same policies (least privilege, separation of duties, change windows). No canonical, shareable, versioned policy library exists.

By cataloging the primitives, `policy-atoms` turns this domain from opaque-and-ephemeral to typed, versioned, composable, machine-readable, and open — the civilization-grade properties the ecosystem requires.

## What it catalogs

### Atom types

- **`subject`** — Who is acting (user, agent, service, anonymous).
- **`resource`** — What is being acted on (file path, API endpoint, capability).
- **`action`** — What action is being taken (read, write, exec, delegate).
- **`effect`** — Permit, deny, audit-only, require-approval.
- **`condition`** — Constraints on when the rule applies (time-of-day, geographic, MFA-recency).

### Compositions: `policies`

A policy composition assembles subject + resource + action + effect + conditions into a complete authorization rule. Policy sets compose individual policies with precedence rules.

### Rule types

- **`precedence`** — When two policies conflict (one permits, one denies), which wins.
- **`conflict-resolution`** — How to merge overlapping policies (most-restrictive, most-specific, explicit-order).
- **`scope-inheritance`** — How policies cascade across resource hierarchies.

## Runtime consumers

- **olympus** — Governance panels enforce policies on Pantheon Module emissions. Drachma spending policies, safe-execution policies, secret-access policies.
- **aish** — Enterprise mode — `aish` can gate destructive commands per policy. Identity-based policy enforcement on `aish secret use`.

## Status & priority

**Current status:** `proposed`

**Priority tier:** Tier 3 — Build when supporting runtimes mature

**Trigger / activation condition:** Olympus governance maturation OR aish enterprise deployment. Whichever pulls first.

## Roadmap *(Generated — milestone shapes mirror aish's roadmap pattern; revise as actual work begins)*

### v0.1 — Bootstrap & spec acceptance

**Goal:** Schema accepted. 30 seed policies covering least-privilege, change-control, and data-residency patterns.

**Success criterion:** Olympus enforces three real policies via policy-atoms evaluation.

**Kill criterion:** Performance unacceptable in hot path — pivot to compiled policy bundles vs runtime evaluation.

**Work:**

- [ ] XAIP: policy composition schema with precedence semantics
- [ ] Define 5 atom type schemas
- [ ] Seed 30 policies across common patterns
- [ ] Integrate evaluation engine into Olympus governance
- [ ] Benchmark evaluation latency (<10ms per decision target)

### v0.2 — Adoption & expansion

**Goal:** aish enterprise mode integration.

**Work:**

- [ ] aish `policy` built-in for command gating
- [ ] Identity-policy binding via identity-atoms
- [ ] Audit trail integration with aish history-engine

### v1.0 — Operational

**Goal:** Cross-organization policy sharing — same policy enforced identically across companies that adopt the same atoms.

## Concrete atom example *(Generated — illustrative, not seed content)*

```yaml
policies/no-prod-write-after-hours/definition.yml
---
id: no-prod-write-after-hours
type: composition
version: 1.0.0
subject: { type: subject-ref, ref: subjects/any-engineer }
resource: { type: resource-ref, ref: resources/prod-databases }
action: { type: action-ref, ref: actions/write }
effect: { type: effect-ref, ref: effects/require-approval }
conditions:
  - { type: condition-ref, ref: conditions/outside-business-hours-utc }
```

## Adoption strategy *(Generated)*

Olympus governance pulls first. Enterprise aish adoption follows. Cross-org sharing emerges when adjacent companies publish their policy bundles.

## Civilization-grade property checklist

Every catalog must satisfy these before v1.0. Failing any blocks a release.

| Property | Mechanism in this catalog |
|---|---|
| Typed | JSON Schema in `schemas/` validates every atom, composition, rule |
| Versioned | Every atom has a semver `version` field; compositions reference atoms by version-pinned ID |
| Machine-readable | `exports/catalog.json` published on every release |
| Composable | Compositions reference atoms by ID; CI verifies references resolve and no circular dependencies |
| Open | Apache-2.0 licensed; LICENSE file present |
| Durable | No external dependencies for primary content (no remote image URLs, no vendor APIs in the hot path) |

## Related

- **Spec:** [atoms-spec](https://github.com/convergent-systems-co/atoms-spec) — the canonical structure every catalog conforms to
- **Tools:** [atoms-tools](https://github.com/convergent-systems-co/atoms-tools) — CLI for validate / export / bootstrap / resolve
- **Federation:** [xdao](https://github.com/convergent-systems-co/xdao) — ecosystem directory and discovery
- **Umbrella:** [atoms](https://github.com/convergent-systems-co/atoms) — every catalog as a git submodule
- **Manifest:** [`ATOMS.yml`](./ATOMS.yml) — this catalog's machine-readable manifest
- **Standard:** [`README.md`](./README.md) — catalog overview and contribution flow
