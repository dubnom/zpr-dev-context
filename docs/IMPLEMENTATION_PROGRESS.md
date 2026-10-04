# Implementation Roadmap Progress

Last updated: **2026-10-04**.

This is the tracked current-progress companion to the workspace-level public
implementation roadmap. Checked items mean implemented local capabilities,
not production readiness, published standards conformance, or clean-checkout
CI acceptance.

## Policy Language and Tools

- [x] Bind target-free `allow` and `never allow` rules to `provide` or embedded
  JSON service groups; migrate simulator bundles and example policies.
- [x] Provide Rust `zplfmt` and a matching Policy Studio Format action with
  permission indentation, group separators, and blank-line normalization.
- [x] Reject empty `with` clauses while allowing class declarations without
  `with`; report the error at the keyword.
- [ ] Reconcile RFC 15, BNF productions, and the dev reference with the
  parser's embedded JSON service scope. `deny` is not a compiler keyword.
- [ ] Define how embedded service JSON becomes signed deployment policy.
- [ ] Complete conditions, limits, route-aware evaluation, and cross-component
  compatibility acceptance. Formatter output alone is not proof of validity.

## Operator Workflows

- [x] Test candidate policies with compiler/ZPT evaluation and bounded fixtures;
  show per-rule matches, full dimension labels, and the first 100 unique members.
- [x] Provide report-only trusted-data assertions and read-only policy/assertion
  and trusted-source browsers with approved LDAP attribute views.
- [x] Add policy-picker context menus and record lifecycle operations with
  protected/archive and unsaved-edit checks.
- [x] Reuse adapter/controller log panels with per-source selection and scroll
  memory; simplify Control Room controls without changing simulator log controls.
- [x] Add independently persisted collapsible navigation and responsive UI checks.
- [ ] Complete per-record authorization, attributable audit, retention,
  backup/recovery, and certificate/key rotation before shared or production use.

## Network Isolation and Acceptance

- [ ] Decide whether simultaneous fully separate organization networks are in
  scope. Current activation switches shared runtime context.
- [ ] If adopted, isolate service processes, databases, PKI, container names,
  ports, LDAP/cache/DNS state, and logs, with explicit organization tool targets.
- [ ] Prove two-node allowed/denied forwarding and restart recovery in CI;
  an active peer link is not sufficient acceptance.
- [ ] Confirm clean-checkout browser CI and add real-backend/network acceptance.

## Verification Notes

Local regression checks cover compiler behavior, dashboard Go tests, and
desktop/mobile browser workflows. The 2026-10-04 commit checks passed 288
compiler unit tests, 11 integration tests, the compiler formatting/warnings
gate, all dashboard Go tests, Go formatting, and 80 browser tests. These
results are local snapshots, not release certification.

See [ZPL](ZPL.md), [security model](SECURITY_MODEL.md), and
[multi-node link contract](MULTI_NODE_LINK_CONTRACT.md) for design and open gaps.