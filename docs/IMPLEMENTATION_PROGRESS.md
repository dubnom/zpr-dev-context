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

## Denied-Flow Backoff

- [x] Add a node-authoritative negative-decision cache with node-TOML
  `denied_flow_backoff_ms` (default 1000, zero disables, maximum 60000). Entries
  are keyed by ingress link, source/destination addresses, protocol, and
  destination port; source-port changes do not bypass an entry.
- [x] Check for an active positive visa first. On an explicit Visa Service
  denial, cache the result and complete matching binds with the same successful
  discard binding. Do not cache timeouts, VS errors, or other indeterminate
  results; in-flight requests are not reserved or coalesced.
- [x] Bound the cache globally and per ingress link, use monotonic expiries, and
  report local suppression with a separate management counter.
- [x] Preserve the no-policy-oracle contract: policy denials return success with
  a reusable blackhole stream, whose packets are silently discarded. The
  backoff interval is never exposed to the Adapter.
- [x] Test cache key scope, expiry, capacity, configuration bounds, and reuse and
  last-unbind cleanup of the blackhole stream.
- [x] Add a real-VS denial/retry scenario to the one-node integration harness.
  It asserts two denied binds with distinct source ports produce one Visa
  request and one local backoff hit. The script passes `bash -n`; runtime
  execution still requires the Linux network-namespace test environment.
- [ ] Nodes receive no policy-generation or attribute-revision invalidation
  signal today. A policy change can therefore remain locally denied until the
  configured TTL expires (at most 60 seconds); add an authenticated invalidation
  or versioned decision mechanism before relying on longer backoffs.

## Verification Notes

Local regression checks cover compiler behavior, dashboard Go tests, and
desktop/mobile browser workflows. The 2026-10-04 commit checks passed 288
compiler unit tests, 11 integration tests, the compiler formatting/warnings
gate, all dashboard Go tests, Go formatting, and 80 browser tests. These
results are local snapshots, not release certification.

See [ZPL](ZPL.md), [security model](SECURITY_MODEL.md), and
[multi-node link contract](MULTI_NODE_LINK_CONTRACT.md) for design and open gaps.