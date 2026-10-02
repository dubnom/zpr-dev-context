# Standard ZPR Multi-Node Link Contract

**Status: implementation proposal.** This document scopes the first
multi-node milestone to ordinary ZPR nodes over the currently implemented
IP/UDP substrate. It does not depend on OCI, a cloud provider, or the demo's
deployment scripts. The downloaded [RFC 17 source](../../zpr-rfcs/src/17-ZPR-Data-Protocol/body.md)
and [PDF](../../zpr-rfcs/pdf/17-ZPR-Data-Protocol.pdf) are from the public
`sj/upload-17` branch at commit
`558383737d0768655214d7fde62ff34329740007`; the RFC is absent from `main` as
of 2026-10-01, so verify that this branch is the intended authoritative
revision before treating it as a released standard.

## Goal

Two independently running ZPR nodes, each with its own node address and
substrate endpoint, establish an authenticated node-to-node link. The Visa
Service installs the policy-declared edge only when the peer link is usable,
then can distribute visa fragments over paths that cross that edge. Existing
adapter-to-node docking behavior must remain unchanged.

## Peer Intent

A peering is an explicit, trusted configuration/policy relationship, not an
inferred connection from arbitrary UDP traffic. RFC 17 leaves configuration
distribution out of scope and requires both ends to be provisioned with their
local and expected peer Node Addresses, cryptographic configuration, link
connection policy, substrate address/protocol/port, and any applicable link
attributes. The existing shared `Peering` model contains a policy `link_id`,
both node ZPR addresses, both substrate addresses, and attributes. See
[`topology.rs`](../../zpr-common/src/policy_types/topology.rs).

For the first implementation, the authenticated Visa Service `SetTopology`
update supplies policy peer addresses and substrate endpoints. There is no UDP
peer discovery. The administrator-owned `[node.peer_noise_certificates]`
mapping supplies each expected peer certificate, from which PH pins the Noise
public key. Certificate trust is still checked against the configured CA
during keying. Do not make a topology learned from an unauthenticated packet
sufficient to authorize a new peer.

The node configuration shape is:

```toml
[node.peer_noise_certificates]
"fd5a:5052:90de:1::2" = "peer-two-noise.pem"
```

The key is the expected peer Node Address; certificate paths are relative to
the node configuration file unless absolute. Every configured certificate
must contain a 32-byte Noise public key.

For a configured edge `A <-> B`:

- Each endpoint resolves the expected peer Node Address and substrate endpoint
  from the configured peering. No dynamic UDP peer discovery is specified.
- Both ends associate the connection with the same policy peering and
  compatible link attributes; the RFC does not define the policy `link_id` as
  a wire-level identifier.
- The substrate endpoint identifies where ZDP packets are sent; it is not the
  node's ZPR address or cryptographic identity.
- Unknown substrate senders must not be promoted directly into trusted peers.
  A candidate must complete keying against the configured expected peer and
  the node-specific link procedure before it can carry transit or management
  traffic.

The current substrate implementation is UDP/IP. This contract does not add
Ethernet discovery, multicast, ZARP, or automatic topology generation.

## Identity And Security

- The configured peer Node Address is the expected logical ZPR identity. Bind
  it in trusted local configuration to the exact expected Noise static public
  key (or its cryptographic fingerprint); source IP/UDP address and
  certificate CN alone are not authorization identities.
- Require a certificate, verify its signature against the configured CA, and
  require its public key to match both the Noise handshake key and the
  configured peer key. Treat the CN as diagnostic metadata unless a separately
  specified, verified mapping binds it to the Node Address.
- The NodeToNode keying path now requires a certificate and checks the pin,
  Noise handshake key, and configured CA. The generic adapter certificate
  path retains its existing optional-certificate behavior.
- The resulting security association is pairwise and specific to this peer
  link. An absent, unverified, or mismatched identity fails closed before the
  peer is admitted to the Node peer table.
- Management traffic is accepted only after the link's security association
  and protocol state permit it. Transit traffic additionally requires valid
  visa state and forwarding-table entries at each hop.
- A node-to-node peer is represented distinctly from a node-to-adapter tether;
  code must not route its packets through adapter-only address-registration or
  docking logic.
- RFC 17 gives each Node its own generated Link Address. This is distinct from
  the policy `Peering.link_id` and PH's process-local `LinkId`; none may be
  treated as interchangeable without an explicit mapping.
- Link security associations and forwarding state are made unusable when the
  link goes down. Reconnection must establish fresh link state without
  reviving stale stream entries.

## Establishment And Lifecycle

1. A node loads trusted configured peerings and resolves each peer's substrate
  endpoint and expected Noise key. Configuration distribution is outside RFC
  17's scope; the first implementation uses administrator-provisioned local
  configuration and applies updates by replacing that configuration.
2. If the keying protocol requires one initiator, the Node with the numerically
  smaller Node Address, compared as an unsigned integer, initiates; the other
  waits. This is the required role rule for the current Noise keying path.
3. After keying establishes the pairwise security association, both Nodes send
  Hello Requests, receive successful matching Hello Responses, and answer the
  peer's Hello Request. A non-success response tears down the link. The FSM
  reaches Hello Done only after both directions complete.
4. Each Node generates and assigns its own Link Address, then both peers start
  periodic Echo liveness checks. A link is eligible for forwarding only after
  successful keying, bidirectional Hello completion, and local link setup.
5. Once the link is available, local topology management is notified. The
  Visa Service's policy edge is live only after the node-side link is usable;
  desired policy topology alone must not report the edge `UP`.
6. On Echo failure, the detecting Node immediately stops transmitting on the
  link, marks it down, releases its security association, and reports failure
  so topology and affected visas can be recomputed. It then retries
  periodically. Resolve RFC 17's conflicting `MAY`/`MUST` wording
  conservatively: retry is required after failure handling; Connection Policy
  may set retry interval/backoff and an up-state quiet period, but may not
  disable retries for a configured peer. Use the existing 5-second PH restart
  holddown as the initial default, subject to policy configuration.
7. For an administrator-requested removal, first stop new forwarding and
  invalidate local link state. Use Terminate Request/Response for graceful
  peer notification when the RFC assigns stable reason-code values; if the
  peer does not respond, finish local teardown without waiting indefinitely.
  Echo failure cleanup does not wait for a Terminate exchange. RFC 17 defines
  the message formats but leaves the Link Termination procedure and reason
  code values `TBD`, so do not invent on-wire codes.

The RFC defines the keying initiator, hello, Link Address, and liveness
requirements, but does not define configuration distribution, the Node
Address-to-key binding, or the local API by which the Visa Service learns that
a Link became available. The choices above are this implementation's
configuration and control-plane contract, not additional RFC wire semantics.

## Topology And Forwarding

- The Visa Service's policy snapshot remains the source of desired topology.
- The existing `SetTopology` direction is Visa Service to Node and describes
  desired links. PH now reconciles configured Node peers from this update,
  including removing stale peers; receipt of the update is not evidence that a
  link is active.
- Add a node-to-Visa-Service `reportLinkStatus` operation on the existing
  `VSHandle` interface. Report policy `link_id`, peer Node Address, local
  state, and a monotonically increasing per-process generation so stale
  reports after reconnect can be ignored. This is a new VSAPI operation; the
  existing interface has no link-status report.
- The live topology manager contains only edges for which the nodes have
  established usable peer links. Require active reports from both endpoints
  and successful persisted router-edge installation before reporting the
  edge `UP`; on either endpoint's down/disconnect report, remove the edge and
  recompute affected paths. Admin/status APIs report policy intent separately
  from live link state.
- The Visa Service selects routes from the policy-declared graph and distributes
  each node only the forwarding/visa state needed for its local hop.
- Nodes forward by per-link Stream ID state. They do not evaluate the whole
  ZPL policy or trust the substrate route as a substitute for ZPR forwarding.
- Loss of an edge invalidates affected forwarding paths/visas according to the
  existing Visa Service policy and revocation rules; the behavior must fail
  closed until a valid route is restored.

The Visa Service already stores policy-declared peerings and has topology
manager/router code. Its current join path installs policy edges based on node
presence; that is desired-route state, not proof that a PH peer link exists.
The RFC requires the live Link to be reported to topology management. See
[`TopologyMgr`](../../zpr-visaservice/vs/src/topology_mgr.rs) and
[`install_policy_links_for_node`](../../zpr-visaservice/vs/src/vsapi_worker.rs).

## Acceptance Tests

The first standard ZPR multi-node milestone is complete only when a local,
reproducible test with two PH node processes demonstrates all of the following:

1. Both nodes authenticate the configured peer identity and report one usable
  link for the policy peering, with local Link Address and PH LinkId kept
  distinct from the policy `link_id`.
2. The Visa Service reports the declared edge `UP` only after link
   establishment; it reports it down after teardown or loss.
3. An allowed endpoint flow crossing node A to node B succeeds and is enforced
   at both nodes.
4. A denied flow across the same topology remains denied; adding a link never
   grants access by itself.
5. Removing the peer link stops cross-node delivery and does not leave stale
   forwarding entries; restoring it recovers without restarting unrelated
   nodes or the Visa Service.
6. Duplicate connection attempts and simultaneous startup converge to one
  link. The smaller-address Node initiates; the other has a preconfigured
  responder entry, and repeat requests for the same policy `link_id` are
  idempotent.
7. Restarting either node re-establishes the edge and restores the expected
   topology/API state.
8. The first milestone permits at most one logical peering per unordered pair
  of Node Addresses. Reject parallel peerings and multi-homed endpoints rather
  than silently collapsing distinct policy IDs into one router edge.
9. A substrate endpoint change preserves the logical Node Address and policy
  `link_id`, but tears down the old association and establishes a fresh link
  from updated trusted configuration. No in-place endpoint migration is
  attempted.
10. A missing, untrusted, or mismatched peer certificate/key fails before the
  candidate enters the node peer table; an unconfigured UDP sender is never
  admitted as a node peer.
11. The Visa Service reports the edge `UP` only after it has active reports
  from both endpoints and persisted the router edge. A down report or node
  disconnect removes the live edge and triggers route/visa reconciliation.
12. A stale status report from a prior node connection generation cannot
  restore an edge after the node reconnects or reports it down.

Tests should assert both wire/link state and user-visible outcomes, not just
that two `ph` processes are running.

## Remaining Standards Dependencies

These are not blockers to the initial implementation proposal, but require
resolution before claiming full RFC conformance or adding new wire behavior:

- RFC 17's restart language is inconsistent: Link Establishment says automatic
  restart `MAY` be policy-controlled; Liveness Detection says the peer `MUST`
  periodically attempt restart after failure handling. This contract follows
  the stronger requirement and lets policy tune timing only.
- RFC 17 leaves the Link Termination procedure and reason-code values `TBD`.
  The implementation allocation below matches PH's existing values, but must
  be confirmed upstream before claiming RFC conformance. Graceful termination
  procedure details still need a standards clarification.
- RFC 17 does not define a standard binding format between Node Address and
  key/certificate identity. The configured key pin above is a local
  provisioning rule, not a standardized certificate name convention.

## Implementation Decisions

- Before keying, classify configured substrate endpoints from the peer table
  as `NodeToNode`; never decide link type from the first packet's claimed
  identity. Unknown endpoints may enter the existing adapter bootstrap path
  only as adapters and must pass adapter credential checks. Reject config that
  assigns the same substrate endpoint to both a node peering and an adapter.
- Key all control-plane state by policy `link_id` plus peer Node Address. Keep
  the RFC-generated per-side Link Address and PH's local `LinkId` as separate
  identifiers. Require policy `link_id` to be unique across configured links
  and stable across endpoint changes.
- For the first release, permit one logical link per unordered Node Address
  pair. Do not merge duplicate policy IDs, create parallel links, or support
  multi-homing/load sharing until the router and status API model them
  explicitly.
- On trusted config replacement, compare peer Node Address, substrate
  endpoint, and expected key. If any changes, close the old link, invalidate
  its SA and forwarding state, then establish a new link; retain the policy
  `link_id` only when it still names the same logical peering.
- The smaller unsigned Node Address starts keying whenever the protocol needs
  an initiator. The other Node creates a responder entry before receiving
  packets. Repeated starts for the same configured peering are idempotent.
- Terminate reason codes use the existing PH allocation: `0 OTHER` (generic
  reason), `1 BAD_SEQUENCE_NUMBER` (invalid response sequence), `2
  REQUEST_TIME_OUT` (request/retry failure), `3 RESET` (reset indication only),
  and `4 SHUTDOWN` (administrative shutdown; receiver must not auto-restart).
  A received `RESET` indication also tears down the peer without automatic
  restart.
  Values `5` through `255` are reserved. Unknown values are handled as `OTHER`
  for restart policy only: preserve/log the unknown numeric value, process the
  termination, and use normal restart behavior unless a future assigned code
  specifies otherwise.

## Current Code Boundary

PH can now create NodeToNode peers from authenticated `SetTopology` updates,
require administrator-pinned peer Noise certificates, choose the RFC initiator
by unsigned Node Address, perform bidirectional Hello, and start Echo
liveness. The peer-table lookup distinguishes preconfigured Node peers from
unknown adapter tethers. Topology-carried bootstrap visas are sent in the
existing `BOOTSTRAP_VISA` Hello TLV and installed only when their endpoints
name the authenticated peer. Still missing are two-node integration tests,
end-to-end route/visa validation, and node-to-node unbind handling. The new
`reportLinkStatus` RPC validates each endpoint's policy link ID and gates the
persisted router edge on fresh up reports from both Nodes; service startup
clears restored edges until they report. The separate OCI multi-node demo is
out of scope for this standard ZPR milestone.