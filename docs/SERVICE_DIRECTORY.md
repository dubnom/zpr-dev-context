# ZPL Service Contracts and DNS Integration

**Status: Initial implementation in progress.** ZPL service declarations, multi-provider actor indexing, a BIND 9 TSIG publisher, and an opt-in adapter loopback DNS stub are implemented. The stub forwards local UDP/TCP requests over TCP to one statically bootstrapped ZPR DNS address. Host resolver and route configuration remain operator-managed. This design has not yet been published as an RFC.

## Model

Keep three concepts distinct:

1. **Service definition:** A stable service class, DNS name, and transport scope in the signed policy.
2. **Service instance:** An authenticated endpoint currently offering that policy-declared service at its assigned ZPR address.
3. **Service resolution:** A DNS answer locating current providers. It is not authorization; every flow still needs a Visa Service decision.

Topology, substrate addresses, bootstrap trust material, and DNS deployment settings remain in `.zplc`/service configuration. Application service identities and transport scopes belong in ZPL.

## ZPL Service Declaration

The compiler accepts one TCP or UDP port per declaration:

```zpl
define PayrollAPI as a service with device.zpr.adapter.cn:payroll.
provide PayrollAPI at payroll.finance.svc.zpr over TCP 443.
allow finance employees to access PayrollAPI.
```

The DNS name is normalized to lowercase and becomes the service ID in signed policy. The compiler rejects duplicate class/name declarations, invalid DNS labels, unsupported protocols, invalid ports, and non-service classes. The optional internal `zpr.addr` attribute may pin a service endpoint to an assigned ZPR address; the value must match the address on which that service listens. Legacy `.zplc` `[services.*]` tables remain temporarily supported for migration.

## BIND 9 as a ZPR Service

BIND 9 is the DNS server and runs as a ZPR endpoint. It binds DNS query service only to its ZPR address, not a host or substrate address. ZPL policy authorizes authenticated ZPR clients to query it. Visa Service updates records over a ZPR flow using RFC 2136 and a dedicated TSIG key. No public resolver, underlay listener, or unauthenticated DNS UPDATE is enabled.

The checked-in example is [the BIND deployment guide](../zpr-visaservice/dns/bind9/README.md). It includes an authoritative-only BIND configuration, a `svc.zpr` zone, and ZPL policy for authenticated clients and the Visa Service publisher. The DNS server's ZPR address must be bootstrapped because clients cannot use DNS to discover the server that provides DNS.

An adapter can optionally expose a loopback-only DNS stub configured with `adapter.dns_proxy_listen` and `adapter.dns_proxy_server`. It accepts local UDP and TCP requests and always forwards over TCP to that one literal server address using a socket bound to the adapter TUN. It does not edit the host resolver, resolve public names, or fall back to an underlay resolver. Operators must configure the host resolver to use the loopback listener. DNS queries then follow ordinary ZPR flow and visa handling; connections to returned addresses still require separate authorization.

## Visa Service Publication

Visa Service stores a set of provider ZPR addresses per service ID. On startup, actor join/leave, and policy update, it reconciles the BIND A/AAAA RRsets to the current desired provider set. A provider is published only when:

- its service ID exists in the active signed policy and in the configured DNS zone;
- its authenticated actor currently advertises that ID;
- a matching current join policy explicitly grants that service; and
- the actor still passes current-policy admission.

Addresses come from Visa Service actor state, not from update callers. Owner names are validated against the configured zone. The publisher uses `nsupdate -k` with a private TSIG key, caps update execution time, replaces owned A/AAAA RRsets to remove stale addresses, and logs failures without granting access. DNS TTL bounds cache staleness; DNS itself never grants a visa.

Configuration is optional and disabled by default. A deployment supplies the BIND ZPR address, private zone, TSIG key file, TTL, and `nsupdate` path in the Visa Service config. The TSIG key must be owner-only and BIND's `update-policy` must restrict it to the dedicated zone and A/AAAA records.

## Bootstrap and Next Phase

The DNS actor's identity and ZPR address are bootstrap inputs. Start with a small local file policy, bootstrap keys, and initial attributes sufficient to bring up the Visa Service and DNS/attribute services. Once reachable, operational policy and attributes should be obtained from their authenticated network services; bootstrap permissions must not silently widen on failure.

The initial adapter query path is available through the optional loopback stub. Remaining client work is safe platform-specific resolver and route provisioning, dynamic adapter-address/bootstrap integration, and an end-to-end test proving DNS queries and connections to returned addresses both traverse normal visa handling. DNS answers remain candidates only; Visa Service policy evaluation authorizes actual flows.

## Remaining Work

- Add platform-specific route/resolver provisioning and end-to-end ZPR DNS flow tests; the local stub already translates UDP/TCP client requests to upstream TCP.
- Provision the BIND TSIG key and zone through deployment tooling; validate `named.conf` and zone files with BIND's `named-checkconf`/`named-checkzone`.
- Add a live BIND integration test for signed updates, RRset replacement, TTL expiry, wrong-key rejection, and policy/provider removal.
- Specify the public service-name convention, multi-instance selection behavior, and recovery/rotation procedures.
- Move bootstrap policy and attributes to authenticated network services with explicit verification, activation, and fail-closed recovery.

The Visa Service does not implement a general DNS server. BIND is the ZPR service endpoint; the compiler and Visa Service provide its policy declaration and authenticated registration lifecycle.
