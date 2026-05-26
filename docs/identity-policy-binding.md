# Identity-Policy Binding

## What it is

Identity-policy binding connects an identity atom (from identity-atoms) to a policy atom (from policy-atoms), creating a typed assertion: "this identity is subject to this policy floor."

## Binding schema

A binding is expressed as a composition that references atoms from both catalogs:

```json
{
  "identity_ref": "identity-atoms/identities/convergent-systems-polliard",
  "policy_floor_ref": "policy-atoms/policies/security-baseline",
  "trust_level_required": "signed",
  "effective_from": "2026-01-01T00:00:00Z",
  "scope": "all-catalogs"
}
```

## How the policy broker uses it

At dispatch time, the Olympus policy broker:
1. Resolves the actor's identity from the session context
2. Looks up any identity-policy bindings for that identity
3. Applies the policy floor from the binding (in addition to any context-specific policy)
4. Denies dispatch if the actor's trust_level is below the binding's `trust_level_required`

## Cross-catalog reference format

Identity refs use the `identity-atoms://` URI scheme:
```
identity-atoms://identities/convergent-systems-polliard@1.0.0
```

Policy refs use the `policy-atoms://` URI scheme:
```
policy-atoms://policies/security-baseline@1.0.0
```

## Policy stacking

When multiple bindings apply (global + context-specific), all policies are evaluated and the strictest wins. A `deny` from any policy blocks dispatch regardless of other policy verdicts.

## Integration with identity-atoms

The `claim-type` atom `github-org-membership` can serve as the trust-level gate:
- `trust_level: authenticated` → requires GitHub OAuth claim
- `trust_level: signed` → requires signed key claim from key-atoms

This allows declarative trust escalation without hardcoded identity checks.
