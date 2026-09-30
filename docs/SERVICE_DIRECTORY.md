# ZPL Service Contracts and DNS Integration

**Status: Architecture proposal; initial compiler and registry slices implemented.**
This is not yet a published RFC. The compiler accepts a service declaration
with one TCP/UDP port and emits its DNS name as the signed service ID. Visa
Service tracks multiple provider addresses per service ID. DNS query serving,
authenticated publication to a third-party backend, and bootstrap migration
remain unimplemented.

## Motivation

Service meaning is currently split across three places:

- ZPL defines service classes and uses them in `allow` and `never` rules.
- The `.zplc` TOML configuration defines concrete service protocol/port scopes.
- Actors advertise policy-authorized service IDs at admission, and Visa Service
  indexes each service ID to an actor ZPR address.

This makes it unclear whether a service is a policy concept, a configured
endpoint, or a live instance. The design target is one authoritative static
contract in ZPL and third-party DNS infrastructure for live service-instance
records. ZPR supplies identity, policy, and lifecycle integration; it does not
need to implement a new DNS server.

## Model

Keep three concepts distinct:

1. **Service definition:** A stable logical service ID, its class and attributes,
   and the protocol/port scopes that belong to it. This is part of the signed
   policy compiled from ZPL.
2. **Service instance:** A running provider of that service, with an instance
   identity, authenticated owner, assigned ZPR address, and an expiring lease.
3. **Service resolution:** A standard DNS query to the deployment's DNS service
  over an authenticated, policy-authorized ZPR flow returns currently
  registered instances for a logical service. A DNS answer is location
  information, not authorization.

The `.zplc` file continues to carry deployment configuration such as topology,
bootstrap trust material, substrate addresses, and the initial locations and
credentials needed to reach control-plane services. It no longer declares the
identity or transport scope of application services. The DNS provider and zone
remain deployment configuration, not policy semantics.

## ZPL Direction

Retain `define` for policy classes and add a declaration for a service contract.
The initial compiler syntax is:

```zpl
define PayrollAPI as a service with data-class:confidential.
provide PayrollAPI at payroll.finance.svc.zpr over TCP 443.

allow finance employees to access PayrollAPI.
```

The DNS name is normalized to lowercase and emitted as the service ID in the
signed policy, with its transport scope. The initial implementation accepts one
TCP or UDP port per declaration. Policy rules authorize communication by
attributes and service class; DNS names are identifiers, not authorization
predicates. The parser rejects duplicate class/name declarations, invalid DNS
labels, unsupported protocols, invalid ports, and non-service classes.

Service classes without `provide` continue to use legacy `.zplc`
`[services.*]` configuration. That fallback exists for migration; new
application service definitions should use ZPL.

## DNS Provider Integration

DNS is itself a ZPR service, not an open network service. Clients reach its
query endpoint only over authenticated ZPR flows that active policy permits;
registration and administration use separately authorized ZPR service
endpoints. Do not expose a public or underlay-reachable recursive resolver,
open DNS UPDATE, or an unauthenticated registration interface. Standard DNS
query messages may be used over the ZPR flow, but that does not make the
service publicly reachable.

Deployments select a third-party DNS implementation that supports the needed
private zone, authenticated updates, record TTLs, and ordinary DNS queries.
The ZPR DNS service fronts that implementation and adapts the ZPR service
lifecycle to its supported update mechanism, such as secure dynamic DNS
updates or an authenticated provider API. The backend is reachable only by the
ZPR DNS service over a private management path; clients never bypass the ZPR
service to query the backend. The provider supplies authoritative storage and
DNS mechanics; ZPR supplies endpoint identity, policy authorization, and the
decision about which records may be published.

The ZPR service namespace is private to the deployment and distinct from
substrate DNS used to locate node-to-node UDP endpoints. Its zone and name
mapping are configured as bootstrap data. The exact suffix and canonical-name
rules remain to be specified. The DNS query endpoint and its client listener
are reachable only in the ZPR network; only the DNS service's private backend
connection reaches the third-party provider interface.

The initial service-instance record should contain only:

- canonical service ID from the active signed policy;
- unique instance ID;
- authenticated service identity and its owning ZPR endpoint;
- the endpoint's assigned ZPR address, obtained from authenticated session state;
- lease expiry and optional policy-defined priority/weight.

Transport protocols and ports come from the signed service definition, never
from an untrusted registration request. The registrar must not accept arbitrary
substrate addresses. A DNS record points to the endpoint's ZPR address or a
policy-defined ZPR service name; it must not publish a caller-selected
underlay address. The publisher creates records only for instances whose
service ID is present in the active policy and whose provider is authorized by
that policy.

Service startup registers an instance through the ZPR registration component,
renews its lease while healthy, and withdraws it on orderly shutdown. The
integration removes or lets the DNS record expire when renewal stops, so
process or host failure does not leave a permanent answer. A policy change that
removes a service or its provider authorization triggers record withdrawal.
DNS TTLs must not exceed the remaining registration lease.

Visa Service now stores a set of provider ZPR addresses per service ID and
removes only the departing or updated provider. Resolution should return the
live instance set; final instance selection does not grant access, and every
resulting flow remains subject to Visa Service policy evaluation. An instance
that cannot prove its service identity or its binding to the owning ZPR actor is
not registrable. Adapter identity alone must not allow arbitrary services to
be claimed.

## Services and Bootstrap

ZPR components should use the same service and authenticated-endpoint model as
application workloads. The Visa Service, attribute service, policy service,
and DNS registration component are services with identities, declared
contracts, and policy-gated access. Control-plane components are not exempt
from ZPL policy merely because they manage the network.

This creates a bootstrap dependency: the Visa Service needs policy and
attributes before it can enforce policy on the services that provide policy
and attributes. Break the cycle with a deliberately small local bootstrap
bundle containing the initial file policy, bootstrap identities/keys, initial
attributes, and the addresses needed to reach the first trusted services. Use
it to bring up the Visa Service and attribute service, authenticate those
services, and establish their permitted flows. Once reachable, obtain
operational policy and attributes from their network services and transition
away from the bootstrap material according to a defined activation procedure.
Do not turn the bootstrap bundle into a permanent exception for all endpoints
or future policy.

The bootstrap policy must be minimal, explicit, and auditable. Its trust roots
and replacement procedure are security-critical: remote policy is accepted
only from the configured policy-service identity and under the configured
cryptographic verification rules. Loss or failure of policy or attribute
services must have specified fail-closed behavior and must not silently widen
bootstrap permissions.

## Trust and Failure Rules

- Registration and renewal occur over an authenticated ZPR control flow to the
  ZPR registration component; DNS provider updates use its authenticated
  provider credentials. Unauthenticated DNS UPDATE is not used.
- DNS queries are accepted only through the ZPR DNS service endpoint and only
  when the requesting endpoint's ZPR policy permits access. The third-party
  backend is not directly exposed to clients or to the substrate network.
- The registrar derives the owner address and identity from that authenticated
  flow rather than accepting them as caller-supplied fields.
- The active signed policy is the allowlist for service IDs, service classes,
  transport scopes, and provider eligibility.
- A compromised or unavailable DNS provider may cause discovery failure or
  return stale/misleading records, but it cannot grant a visa or bypass the
  evaluator. Clients and Visa Service fail closed when no valid instance is
  available; the Visa Service remains authoritative for authorization.
- The DNS provider, registration component, Visa Service, and initial
  attribute/policy services need bootstrap identities and reachable
  information independent of ordinary service discovery, avoiding a discovery
  cycle.
- DNS TTLs do not exceed the registration lease. On policy replacement or
  revocation, the publisher withdraws affected records; cached answers may
  remain until their bounded TTL expires. Any attempt to use a stale answer
  still goes through Visa Service policy evaluation and cannot gain access from
  DNS alone.

## Implementation Sequence

1. Extend the initial compiler declaration to multiple transport scopes and
  publish the naming and policy-ID rules in the ZPL specification. The first
  TCP/UDP single-port compiler slice and multi-provider actor registry are
  implemented.
2. Define the service identity and bootstrap model for Visa Service, attribute,
  policy, and DNS registration components. Specify local bootstrap policy and
  attributes, trust roots, remote policy verification, and transition/recovery.
3. Define a provider-neutral registration lifecycle and DNS adapter interface;
  select supported third-party DNS providers and their authenticated update
  mechanism, lease/TTL mapping, deletion, and conflict behavior.
4. Add compiler tests and binary-policy round-trip tests; remove the equivalent
  application-service endpoint declarations from `.zplc` while retaining
  deployment and bootstrap configuration.
5. Integrate service startup, authenticated registration/renewal, and client
  resolution. Keep actor admission's policy-derived service grants; do not
  trust self-asserted service claims.
6. Test spoofed identities, unauthorized names, expired leases, provider
  outages, stale caches, replayed updates, and policy replacement against the
  selected DNS provider adapter.
7. Update the Control Room to show policy declarations separately from live
  DNS registrations, including declared-only, registered, expired, and
  unauthorized states.

## Decisions Required Before Wire Implementation

- Which third-party DNS provider(s) to support first, and whether updates use
  RFC 2136 with TSIG, a provider API, or a provider-specific adapter.
- The exact private zone and canonical-name rules, including mapping current
  ZPL identifiers to DNS labels without silently changing identity.
- Which records to publish (for example A/AAAA and SRV), and how ZPR addresses
  map to DNS data without exposing underlay addresses.
- How a service process obtains a credential bound to both service ID and
  owning endpoint; current adapter credentials do not provide process-level
  separation.
- Lease duration, renewal cadence, provider TTL limits, and conflict behavior
  for multiple instances.
- Bootstrap and availability for the DNS provider, registration component,
  Visa Service, policy service, and attribute service.

These decisions must be recorded in the protocol specification and represented
in integration tests before the feature is described as available.

## Current Implementation Boundary

The compiler accepts `provide <class> at <dns-name> over TCP|UDP <port>`, emits
the normalized DNS name as the signed service ID, and serializes the declared
transport scope. Legacy `[services.*]` configuration remains supported.
Visa Service derives actor service IDs from policy-authorized admission and
stores each service ID's provider addresses in a set. Its authentication
service advertisement consumes every registered provider. It does not yet
serve DNS queries, publish records to a third-party DNS backend, or load
operational policy/attributes over the network bootstrap sequence.