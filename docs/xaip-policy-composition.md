# XAIP: Policy Composition Schema

**Atom type:** `policy`
**Version:** 0.1
**Audience:** Policy architects, platform engineers, identity integrators

---

## 1. Purpose

A policy composition assembles individual `policy` atoms into a precedence-ordered stack. The stack is the unit of evaluation: when a policy decision is requested, all policies in the stack are evaluated in priority order and their results are combined by the precedence resolution algorithm. The composition is authoritative for access control, compliance enforcement, and organizational governance.

---

## 2. Composition Structure

A policy composition is a JSON document at the path `compositions/<slug>/v<version>/composition.json` within the policy-atoms catalog.

```json
{
  "atom_type": "policy-composition",
  "composition_id": "convergent-systems-co/platform-access",
  "version": "1",
  "display_name": "Platform Access Policy Stack",
  "policies": [
    {
      "policy_ref": "policy:convergent-systems-co/org-security-baseline",
      "priority": 100,
      "label": "org-policy"
    },
    {
      "policy_ref": "policy:convergent-systems-co/context-aware-access",
      "priority": 50,
      "label": "context-policy"
    },
    {
      "policy_ref": "policy:convergent-systems-co/user-delegated-access",
      "priority": 10,
      "label": "user-policy"
    }
  ],
  "default_decision": "deny",
  "identity_bindings": [
    {
      "identity_ref": "https://identity-atoms.convergent-systems.co/atoms/org-member/v1/atom.json",
      "min_trust_level": "authenticated",
      "policy_floor_ref": "policy:convergent-systems-co/org-security-baseline"
    }
  ],
  "metadata": {
    "catalog_ref": "policy-atoms.convergent-systems.co"
  }
}
```

### 2.1 Required fields

| Field | Type | Description |
|---|---|---|
| `composition_id` | string | Stable, namespaced identifier |
| `version` | string | Semantic version string |
| `policies` | array | Ordered list of policy entries (see §2.2) |
| `default_decision` | enum | `allow` or `deny` — decision when no policy produces an explicit result |

### 2.2 Policy entry fields

| Field | Type | Required | Description |
|---|---|---|---|
| `policy_ref` | URI | yes | Stable reference to a `policy` atom |
| `priority` | integer | yes | Evaluation order. Higher value = evaluated first. |
| `label` | string | no | Human-readable role of this policy in the stack |

---

## 3. Precedence Resolution

When an access decision is requested against a composition, the engine evaluates all policies in descending priority order and applies the following resolution rules.

### 3.1 Resolution algorithm

```
Input:  subject, resource, action, context
Output: decision (allow | deny), matched_policy_ref, reason

1. Sort policies by priority descending (highest first).
2. For each policy p in sorted order:
   a. Evaluate p against (subject, resource, action, context).
   b. If p returns explicit DENY:
      → decision = deny
      → matched_policy_ref = p.policy_ref
      → reason = p's deny reason
      → STOP. First explicit deny wins; lower-priority policies are not evaluated.
   c. If p returns explicit ALLOW:
      → record (allow, p.policy_ref) as candidate.
      → Continue evaluating remaining policies.
   d. If p returns NOT_APPLICABLE:
      → Continue.
3. After all policies evaluated:
   a. If any explicit ALLOW was recorded → decision = allow (first recorded wins).
   b. If no explicit ALLOW and no explicit DENY → decision = composition.default_decision.
```

### 3.2 Decision priority summary

| Condition | Outcome |
|---|---|
| Any policy returns explicit DENY | Deny immediately (no further evaluation) |
| At least one ALLOW, no DENY | Allow (first ALLOW wins over implicit deny) |
| All policies return NOT_APPLICABLE | Apply `default_decision` |

### 3.3 Rationale for deny-wins semantics

Explicit deny wins because policy stacks in enterprise contexts are layered: org-level policies set the security floor and may impose hard denies (e.g., MFA required, geographic restriction). A user-level ALLOW cannot override an org-level DENY — that would allow privilege escalation through delegation. The `priority` field controls which policy is evaluated first, but `explicit deny` from any evaluated policy short-circuits the stack.

---

## 4. Policy Stacking Examples

### 4.1 Three-layer stack: org + context + user

```
Priority 100: org-security-baseline
  → DENY if MFA not satisfied
  → DENY if source IP is not in corporate range
  → otherwise: NOT_APPLICABLE

Priority 50: context-aware-access
  → DENY if time-of-day outside business hours AND resource.sensitivity == "high"
  → ALLOW if device posture check passed AND resource.sensitivity == "low"
  → otherwise: NOT_APPLICABLE

Priority 10: user-delegated-access
  → ALLOW if user explicitly granted access by resource owner
  → otherwise: NOT_APPLICABLE

default_decision: deny
```

**Scenario A:** User with MFA, inside IP range, good device posture, granted access by resource owner, low-sensitivity resource.
- org-policy: NOT_APPLICABLE
- context-policy: ALLOW
- Resolution: allow (context-policy ALLOW wins, user-policy not reached)

**Scenario B:** User without MFA (any other conditions).
- org-policy: DENY (MFA not satisfied)
- Resolution: deny immediately (user-policy and context-policy not evaluated)

**Scenario C:** User with MFA, inside IP range, no device posture check, outside business hours, high-sensitivity resource.
- org-policy: NOT_APPLICABLE
- context-policy: DENY (outside hours + high sensitivity)
- Resolution: deny (context-policy DENY wins)

### 4.2 Composition with a policy floor

A policy floor ensures that a specific policy is always present in the stack for a given identity, regardless of what other compositions may be applied:

```json
{
  "identity_bindings": [
    {
      "identity_ref": "https://identity-atoms.convergent-systems.co/atoms/org-member/v1/atom.json",
      "min_trust_level": "authenticated",
      "policy_floor_ref": "policy:convergent-systems-co/org-security-baseline"
    }
  ]
}
```

The floor policy is injected at priority `MAX_INT` (evaluated first) regardless of the composition's own priority ordering. It cannot be overridden or removed by a user-delegated policy.

---

## 5. Integration with identity-atoms

Policy compositions bind obligations to identities through `identity_bindings`. This is the mechanism by which policy floors are attached to specific identity trust levels.

### 5.1 Identity binding object fields

| Field | Type | Required | Description |
|---|---|---|---|
| `identity_ref` | URI | yes | Fully-qualified reference to an identity atom |
| `min_trust_level` | enum | yes | Minimum trust level the identity must present: `anonymous`, `authenticated`, `signed`, `verified` |
| `policy_floor_ref` | URI | yes | Policy atom that is always included at maximum priority for this identity |

### 5.2 Trust level gating

When an identity presents at a trust level below `min_trust_level`, the composition rejects the request before policy evaluation begins:

```
Presented trust_level: authenticated
Required min_trust_level: signed
→ Pre-evaluation rejection: "Trust level insufficient. Escalation required."
```

The escalation flow is governed by the identity composition (see identity-atoms `xaip-identity-composition.md §4.3`).

### 5.3 Cross-catalog reference pattern

All identity references in policy compositions use fully-qualified HTTPS URIs:

```
https://identity-atoms.convergent-systems.co/atoms/<slug>/v<version>/atom.json
```

---

## 6. Composition Conventions

| Convention | Value |
|---|---|
| Composition path | `compositions/<slug>/v<version>/composition.json` |
| Policy atom path | `atoms/policies/<slug>/v<version>/atom.json` |
| Priority range | `1` (lowest) – `MAX_INT` (reserved for floors). User-defined range: `1–999`. |
| Default decision | Always declared explicitly; never rely on absent-field default |

---

## 7. Related Atoms and Docs

- `policy` atom — individual access rule with conditions, effects, and resource scope
- `policy-condition` atom — boolean predicate evaluated against subject/resource/context
- `policy-effect` atom — `allow` or `deny` with optional reason and remediation hint
- identity-atoms: `xaip-identity-composition.md` — trust levels and escalation flows
- compliance-atoms: `xaip-framework-composition.md` — compliance obligations implemented as policy floors
- service-atoms: `xaip-service-composition.md` — auth schemes that reference policy compositions
