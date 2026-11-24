# Dynamic IPv6 Multi-tenancy for Port Access Control

## 1. Overview

This document proposes an extension to SONiC Port Access Control (PAC) that
assigns an IPv6 address to each successfully authenticated 802.1X client,
isolates tenants with dynamic ACLs in both directions, and advertises aggregated tenant prefixes
through BGP. RADIUS provides the tenant identifier. PAC remains responsible
for the authentication session and coordinates address, ACL, and route state.

This design extends the PAC behavior in [Port Access Control](Port%20Access%20Control.md).
The current PAC design documents VLAN, session timeout, and session termination
authorization attributes; tenant identity, dynamic IPv6 profiles, and ACL-based
isolation are additions.

## 2. Goals and non-goals

### Goals

- Use the existing 802.1X authenticator and RADIUS exchange.
- Extract the tenant ID of the authenticated client from the RADIUS reply.
- Derive a unique IPv6 address from a configured profile and client/port data.
- Enforce tenant isolation using ingress ACL rules, not VRFs or VLANs.
- Originate aggregated IPv6 routes for authenticated tenants into BGP.
- Remove addresses, ACL entries, and route contributions when a session ends.
- Fail closed if the switch cannot install the required isolation ACL.

### Non-goals

- Replacing PAC authentication, supplicant, or RADIUS server behavior.
- Assigning tenant-specific VLANs or VRFs.
- Defining a new RADIUS standard attribute. The tenant ID is carried in the
  NVIDIA VSA `Nvidia-User-Tenant-ID`, as on Cumulus Linux.
- Changing IPv4 forwarding or providing address assignment through DHCPv6.

## 3. Configuration model

The management model defines named IPv6 profiles, profile properties, and
profile-to-interface bindings in CONFIG_DB. The YANG model is
`sonic-pac-ipv6.yang` and defines these tables:

| CONFIG_DB table | Key | Fields |
| --- | --- | --- |
| `PAC_IPV6_PROFILE` | profile name | `route_tag` (optional) |
| `PAC_IPV6_PROFILE_PROPERTY` | `profile|property` | `type`, `offset`, `length`, `value`, `isolation`, `summarize` |
| `PAC_PORT_IPV6_PROFILE` | interface name | `profile`, `port_id` (`auto` by default) |

### 3.1 Profiles and properties

A profile is a set of bit-field properties that together build a 128-bit IPv6
address. Bits not covered by any property are zero. `offset` counts bits from
the most-significant bit of the address.

| Field | Meaning |
| --- | --- |
| `route_tag` (profile) | Optional route tag for the profile's summary route, 3 to 4294967295. See [Summary routes](#6-summary-routes). |
| `type` | `literal` writes `value`; `tenant-id` writes the tenant ID from the RADIUS reply; `port-id` writes the port ID of the bound interface. |
| `offset`, `length` | Bit range of the property. `offset + length` must not exceed 128. `literal` and `tenant-id` properties are at most 32 bits; `port-id` properties are at most 64 bits. |
| `value` | Unsigned decimal or `0x` hexadecimal value; required for `literal`, not allowed for other types. |
| `isolation` | Include this property's bits in the ACL isolation value and mask. Defaults to on for `tenant-id` and off otherwise. If no property sets it, the `tenant-id` bits are used. |
| `summarize` | Summarize out this property and every bit after it. At most one property per profile, and never at offset 0. Defaults to off. |

The tenant ID is read from the NVIDIA vendor-specific attribute
`Nvidia-User-Tenant-ID` (vendor 5703, attribute 1, integer). The attribute is
fixed; it is not configurable.

### 3.2 Interface binding and port ID

`PAC_PORT_IPV6_PROFILE` binds a profile to a physical port. `port_id` is either
an explicit unsigned integer or `auto`. `auto` uses the number in the SONiC
interface name, so `Ethernet0` gets port ID 0 and `Ethernet48` gets 48.

This differs from Cumulus, which derives the port ID from the low-order bits
of the port's MAC address. SONiC front-panel ports normally share the system
MAC, so a MAC-derived ID would be the same on every port and the same tenant
would get the same address on every port. The interface name is unique per
port and stable across reboots and link changes.

A port ID must fit its property without truncation. With an 8-bit `port-id`
property, `auto` works for `Ethernet0` to `Ethernet255`; higher-numbered ports
need a wider property or an explicit port ID.

### 3.3 Port mode

A port with a bound profile is a tenant port. It is configured as an L2 access
port: a VLAN member with no `INTERFACE` entry. While no tenant session is
authorized it behaves as any PAC port, so unauthenticated traffic is blocked
by PAC's L2 mechanisms. When the first tenant session is authorized, PAC turns
the port into a routed port; after the last session ends, it returns the port
to its VLANs.

| | L2 (no session) | Routed (one or more tenant sessions) |
| --- | --- | --- |
| VLAN membership | Configured VLANs | None |
| Router interface | None | Created by PAC |
| IPv6 addresses | None | One `/128` per session |
| Isolation ACLs | None | Bound at ingress and egress |
| Summary routes | None | Advertised |

Every client on a tenant port must carry a tenant ID. A client whose RADIUS
reply has no tenant ID is not authorized there, and RADIUS VLAN attributes are
ignored for tenant sessions. On a port without a profile, the tenant attribute
is ignored and PAC behaves as before.

The switch never changes CONFIG_DB, so `config save` cannot capture the
routed state:

- PAC sets `pac_routed` on the port in STATE_DB `OPER_PORT`. `vlanmgrd` then
  removes the port's configured VLAN memberships from the kernel and APPL_DB,
  which also detaches it from the bridge. VLAN memberships configured while
  the port is routed are recorded and applied when it is released.
- PAC writes the router interface and each session's `/128` to APPL_DB
  `INTF_TABLE`, so orchagent creates the router interface and traps the
  addresses to the CPU. orchagent creates a port router interface only after
  the port has left its VLANs. PAC also installs each address on the kernel
  interface.
- When the last session ends, PAC removes the addresses, flushes the
  neighbors the kernel learned on the port (orchagent cannot remove a router
  interface that neighbor entries still reference), removes the router
  interface, and clears `pac_routed` so `vlanmgrd` restores the VLANs.
- PAC records the ports and addresses it owns in STATE_DB
  `PAC_IPV6_ROUTED_PORT`. When pacd starts it clears its session state and
  releases every port recorded there.
- Leaving the bridge produces an `AF_BRIDGE` `RTM_DELLINK` for the port.
  pacd and hostapdmgrd ignore it; it does not mean the link was deleted.

Cumulus documents 802.1X only on bridge access ports and does not describe
the port's L3 state after authorization.

### 3.4 CLI

Profiles can be configured from the SONiC CLI without editing CONFIG_DB JSON:

```shell
sudo config pac ipv6-profile add TENANT-A --route-tag 911
sudo config pac ipv6-profile property add TENANT-A PREFIX \
  --type literal --offset 0 --length 32 --value 0xfd0007b0
sudo config pac ipv6-profile property add TENANT-A TENANT-ID \
  --type tenant-id --offset 32 --length 16
sudo config pac ipv6-profile property add TENANT-A PORT-ID \
  --type port-id --offset 48 --length 16 --summarize
sudo config pac ipv6-profile bind Ethernet0 TENANT-A --port-id auto
sudo config save -y
```

Other commands:

- `set <profile> --route-tag <tag>` changes the route tag and
  `set <profile> --no-route-tag` removes it; exactly one of the two is required.
- `property delete <profile> <property>` removes a property.
- `delete <profile>` removes a profile that has no properties and no bound
  interfaces.
- `unbind <interface>` removes a binding.

### 3.5 Validation

The CLI rejects:

- a route tag outside 3 to 4294967295 (bgpcfgd reserves tags 1 and 2);
- a property that overlaps another property or does not fit in 128 bits;
- a `literal` or `tenant-id` property longer than 32 bits, and a `literal`
  value that does not fit its length;
- a summarize property at offset 0, or a second summarize property;
- `bind` to an interface that is not a physical port, that is not a VLAN
  member, or that has an `INTERFACE` entry, or to a profile without a
  `tenant-id` property;
- an explicit port ID on a profile without a `port-id` property;
- a port ID, explicit or automatic, that does not fit the `port-id` property,
  or that another interface bound to the same profile already uses.

The YANG model enforces types and ranges, including the route-tag range, but
not the cross-field rules above. Configuration loaded without the CLI is
checked again when a client authenticates: PAC rejects authorization when the
profile is invalid, when the tenant ID or port ID does not fit its property,
when the generated address is unspecified, multicast, or link-local, or when
another active session already uses the address.

### 3.6 Operational state

For each authorized client PAC writes these fields to
`PAC_AUTHENTICATED_CLIENT_OPER_TABLE` in STATE_DB, in addition to the existing
PAC session fields. Tenant sessions have `vlan_id` 0.

| Field | Meaning |
| --- | --- |
| `ipv6_profile` | Profile bound to the interface. |
| `ipv6_address` | Generated address, installed as a `/128` on the interface. |
| `tenant_id` | Tenant ID from the RADIUS reply. |
| `isolation_id`, `isolation_mask` | Masked value and mask used by the isolation ACL. |
| `summary_prefix` | Summary prefix the session contributes to; empty without a summarize property. |
| `route_tag` | Route tag of the summary route; empty without a profile route tag. |

PAC also programs per-interface isolation ACLs before installing the address;
see [ACL isolation policy](#5-acl-isolation-policy).

## 4. Architecture and data flow

1. `hostapd` authenticates the supplicant, parses the NVIDIA tenant VSA, and
   passes the tenant ID to PAC with the authentication result.
2. authmgr sees that the port has a profile, requires the tenant ID, and
   authorizes the session without a VLAN. It removes the L2 discard entry that
   blocked the client before authentication.
3. PAC selects the interface profile and derives the client's IPv6 address.
4. PAC writes session ACL tables and rules to APP_DB. The ACL orchestrator
   programs the ASIC and reports status in STATE_DB.
5. PAC waits for each required table and rule to become `Active`. On a timeout
   or inactive rule, PAC rejects authorization and does not install the address.
6. For the port's first session, PAC switches the port to routed mode
   ([Port mode](#33-port-mode)) and waits for it to leave its VLANs.
7. PAC installs the `/128` in APPL_DB `INTF_TABLE` and on the kernel
   interface, advertises the summary route, and records the session in
   STATE_DB.
8. On logout or teardown, PAC withdraws the summary contribution and the
   address. After the port's last session it removes the router interface and
   returns the port to its VLANs, then removes the isolation ACLs.

The implementation should reuse PAC's existing hostapd-to-pacd authorization
message and its operational session state. It should not let `hostapd` directly
program addresses, ACLs, or BGP.

## 5. ACL isolation policy

PAC owns dynamic ingress and egress ACL objects in APP_DB; operators do not
configure the session rules. PAC registers a custom ACL table type with source
IPv6, destination IPv6, IP protocol, and ICMPv6 type matches, then binds a
per-interface table at both stages. For each isolation group present on an
interface, PAC permits packets when both source and destination match that
group's configured masked IPv6 value. Four explicit permits pass ICMPv6
Neighbor Solicitation, Neighbor Advertisement, Router Solicitation, and Router
Advertisement (types 135, 136, 133, and 134). A lower-priority catch-all IPv6
source rule drops all remaining IPv6 traffic. IPv4 and non-IP traffic are
outside this PAC rule set. The policy applies even when a destination address
has no active session on this switch.

PAC waits for the ACL orchestrator to report `Active` for each required table
and rule in STATE_DB before address installation and authorization. Table or
rule failure keeps the client unauthorized. Rule identifiers are stable across
retries and cleanup. PAC removes stale rules after session teardown and removes
the per-interface ACL table when the last profiled session leaves.

The same masked group value is matched on both packet endpoints, as in the
Cumulus rules observed on `swp1`. This requires ASIC support for ingress and
egress ACLs, masked `SRC_IPV6`/`DST_IPV6`, `IP_PROTOCOL`, `ICMPV6_TYPE`, and
packet action. Unsupported or exhausted ACL resources reject authorization.
Each tenant permit is derived from an authenticated session on that interface;
the mask is determined by the profile's isolation properties.

## 6. Summary routes

Summarization follows the Cumulus summarize-out model. When a profile has a
property with `summarize` set, the summary prefix is the session address
truncated at that property's offset: the property and every bit after it are
cleared. For example, with the profile above, tenant 42 on any port yields
`fd00:7b0:2a::/48`. A profile without a summarize property assigns addresses
but advertises nothing.

PAC writes each summary to APPL_DB `STATIC_ROUTE` as
`blackhole=true`, `advertise=true`, `expiry=false`, plus `tag=<route_tag>` when
the profile has a route tag. PAC records the routes it owns in STATE_DB
`PAC_IPV6_ROUTE_SUMMARY` with `prefix`, `session_count`, and `route_tag`, and
keeps a summary while at least one session contributes to it. It refuses to
replace an existing APPL_DB or CONFIG_DB static route at the same prefix. If
two sessions request the same summary with different route tags, the new
session is not authorized.

bgpcfgd's static-route manager encodes advertisement in the FRR route tag:
tag 1 means advertise and tag 2 means do not advertise, and
`route-map STATIC_ROUTE_FILTER permit 10` matches tag 1 for
`redistribute static`. A summary without a route tag uses tag 1. A summary
with a route tag `T` is installed as `ipv6 route <prefix> blackhole tag T`,
and bgpcfgd adds a matching entry to the same route map:

```
route-map STATIC_ROUTE_FILTER permit N
 match tag T
```

`N` is the lowest free sequence number from 100 upward, allocated per tag and
shared by every static route that uses the tag, in either CONFIG_DB or APPL_DB.
bgpcfgd removes the entry when the last route using the tag is deleted, and
re-adds all custom-tag entries when redistribution is re-enabled. A custom tag
is used only for advertised routes; a route with `advertise=false` keeps tag 2.
bgpcfgd ignores a static route whose tag is not a number from 3 to 4294967295.

A session's summary prefix and route tag are fixed at authorization. Changing
`route_tag` or the profile's properties affects new sessions only; an existing
session keeps its route until it re-authenticates or ends.

## 7. Address and route lifecycle

- Address generation must be deterministic for a profile, tenant, interface,
  and client identity, while guaranteeing uniqueness within the profile.
- The port identifier should be stable across reboot and independent of the
  interface's transient link state. Multi-auth ports require a client identifier
  so clients do not collide.
- Install address and local route state only after ACL installation succeeds.
- Switch the port to routed mode for its first session and back to L2 after
  its last; never leave a port routed without an authorized session.
- Generate one summary per distinct summarize-out prefix, not one BGP route per
  client. Maintain a reference count for active authorized sessions contributing
  to each summary.
- Redistribute generated summaries through SONiC's `STATIC_ROUTE_FILTER` route
  map, using the profile's route tag when set. Only static routes tagged 1 or
  with a configured custom tag are redistributed.
- On restart, reconcile persisted PAC session state against hostapd sessions,
  addresses, ACL entries, and route contributions; remove orphaned state.
- If address or route registration fails after ACL installation, remove the ACL
  and leave the client unauthorized. Cleanup failures must be retried and
  reported as operational errors.

## 8. Operational state and observability

Expose per-session operational state including interface, client MAC, profile,
tenant ID, IPv6 address/prefix, ACL installation status, authentication state,
session age, summary prefix, and route tag. Provide aggregate views for configured
profiles, active tenant summaries, and ACL programming errors. Tenant identifiers
should be treated as operational data and not logged with credentials or shared
secrets.

## 9. Failure handling and security

- Missing, malformed, or out-of-range tenant attributes: reject authorization.
- Unknown profile or invalid generated address: reject authorization.
- ACL resource exhaustion or unsupported match/action: reject authorization.
- Address installation failure: withdraw ACL and reject authorization.
- Port does not leave its VLANs within the timeout: release it back to L2 and
  reject authorization.
- Client without a tenant ID on a tenant port: reject authorization.
- While a port is routed, PAC's L2 blocking, which works per MAC in a VLAN,
  no longer applies to it. A second host on the port that has not
  authenticated can send traffic: IPv6 is limited by the isolation ACLs,
  which match addresses rather than authenticated MACs, and IPv4 is not
  filtered by PAC. Restricting a routed port to authenticated MACs is an open
  item.
- Route advertisement failure: keep the client isolated and report degraded
  routing; route export must never be used as the isolation mechanism.
- Session teardown: remove forwarding permissions before deleting address state.
- RADIUS reauthentication: retain the old valid state until the new ACL and
  address are installed, then atomically switch policy where platform support
  permits; otherwise briefly fail closed during transition.

## 10. Dependencies

- PAC/hostapd support to preserve vendor-specific authorization attributes and
  expose them to `pacd`.
- PAC operational state and lifecycle handling for address and ACL resources.
- SONiC ACL manager support for required ingress IPv6 source/destination
  matches, actions, priority, and dynamic entry lifecycle.
- A supported mechanism for programming per-client IPv6 addresses and routes.
- FRR/BGP integration for narrowly scoped summary route origination and
  withdrawal.
- Platform capability and resource reporting for ACL scale.

## 11. Validation plan

Validate successful 802.1X authorization and address generation; same-tenant
communication; cross-tenant drop; permitted non-tenant traffic; source spoofing
drop; invalid/missing RADIUS attribute; duplicate address prevention; multiple
clients on one port; logout, timeout, reauthentication, link flap, daemon restart,
and warm reboot cleanup; ACL exhaustion and unsupported ASIC behavior; summary
route reference counting and withdrawal; route-tag route-map entries shared
across tenants and released after the last route; rejection of conflicting
route tags for one summary; and mixed coexistence with existing
user ACLs. Run the PAC tests on every supported platform class and verify actual
hardware counters/traffic, not only CONFIG_DB state.

## 12. Implementation sequence

1. Extend the hostapd authorization message and PAC session model with the
   configured tenant attribute.
2. Add profile schema, validation, and deterministic IPv6 address generation.
3. Add PAC-to-ACL-manager session rules and fail-closed authorization ordering.
4. Add address installation and teardown integration.
5. Add summarize-out summary reference counting, route tags, and bgpcfgd
   custom-tag route-map entries.
6. Add operational show state, restart reconciliation, and platform capability
   checks.
7. Add unit, integration, scale, and hardware traffic tests before enabling the
   feature by default on any platform.
