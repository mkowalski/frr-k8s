# Redistribution Support for FRRConfiguration

## Summary

This proposal adds route redistribution to the FRRConfiguration CRD.
Users can advertise routes from a kernel routing table.
The reachability decision stays in the kernel table.
The CRD only bounds which prefixes may leave.

## Motivation

Some agents signal per-node state through kernel routes.
Example: a health checker (kubevip) installs a /32 into table 198 while a local backend is healthy.
FRR should advertise that route only while it exists.
This gives automatic per-node withdrawal and ECMP across healthy nodes.

Today the CRD cannot express this:

1. `router.prefixes` renders unconditional `network` statements. This bypasses the gating.
2. `toAdvertise.allowed.mode: all` only allows prefixes declared in `router.prefixes`. Redistributed routes are always filtered out on egress.

The only workaround is `rawConfig`. It must append permits to the generated `<neighbor>-out` route-maps.
That couples user config to internal naming and sequence numbers. It can silently break on frr-k8s upgrade.

OpenShift's BGP-based VIP management plans to use this pattern in production and carries the fragile rawConfig today in the POC.

### Goals

- Advertise routes redistributed from a kernel table (`table-direct`).
- Filter egress to an explicit prefix allow-list.
- Compose with the generated per-neighbor route-maps. No raw config.

### Non-Goals

- Redistributing other protocols (connected, static, kernel, OSPF). The API leaves room for them.
- Import policy or route modification (communities, med) for redistributed routes.
- Managing the kernel table content. That is the user's agent's job.
- VRF routers. Deferred until FRR's per-VRF `table-direct` behavior is verified and we actually need that.

## Proposal

### User Stories

As a cluster administrator, I want to:

1. Advertise a VIP only while my health-check agent keeps its route in a kernel table.
2. Bound the advertisement to an explicit prefix list so nothing else can leak.
3. Upgrade frr-k8s without my egress filters breaking.

### API Changes

Add a `redistribute` list to `Router`:

```yaml
apiVersion: frrk8s.metallb.io/v1beta1
kind: FRRConfiguration
metadata:
  name: vip-advertisement
spec:
  bgp:
    routers:
    - asn: 64512
      neighbors:
      - address: 192.168.1.1
        asn: 64513
        toAdvertise:
          allowed:
            mode: all
      redistribute:
      - protocol: table-direct
        table: 198
        allowedPrefixes:
        - 192.168.111.4/32
        - 192.168.111.5/32
```

Fields:

- `protocol`: only `table-direct` initially. Enum, extensible.
- `table`: kernel table id. Required for `table-direct`.
- `allowedPrefixes`: prefixes permitted to leave. Required. No implicit "all".

### Dual-stack

`allowedPrefixes` may mix IPv4 and IPv6. The renderer splits them by family.
Each family gets its own route-map, prefix-list and `address-family` block.
A family with no prefixes renders nothing. No validation against neighbor families is needed.
`table-direct` supports both families in FRR.

### Generated FRR Configuration

Names are scoped by VRF and family: `redistribute-<vrf>-<table>-<family>`. Initially always `default`.
The VRF placeholder future-proofs the naming for VRF support.

```
router bgp 64512
 address-family ipv4 unicast
  redistribute table-direct 198 route-map redistribute-default-198-ipv4
route-map redistribute-default-198-ipv4 permit 1
 match ip address prefix-list redistribute-default-198-allowed-ipv4
route-map redistribute-default-198-ipv4 deny 2
ip prefix-list redistribute-default-198-allowed-ipv4 seq 1 permit 192.168.111.4/32
ip prefix-list redistribute-default-198-allowed-ipv4 seq 2 permit 192.168.111.5/32
```

IPv6 prefixes render the same under `address-family ipv6 unicast`, with `ipv6 prefix-list` and `-ipv6` names.

Egress: for neighbors with `toAdvertise.allowed.mode: all`, the redistributed `allowedPrefixes` are appended to the neighbor's generated allowed prefix-lists (`ToAdvertisePrefixListV4`/`V6`).
No extra route-map clauses.
Neighbor modifiers like `set ip next-hop` live in the main permit rule and apply uniformly.
Neighbors with explicit `allowed.prefixes` are untouched. They advertise a redistributed prefix only if it is also in their own allow-list.
`toAdvertise` semantics for declared prefixes stay unchanged.

`table-direct` reads the kernel table directly. No `ip import-table` is needed.

### Validation

- Reject `table` outside 1-65535. This mirrors FRR's `redistribute table-direct (1-65535)`.
- Reject empty `allowedPrefixes` or any invalid CIDR block within it.
- Reject duplicate `table` entries within one router.
- Reject `redistribute` on VRF routers.
- Merge across FRRConfigurations: union of `allowedPrefixes` per table.
  Fail the merge if the same table is declared with different protocols.
- The webhook's outgoing-prefix check (`validateOutgoingPrefixes`) must
  accept redistributed prefixes: the union of `redistribute.allowedPrefixes`
  joins the router's known prefixes. Otherwise a `filtered` neighbor listing
  a redistributed prefix is falsely rejected.

## Alternatives Considered

- **Keep rawConfig.** Fragile and defeats the purpose.
- **CRD-declared prefixes (`network` statements).** Unconditional. Defeats health gating.

## Test Plan

- Unit: api_to_config coverage for the new stanza.
- E2E: install route in table, expect advertisement; remove route, expect withdrawal; verify a non-allowed prefix in the table never leaves.
