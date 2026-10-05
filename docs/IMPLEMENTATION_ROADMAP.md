# ZPR Public Implementation Roadmap

This checklist describes work needed to complete the **public ZPR reference
implementation**, based on the checked-out `org-zpr` repositories and their
public documentation. It is a durable working plan, not a claim that every
design feature is already specified or production-ready.

Last updated: **2026-10-05**. Checked items below distinguish implemented local
capabilities from still-open clean-checkout, CI, and production acceptance work.

The tracked current-progress companion is
[`IMPLEMENTATION_PROGRESS.md`](IMPLEMENTATION_PROGRESS.md).
This is the canonical roadmap, maintained in the `zpr-dev-context` repository
so changes can be committed and pushed with the shared documentation.

## Definition of Done

Call the public implementation complete when:

- Its supported behavior is specified by published ZPR documentation, or by
  an explicitly adopted public implementation specification.
- The compiler, policy evaluator, Visa Service, nodes, docks, and adapters
  agree on the same policy and wire semantics.
- Security properties claimed for the public implementation have automated
  tests and are verified on a running multi-node network.
- A clean checkout can build, test, configure, run, upgrade, and diagnose the
  supported deployment using documented steps.
- Known limitations and intentionally deferred features are clearly listed.

The public docs report working nodes/adapters, Noise-secured links, A2A packet
integrity, bootstrap authentication, visa issuance/distribution/revocation,
the ZPL compiler, and file-backed trusted attributes. Treat that as the
starting status; verify it against the current code and CI before relying on it.

## Test Rig Strategy

Build test capability incrementally with the implementation; do not block all
feature work on completing a comprehensive test rig first. Establish a small,
reliable baseline before the first behavioral change, then make tests for the
changed behavior part of that step's acceptance criteria.

- Before implementation, record clean-checkout build/test results and exact
  commands for the affected repositories.
- Keep unit and component tests close to the implementation, then add
  cross-process integration coverage for behavior that crosses repository or
  wire-protocol boundaries.
- Extend the shared test harness only when a roadmap item needs it. Add
  two-node integration tests for the implemented node-to-node peering path;
  their current absence is an acceptance gap, not a reason to postpone
  independent compiler, evaluator, or service work.
- The draft standard-substrate peer-link behavior and RFC-17 decisions are
  tracked in [`MULTI_NODE_LINK_CONTRACT.md`](MULTI_NODE_LINK_CONTRACT.md);
  finalize it against the authoritative wire specification before coding new
  ZDP messages.
- Run tests in the narrowest layer that can falsify the change, and retain
  broader regression suites in CI. Record environmental constraints such as
  Linux network namespaces, privileges, and external services.
- A feature is not complete because its test was added: required checks must
  run in CI, and security properties must include adversarial cases.

## Phase 0: Set Scope and Make the Baseline Reproducible

- [ ] Define the target: public-RFC-conformant reference implementation,
  production-hardened deployment, or both. Keep these acceptance targets
  separate.
- [ ] Inventory each published requirement against implementation, tests, and
  docs. Record gaps with a repository, owner, dependency, and acceptance test.
- [ ] For design areas that cite unpublished/internal RFCs, obtain a public
  specification or explicitly defer the feature. Do not infer private protocol
  details from summaries in public context files.
- [ ] Pin and document the supported Rust toolchain, system packages, schema
  compiler, database, and optional tools. Use the shared dev environment as the
  canonical dependency source.
- [ ] Initialize all required Git submodules and validate the standard
  multi-repository layout.
- [ ] Run each repository's build, unit tests, formatting, and warning checks
  from a clean checkout; capture current failures before changing code.
- [ ] Record existing integration coverage and limitations: core CI runs Linux
  one-node IPv6 and capture tests; the A2A tampering script is not currently
  wired into CI; node-to-node peering is implemented, but reproducible two-node
  integration and end-to-end route/visa validation remain open.
- [ ] Check whether core integration CI's pinned Visa Service release is
  compatible with the component revisions under test. Define a reproducible
  coordinated-revision strategy for cross-repository changes.
- [ ] Make the existing A2A tampering test reproducible in CI, or document and
  resolve its prerequisites before relying on it as a security regression
  test.
- [ ] Run the single-node and multi-node demos and save reproducible startup,
  health-check, and teardown commands. Treat the multi-node demo as a
  capability check: distinguish an active peer link from verified cross-node
  policy enforcement, forwarding, and failure recovery.
- [x] Define a reusable generic ZPR base-install manifest covering required
  packages, toolchain, built artifacts, base trusted services, and boot agents.
- [x] Define a scenario manifest that extends the generic base and supplies
  simulation-specific services without changing the installation contract.
- [x] Add a dependency-aware local stack lifecycle with managed binaries, PIDs,
  logs, and protocol-aware readiness checks.
- [x] Boot a Linux one-node simulator with authenticated actors and services;
  Control Room reads live registrations through the Visa Service Admin API.
- [x] Recover the local one-node rig after a control-plane restart: renew the
  expired node certificate under the existing CA and pinned key, restore
  mixed-prefix DNS routes and the private machine substrate/control relay,
  and reconnect the four running machine controllers and observability.
- [ ] Turn that recovery into an automated clean-boot/restart test, including
  certificate expiry, namespace recreation, controller heartbeats, DNS
  reachability, and preservation of company workspaces and DNS zone data.
- [ ] Run the manifest-driven extended boot in its Linux container and verify
  that **every** declared service receives a live Visa Service registration
  and is visible in Control Room; automate this assertion in CI.

**Exit criteria:** a clean environment can build the supported repositories,
run their tests, start a supported demo ZPRnet, and reproduce the recorded
baseline. Cross-repository CI inputs and currently runnable security tests are
understood; uncovered behavior is tracked rather than assumed tested.

The generic base-install and simulation-overlay manifests are in use. The
single-node Linux stack has run with live actor/service registrations; complete
manifest-to-registration coverage and clean-checkout CI remain open.

## Phase 1: Complete Policy Semantics and Enforcement

**Owners:** `zpr-compiler`, `zpr-visaservice/libeval`, `zpr-policy`,
`zpr-common`.

- [x] Parse `provide <class> at <dns-name> over <TCP|UDP> <port>` as a signed
  service identity and transport scope; keep legacy `.zplc` service tables
  available for migration.
- [x] Accept `service <class> as json <object>` and validate the class in the
  compiler's parsed policy; Northstar demo policies embed their former JSON
  catalog records instead of maintaining a separate service tree.
- [x] Bind target-free `allow` and `never allow` rules to the immediately
  preceding `provide` or embedded JSON service declaration; migrate simulator
  policy layers and demo fixtures, preserving organization-specific grants.
- [x] Add a Rust `zplfmt` command and matching Policy Studio Format action:
  indent permission lines, separate service groups and completed rule blocks,
  and normalize repeated blank lines. Formatting is not compiler validation.
- [x] Reject a written `with` clause with no attributes and report the error
  at `with`; keep attribute-free class declarations without `with` valid.
- [ ] Reconcile RFC 15, the BNF group productions, and the dev reference with
  embedded JSON service rule scope. Document formatter style separately from
  grammar; `deny` is not currently a compiler keyword (`never allow` is).
- [ ] Define how JSON service records become signed policy, or replace them
  with native ZPL clauses. Today the compiler retains the JSON only in memory;
  it does not emit those fields in the signed binary.
- [ ] Specify and implement ZPL conditions, circumstances, and quantitative
  limits, including bandwidth, connection, and transferred-data limits.
- [ ] Specify and implement policy assertions and their validation behavior;
  document how they differ from existing `never` denials.
- [ ] Complete route-aware evaluation of `over` constraints. Evaluate candidate
  policy matches against actual path/link attributes before granting a visa.
- [ ] Preserve deterministic decisions: evaluate denials before allows, use a
  stable documented ordering, and fail closed when no allowed route exists.
- [ ] Carry new semantics through the compiler's signed binary format, schema,
  shared types, Visa Service version checks, and diagnostic tooling.
- [ ] Add compiler and evaluator tests for boundary values, conflicting rules,
  attribute changes, expired attributes, route changes, no-route cases, and
  malformed or unsupported policy versions.
- [ ] Add compatibility fixtures proving old policy binaries are either safely
  supported or rejected with an actionable error.

**Exit criteria:** policy features accepted by the language are represented in
the binary policy, evaluated consistently, route-checked, and covered by
deterministic tests.

## Phase 2: Close Packet-Security Gaps

**Owners:** `zpr-core`, `zpr-common`, `zpr-visaservice`.

- [ ] Implement receiver-side anti-replay enforcement for each security
  association, using a defined replay window and sequence-number lifetime.
- [ ] Define rollover and rekey behavior before sequence wrap; test duplicate,
  delayed, reordered, concurrent, and post-restart packet cases.
- [ ] Decide the supported A2A confidentiality requirement. If required,
  specify cipher, nonce construction, key derivation, key separation, rotation,
  failure handling, and compatibility with existing integrity-only flows.
- [ ] Review the full ZDP security lifecycle: handshake identity validation,
  certificate/key expiry, SA establishment and retirement, key distribution,
  revocation, and restart recovery.
- [ ] Add protocol parser fuzzing and malformed-packet tests, plus adversarial
  tests for tampering, forgery, replay, unauthorized streams, stale visas, and
  compromised intermediate nodes.
- [ ] Ensure security failures fail closed, do not leak policy or endpoint
  information, and produce useful bounded audit events.

**Exit criteria:** every security guarantee advertised for a deployment has
an implementation-level test, including tests across actual node/adapter
processes rather than only isolated functions.

## Phase 3: Complete Identity, Attribute, and Administration Paths

- [x] Add a separate report-only trusted-data assertion subsystem: one global
  Control Room editor/store, bounded live LDAP membership reads, cardinality,
  exact-one and mutual-exclusion syntax, manual draft evaluation, and opt-in
  single-flight periodic checks. Unknown groups and source failures are errors.
  This does not implement ZPL permission/assertion consistency checks or block
  policy/organization activation. See
  [`ASSERTIONS.md`](../../zpr-visaservice/zpr-dashboard/ASSERTIONS.md) in the
  sibling workspace checkout for v1 scope.
- [x] Extend trusted-data assertions with safe LDAP attribute allowlists,
  presence/absence, exact string and integer comparisons, allowed-value lists,
  and multi-value containment for people/scoped people and groups. Catalogs
  expose coverage counts, not attribute values; unknown/excluded names and
  invalid numeric data report errors.

**Owners:** `zpr-visaservice`, `zpr-core`, `zpr-vsapi`, `zpr-dashboard`.

- [x] Run the local TLS/mTLS HTTP trusted-service and LDAP-backed attribute
  provider alongside file-backed sources in the simulator.
- [ ] Specify and complete the supported networked trusted-service connector
  contracts beyond the local implementation; retain the provenance, refresh,
  failure, interoperability, and CI acceptance requirements below.
- [x] Run the Policy Repository REST API as a separate, mutually authenticated
  Policy-Service process behind Control-Service and the loopback Control Room.
  Keep its category/record/revision model in SQLite; demo import is idempotent
  and does not overwrite previously edited records.
- [x] Add candidate policy testing using the compiler and ZPT evaluator with
  bounded actor/service fixtures, per-rule match counts, and matching-identity
  dialogs with full dimension labels and the first 100 unique members.
  Testing does not save, stage, or activate the candidate.
- [x] Add read-only policy/assertion and trusted-source browsers, including
  approved LDAP attribute views; these do not edit directory data or grant
  production record authorization.
- [x] Add policy-picker context-menu workflows and repository record lifecycle
  operations with protected-record, archived-record, and unsaved-edit checks.
- [ ] Specify and deploy the Policy Repository as a ZPR trusted-service peer
  (not an attribute provider). Add per-record authorization, attributable
  audit events, a replaceable storage contract, and recovery/rotation before
  enabling shared or production use.
- [ ] Specify authentication, signed responses, attribute provenance, cache
  behavior, expiry, refresh, push invalidation, timeout, retry, and failure
  semantics for each connector.
- [ ] Test joins and active flows when attributes change, expire, become
  unreachable, or are revoked; ensure affected visas are re-evaluated and
  revoked promptly.
- [ ] Define a public admin authorization model, including separation of
  read-only and policy-changing actions, key rotation, audit records, and
  recovery procedures.
- [ ] If multi-party approval is a required guarantee, implement k-of-n
  concurrence with replay-resistant signed approvals, distinct-admin checks,
  expiry, cancellation, and fail-closed behavior. The current API is documented
  as single-key authorization.
- [ ] Test admin operations end to end: policy install/test/rollback, actor and
  service removal, visa revocation, and authorization failures.
- [x] Make simulation-declared services use the normal actor registration path
  by sending `zpr.services` claims during `authorize_connect`; keep the
  dashboard read-only with respect to topology registration.
- [ ] Define an authenticated HTTPS access gateway for browsers outside ZPR
  as a public deployment contract before publishing administrative UIs.
  Production rollout remains deferred; current application listeners are
  loopback-only.
- [x] Implement an optional host-side browser mTLS gateway for Control Room
  and simulator with a dedicated browser-client CA and loopback upstreams.
  It is separate from the ZPR internet-gateway actor and is not enabled in the
  current local rig.
- [ ] Add gateway roles, attributable audit, certificate revocation/rotation,
  and authenticated LDAP/OpenObserve routes before shared or public use.

**Exit criteria:** live identity and attribute changes reliably affect
admission and existing visas; privileged changes are authenticated, auditable,
and protected to the level claimed by the security model.

## Phase 4: Make Configuration Changes and State Resilient

**Owners:** `zpr-visaservice`, `zpr-core`, shared APIs/schemas.

**Multi-node snapshot (2026-10-02):** the separate Docker demo has `node0` and
`node1` running, with their peer link reported **Active on both ends**. Alice's
adapter is still **Helloing**, so the demo is not fully healthy. The main
simulator remains one synchronized forwarding node with no inter-node links.
This snapshot does not establish cross-node traffic or recovery acceptance.

- [ ] Specify versioned configuration identity and implement concurrent
  configuration transition: validate/test, activate, stop admitting new flows
  on the old configuration, drain in-flight flows, then retire old state.
- [ ] Test interrupted installs, rollback, node disconnect/reconnect, partial
  distribution, stale messages, and process/database restarts.
- [ ] Define the availability target for the Visa Service. The current public
  notes describe one active instance protected by a database lock.
- [ ] If high availability is in scope, design state ownership, leader
  election, replication, split-brain prevention, recovery, and behavior during
  loss of the database or control-plane links before implementing replicas.
- [ ] Decide whether automatic topology generation is in scope. Publish its
  inputs, constraints, failure behavior, and generated configuration format
  before replacing the current configured topology and live route selection.
- [ ] Document and test node/link failure handling and the expected service
  behavior during partitions and recovery.
- [ ] Replace shared active-organization switching with independently scoped
  runtime stacks if simultaneous organizations are adopted: isolate service
  processes, databases, certificates, containers, ports, directory/cache/DNS
  state, and logs; require explicit organization targeting in operator tools.
  This remains design work, not an implemented isolation guarantee.

**Multi-node implementation and acceptance:** follow the current code boundary
and acceptance cases in
[`MULTI_NODE_LINK_CONTRACT.md`](MULTI_NODE_LINK_CONTRACT.md).

- [x] Implement configured, authenticated NodeToNode peers with pinned Noise
  certificates, initiator selection, bidirectional Hello, and Echo liveness.
- [x] Add link-status reporting and gate persisted router edges on fresh active
  reports from both node endpoints.
- [ ] Add reproducible two-node integration tests proving allowed cross-node
  flows succeed and denied flows remain denied, with visa/forwarding enforcement
  at both hops; an Active link alone is not sufficient.
- [ ] Complete node-to-node unbind handling and verify that peer removal or
  link loss invalidates affected routing and forwarding state without leaving
  usable stale visas or stream entries.
- [ ] Verify peer-link restoration and restart of either node recover without
  restarting unrelated nodes or the Visa Service; cover stale status reports
  and missing, untrusted, or mismatched peer certificates/keys.
- [ ] Add the two-node acceptance suite to cross-repository CI, then extend the
  main simulator with a second forwarding node and a policy-declared peering.
- [ ] Resolve the remaining RFC 17 authority, termination, and restart wording
  dependencies before claiming full standards conformance or production readiness.

**Exit criteria:** configuration deployment is atomic from the operator's
perspective, old flows are handled according to documented rules, and any
availability claims are demonstrated by fault-injection tests.

## Phase 5: End-to-End Verification and Release Readiness

**Owners:** all implementation repositories, `zpr-demo`, `zpr-dev-tools`.

- [x] Add a repeatable offline dashboard browser suite for implemented behavior:
  regressions on desktop and mobile cover log radios, follow/scroll memory,
  errors and polling, ANSI/HTML safety, policy suggestions, service colors and
  gateway clouds, visa refresh/DNS names, LDAP popup/collapse behavior, and
  explicit organization activation approval, cancellation, and stale-state checks.
  Use real static assets, CSP, and fixture APIs without live-stack credentials;
  configure a Chromium CI job with failure screenshots/traces.
- [ ] Confirm the browser job passes in a clean GitHub CI checkout, and add
  real-backend/network acceptance tests separately; mocked browser responses
  are not proof of ZPR forwarding, authentication, or multi-node recovery.
- [x] Add per-machine maximize/restore controls to simulator logs and serve the
  updated view on the standard local simulator port without restarting the machine
  containers.
- [x] Add company-scoped, versioned scenario and LDAP-directory workspaces,
  published scenario execution, folders, and explicit organization activation.
  Scenarios displays the active company as text; switching belongs on the
  Organizations page and does not itself deploy policy or reseed LDAP.
- [x] Implement opt-in per-machine scenario timelines with stable step IDs,
  explicit cross-lane dependencies, an ordered Shared lane, cancellation, and
  cleanup after workers stop. Keep existing scenarios sequential by default.
- [x] Display machine tracks in a dedicated horizontally scrollable frame
  rather than wrapping lanes into multiple rows.
- [x] Provide reusable LDAP organization graphs for profile department
  hierarchy, people, and group membership, with search, pan/zoom, branch
  collapse/expand, keyboard selection, and component-info popups. This is a
  profile visualization, not a live LDAP query or LDIF parser.
- [x] Add a fleet-wide Machine logs page with controller, adapter, and workload
  sources; bounded polling, redaction, pause/filter/follow controls, independent
  source selection and scroll memory, ANSI colors, and white-on-black running
  feeds. Stop polling when the page is inactive; this is not a durable archive.
- [x] Simplify Control Room adapter logs: reuse horizontal panels for adapter
  and controller views, remember per-source scroll/follow state, and remove
  repeated panel headings, the Control Room follow checkbox, and fleet counts.
  Keep simulator workload-log controls independent.
- [x] Add separately persisted collapsible navigation for Control Room and
  Simulator and remove redundant explanatory blocks from trusted-source views;
  exercise responsive layouts with desktop/mobile browser checks.
- [x] Add a sortable Activity view for recent Visa Service grants and denials,
  counted Control Room status tabs, and map-only summary metrics. Preserve
  rendered topology bounds during Fit and animate retained components unless
  reduced motion is requested.
- [x] Use device terminology in the Simulator inventory, group unfiled scenarios
  explicitly, and allow clearing only completed/cancelled run history.
- [ ] Resolve current full-desktop browser regressions in assertion-record
  visibility, picker focus/history visibility, and adapter-log retention; keep
  clean-checkout browser CI acceptance open until the complete suite passes.
- [ ] Add a consistent, accessible help system across Control Room and
  Simulator. Provide page- and task-specific guidance for common workflows,
  explain consequential actions such as policy staging and organization
  switching before confirmation, and surface recovery steps for expected
  service/runtime failures. Link concise in-app guidance to the authoritative
  component docs; support keyboard and screen-reader use, avoid blocking normal
  work, and test help visibility/content on desktop and mobile.
- [x] Show an adapter's complete current unexpired visa list from its map
  inspector, beyond the recent-ten snapshot limit. Distinguish empty lists
  from failures; show flow, protocol, expiry, node, and policy. These are
  grants involving the address, not adapter installation acknowledgements.
- [x] Provide a ZPR-hosted BIND 9 DNS profile with a policy-declared DNS
  service, TSIG-protected Visa Service publication, and a separate restricted
  DNS statistics endpoint. The simulator has a local stack profile and a
  same-origin Control Room DNS statistics proxy.
- [x] Publish forward A/AAAA and reverse PTR records for every registered,
  unexpired adapter under `<common-name>.adapters.svc.zpr.`. Reserve that
  namespace, use safe address-derived fallbacks, and keep service aliases
  policy-gated. Restrict TSIG updates to forward records and configured PTR
  zones; journal ownership for restart-safe withdrawal and periodic retry.
  Live forward/PTR coverage was verified for all 14 registered adapters.
- [ ] Automate adapter DNS join, leave, expiry, address reuse, publisher/DNS
  outage, and restart tests against BIND in CI. Retain the distinction between
  authoritative withdrawal and resolver caches that survive until TTL expiry.
- [x] Build an OpenObserve image and ZPL deployment profile for a dedicated
  ZPR observability service; specify OTLP/HTTP logs and metrics, authenticated
  queries, publisher/reader identities, and private storage.
- [x] Provision observability and telemetry-publisher adapters with the combined
  local policy; run OpenObserve with retained private storage and a collector
  exporting Visa Service counters, denials, and redacted process logs through
  authenticated OTLP/HTTP. The recovered local collector exports 16 metric
  series, and the logger UI is reachable through its local relay.
- [ ] Add observability provisioning/recovery and ingest/query assertions to CI;
  define production retention, roles, credential rotation, and durable audit
  semantics. The denial source remains a bounded in-memory window, and policy
  signal delivery is not implemented by this collector.
- [ ] Maintain an end-to-end test network covering endpoint join, allowed and
  denied communication, multi-hop forwarding, route constraints, visa expiry,
  renewal, policy changes, revocation, and restart/reconnect.
- [ ] Add negative and fault tests for unavailable trusted services, invalid
  credentials, stale policy, malformed traffic, missing routes, and control-
  plane interruption.
- [ ] Add load and latency tests for visa issuance, route selection, per-packet
  enforcement, large attribute sets, and revocation fan-out; publish limits.
- [x] Add Velocity Labs and a two-machine, single-forwarder benchmark scenario
  with 100 warmed HTTP round-trip samples (p50/p95/p99) and a bounded 16 MiB
  payload-goodput measurement. Keep connection warm-up outside latency samples
  and report HTTP timing separately from authorization.
- [ ] Instrument direct uncached visa grant request/response timing. Add
  sustained/concurrent throughput, multi-node comparisons, repeatability,
  resource-use baselines, and published performance limits. The HTTP baseline
  is not a visa-grant timer or a network-capacity certification.
- [ ] Run format, lint, unit, integration, schema compatibility, fuzz, and
  multi-node tests in CI for every affected repository.
- [ ] Verify secrets are not logged; define audit retention, health checks,
  metrics, alerts, backup, restore, and key/certificate rotation procedures.
- [ ] Publish a deployment guide with secure defaults, threat assumptions,
  upgrade/rollback instructions, supported platforms, and known limitations.
- [ ] Make the demos exercise the documented secure path and continuously test
  them in CI where practical.
- [ ] Add a base-install smoke test that boots the generic manifest, overlays a
  scenario manifest, and asserts the same service set through Visa Service
  Admin API and Control Room.

**Exit criteria:** a clean release candidate passes the cross-repository CI
and end-to-end suite, and an operator can deploy and diagnose it from the
published guide without undocumented steps.

## Explicit Scope Decisions

These features appear in the public status docs as design work, but should not
silently block a public-reference release unless adopted as requirements:

- Byzantine agreement or replicated/federated Visa Services.
- Multiple policies active in parallel.
- Multicast and combining flows.
- Automatic topology generation.
- A2A confidentiality, if deployments explicitly permit integrity-only
  operation; document the choice and resulting guarantee either way.

For every deferred item, record the decision, security impact, and conditions
that would bring it back into scope. Features dependent on unpublished RFCs
remain blocked until a public specification or approved replacement exists.

## Repository Ownership Map

| Work area | Repositories |
|---|---|
| Data plane, nodes, adapters, ZDP, packet security | `zpr-core`, `zpr-common` |
| Policy language/compiler | `zpr-compiler`, `zpr-policy`, `zpr-common` |
| Visa issuance, evaluation, identity, admin, state | `zpr-visaservice`, `zpr-vsapi`, `zpr-common` |
| Control Room and Policy Repository API/UI | `zpr-visaservice/zpr-dashboard` |
| Public protocol/design requirements | `zpr-rfcs` |
| Integration examples and deployment tooling | `zpr-demo`, `zpr-dev-tools` |
| Build orchestration and cross-repository context | `zpr-dev-context` |

## Source Notes

This roadmap is based on the public repository set in the workspace, especially
[`SYSTEM_OVERVIEW.md`](SYSTEM_OVERVIEW.md),
[`SECURITY_MODEL.md`](SECURITY_MODEL.md),
[`ZPL.md`](ZPL.md),
[`VISA_SERVICE.md`](VISA_SERVICE.md), and
[`BUILD.md`](BUILD.md). The public RFC repo says that not
all RFCs and documentation are public. This plan does not use Oracle-hosted
specifications or assume access to private materials.