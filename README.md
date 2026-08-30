# TetherM3

**Modern rootless tethering and network-source control for jailbroken iOS.**

TetherM3 is a rootless-compatible reimplementation and modernization of the classic TetherMe concept, designed for current jailbreak environments and newer iOS networking architectures.

The project focuses on giving users explicit control over how tethered devices reach the network, including support for sharing cellular, Wi-Fi, and compatible VPN-backed connections.

> **Project status:** Active development / security research
> **Platform:** iOS / iPadOS
> **Environment:** Rootless jailbreaks
> **Architecture:** arm64 / arm64e where supported

---

## Overview

Traditional iOS tethering primarily shares the device's cellular Internet connection using Apple's Personal Hotspot infrastructure.

TetherM3 expands that model by introducing configurable upstream network selection.

The goal is simple:

```text
Tethered Client
      |
      v
   iPhone
      |
      v
Selected Network Source
      |
      +--> Cellular
      |
      +--> Wi-Fi
      |
      +--> VPN
      |
      +--> Tor/VPN-compatible tunnel
```

Rather than treating tethering as a fixed cellular-only feature, TetherM3 treats the active upstream connection as a selectable network resource.

---

# Features

## Rootless Jailbreak Support

TetherM3 is designed specifically around modern rootless jailbreak environments.

The implementation should avoid assumptions associated with legacy rootful jailbreaks, including hard-coded filesystem paths such as:

```text
/Library
/usr/lib
/usr/bin
```

Runtime paths and dependencies should use the appropriate jailbreak root prefix whenever required.

Typical environments may include:

* Dopamine
* palera1n rootless
* ElleKit-based tweak injection
* RootHide-compatible environments where practical
* Future rootless jailbreak frameworks

Compatibility varies by jailbreak and iOS release.

---

## Override Data Source

TetherM3 restores and expands the classic **Override Data Source** concept.

Available upstream modes may include:

```text
Automatic
Cellular
Wi-Fi
VPN
```

### Automatic

Allows iOS to determine the normal upstream network.

### Cellular

Forces tethered clients toward the active cellular data connection when supported.

Example interfaces may include:

```text
pdp_ip0
pdp_ip1
pdp_ip*
```

### Wi-Fi

Allows an iPhone connected to Wi-Fi to share that connection with a tethered client where the underlying iOS configuration supports it.

### VPN

Attempts to route tether-originated traffic through the currently active VPN path.

Potential tunnel interfaces include:

```text
utun0
utun1
utun2
...
```

TetherM3 must dynamically identify the correct tunnel rather than assuming a specific `utun` interface number.

---

# VPN Tethering

VPN tethering is one of TetherM3's primary improvements.

The intended topology is:

```text
Laptop / Tablet / Client
          |
          v
   USB / Wi-Fi / Bluetooth
          |
          v
        iPhone
          |
          v
 TetherM3 Forwarding Layer
          |
          v
     VPN Interface
        utunN
          |
          v
     VPN Provider
          |
          v
 Cellular / Wi-Fi Underlay
          |
          v
       Internet
```

The physical cellular or Wi-Fi interface remains the VPN's underlying transport.

TetherM3's responsibility is to ensure that tether-client traffic enters the VPN path **before** encrypted VPN transport leaves through the physical network.

---

## VPN Compatibility

TetherM3 is intended to work with VPN technologies that expose compatible iOS networking interfaces.

Potential targets include:

* WireGuard-based VPNs
* OpenVPN-based VPNs
* IKEv2
* IPSec
* NetworkExtension Packet Tunnel providers
* Privacy VPN applications
* Tor-backed VPN applications
* Custom research VPN implementations

Compatibility depends heavily on how the specific VPN provider handles forwarded traffic.

A VPN displaying as "Connected" does not automatically guarantee that tethered traffic is traveling through it.

TetherM3 therefore aims to verify the actual egress path.

---

# Tor and Privacy Network Support

TetherM3 can support Tor-based applications when the application presents its networking layer through an iOS-compatible VPN or packet-tunnel interface.

The preferred architecture is:

```text
Tethered Client
      |
      v
TetherM3
      |
      v
VPN / Packet Tunnel
      |
      v
Tor Provider
      |
      v
Tor Network
```

TetherM3 does not require direct manipulation of Tor internals when a compatible packet tunnel is available.

This keeps Tor routing isolated from the tethering implementation and reduces provider-specific logic.

---

# VPN Leak Protection

TetherM3 is intended to provide optional safeguards against accidental VPN bypass.

Potential protection mechanisms include:

### Require VPN for Tethering

When enabled:

```text
VPN Connected
    -> Tethering Allowed

VPN Lost
    -> Tether Internet Blocked

VPN Restored
    -> Tunnel Rediscovered
    -> Routing Revalidated
    -> Tethering Resumed
```

The system should never silently fall back to cellular when this mode is enabled.

---

## IPv6 Protection

IPv6 must be treated independently from IPv4.

A configuration such as:

```text
IPv4 -> VPN
IPv6 -> Cellular
```

would constitute a privacy leak.

TetherM3 should therefore expose states such as:

```text
IPv4: VPN Protected
IPv6: VPN Protected
```

or:

```text
IPv4: VPN Protected
IPv6: Blocked
```

when the VPN does not provide IPv6 connectivity.

---

## DNS Protection

TetherM3 should also verify DNS behavior.

Potential states include:

```text
DNS: VPN
DNS: Local
DNS: Physical Network
DNS: Unknown
```

Optional DNS leak protection may prevent tethered clients from using carrier or Wi-Fi resolvers outside the selected VPN path.

---

# Dynamic Interface Discovery

Modern iOS devices can create multiple virtual interfaces.

TetherM3 must never assume:

```text
utun0 == VPN
```

Instead, it should dynamically inspect:

* interface state
* interface index
* IPv4 addresses
* IPv6 addresses
* point-to-point flags
* route ownership
* tunnel routes
* traffic counters
* default routes
* NetworkExtension state where accessible

A conceptual interface inventory may include:

```text
InterfaceInventory

├── Cellular
│   └── pdp_ip*
│
├── Wi-Fi
│   └── en*
│
├── VPN
│   └── utun*
│
├── Tethering
│   ├── bridge*
│   ├── ap*
│   └── USB-related interfaces
│
└── Other
```

---

# Routing Verification

TetherM3 should not report that VPN tethering is active merely because iOS reports that a VPN is connected.

Verification should distinguish between:

```text
Tethered Client Traffic
        |
        v
      utunN
```

and:

```text
VPN Encrypted Transport
        |
        v
       en0
```

or:

```text
VPN Encrypted Transport
        |
        v
     pdp_ip0
```

The latter is expected because the VPN itself needs Wi-Fi or cellular connectivity.

---

# Connection Diagnostics

TetherM3 aims to provide visibility into the current network topology.

Example diagnostic information:

```text
TetherM3 Routing Diagnostics

Downstream Interface:
    bridge100

Requested Upstream:
    VPN

Detected VPN:
    utun4

VPN Mode:
    Full Tunnel

Physical Underlay:
    pdp_ip0

IPv4:
    Protected

IPv6:
    Protected

DNS:
    VPN

Tether -> VPN:
    Verified

Fallback:
    Disabled

Leak Status:
    None Detected
```

The exact interface names vary by iOS release and device.

---

# Full-Tunnel and Split-Tunnel Detection

VPN configurations may be either:

```text
FULL TUNNEL
```

or:

```text
SPLIT TUNNEL
```

A full tunnel typically provides routing for:

```text
0.0.0.0/0
::/0
```

A split tunnel may only route specific networks.

TetherM3 should expose this distinction so a user is not given a false indication that all tethered traffic is protected.

Example:

```text
VPN
Interface: utun4
Mode: Split Tunnel
Tether Routing: Partial
```

---

# Supported Tethering Methods

The long-term goal is to support Apple's standard tethering transports:

* USB
* Wi-Fi Personal Hotspot
* Bluetooth PAN

Actual support depends on jailbreak capabilities and the networking implementation used by the installed iOS version.

USB tethering is generally the preferred development and diagnostic target because the downstream network relationship is easier to inspect.

---

# Settings

Example TetherM3 preferences:

```text
TetherM3
────────────────────────

Tethering
    Enabled

Override Data Source
    Automatic
    Cellular
    Wi-Fi
    VPN

VPN Protection
    Require VPN for Tethering        ON
    Block IPv6 VPN Bypass            ON
    Block DNS Bypass                 ON
    Allow Physical Fallback          OFF

Diagnostics
    Active VPN                       utun4
    VPN Mode                         Full Tunnel
    Underlay                         Cellular
    Tether Routing                   Verified
```

---

# Architecture

A maintainable implementation should separate discovery, routing, policy, and UI responsibilities.

Suggested components:

```text
TetherM3
│
├── InterfaceInventory
│
├── VPNPathResolver
│
├── UpstreamSelector
│
├── RoutingController
│
├── NATController
│
├── DNSController
│
├── IPv6Controller
│
├── LeakProtectionController
│
├── NetworkStateMonitor
│
├── RoutingVerifier
│
└── Preferences
```

This prevents networking behavior from becoming tightly coupled to the Settings UI.

---

# State Machine

VPN-backed tethering should use an explicit state machine.

```text
DISABLED
    |
    v
DISCOVERING
    |
    v
CONFIGURING
    |
    v
VERIFYING
    |
    v
ACTIVE
```

Failure states:

```text
VPN_LOST

BLOCKED

ROUTING_FAILED

DNS_LEAK

IPV6_LEAK

ERROR
```

Tethering should only be reported as VPN-protected once verification succeeds.

---

# Network Changes

iOS networking changes dynamically.

TetherM3 must handle events such as:

* Wi-Fi connection changes
* Cellular PDP changes
* VPN connection
* VPN disconnection
* VPN reconnection
* VPN server change
* new `utun` allocation
* route-table changes
* USB tether connection
* Personal Hotspot activation
* interface disappearance
* device sleep/wake

A VPN that was previously:

```text
utun3
```

may reconnect as:

```text
utun6
```

Interface identifiers must therefore be rediscovered rather than permanently cached.

---

# Safety Requirements

TetherM3 should avoid destabilizing the host iPhone's network configuration.

The implementation should not unnecessarily replace the global system default route.

The intended goal is:

```text
Select the tethering upstream
```

rather than:

```text
Rewrite the entire phone's networking configuration
```

Special care must also be taken to avoid VPN routing loops.

Incorrect:

```text
VPN Provider
    ->
VPN Interface
    ->
VPN Provider
    ->
VPN Interface
```

Correct:

```text
Tether Client
    ->
VPN Interface
    ->
VPN Provider
    ->
Physical Underlay
    ->
Internet
```

---

# Rootless Filesystem Considerations

Do not assume legacy jailbreak paths.

Where appropriate, resolve the jailbreak prefix dynamically.

Possible rootless layouts may place tweak components beneath paths resembling:

```text
/var/jb/
```

Potential package locations may therefore resolve to paths such as:

```text
/var/jb/Library/MobileSubstrate/DynamicLibraries/
/var/jb/Library/PreferenceBundles/
/var/jb/usr/lib/
```

Exact locations depend on the jailbreak and injection framework.

Avoid embedding device-specific paths in source code.

---

# Development Principles

TetherM3 development should prioritize:

1. Correct routing before UI behavior.
2. Dynamic interface discovery.
3. Explicit network-state tracking.
4. IPv4 and IPv6 parity.
5. DNS leak prevention.
6. VPN-loss handling.
7. Fail-closed behavior where requested.
8. Minimal modification of global iOS routing.
9. Compatibility across VPN providers.
10. Clear diagnostic visibility.

---

# Testing

At minimum, test the following matrix:

| iPhone Upstream | VPN             | TetherM3 Selection | Expected            |
| --------------- | --------------- | ------------------ | ------------------- |
| Cellular        | Off             | Cellular           | Cellular            |
| Wi-Fi           | Off             | Wi-Fi              | Wi-Fi               |
| Cellular        | On              | VPN                | VPN over cellular   |
| Wi-Fi           | On              | VPN                | VPN over Wi-Fi      |
| Cellular        | Tor VPN         | VPN                | Tor VPN             |
| Wi-Fi           | Tor VPN         | VPN                | Tor VPN             |
| Cellular        | VPN disconnects | VPN + Require VPN  | Block               |
| Wi-Fi           | VPN reconnects  | VPN                | Rediscover tunnel   |
| Cellular IPv6   | IPv4-only VPN   | VPN                | IPv6 blocked        |
| Any             | Split tunnel    | VPN                | Report split tunnel |

---

# VPN Acceptance Test

VPN source override should not be considered operational until the following works:

```text
1. Connect iPhone to cellular or Wi-Fi.

2. Connect a VPN.

3. Select:
       Override Data Source -> VPN

4. Connect a laptop through USB or Personal Hotspot.

5. Generate Internet traffic from the laptop.

6. Verify tether traffic enters the VPN tunnel.

7. Verify the laptop's public IP corresponds to VPN egress.

8. Verify DNS does not bypass the VPN.

9. Verify IPv6 does not bypass the VPN.

10. Disconnect the VPN.

11. Confirm tether connectivity is blocked when
    "Require VPN for Tethering" is enabled.

12. Reconnect the VPN.

13. Rediscover the new tunnel interface.

14. Revalidate routing.

15. Resume tether connectivity.
```

---

# Logging

Debug builds should provide structured routing diagnostics.

Recommended fields:

```text
timestamp
tether_interface
requested_upstream
selected_upstream
vpn_interface
vpn_mode
physical_underlay
address_family
dns_path
route_result
verification_result
error_code
```

Do not log packet payloads by default.

---

# Privacy

TetherM3 should operate locally on the device.

Network diagnostics should not require telemetry or remote collection.

If optional external IP verification is implemented, users should be informed when an external endpoint is contacted.

No tethered traffic content should be collected.

---

# Security

TetherM3 modifies sensitive networking behavior on a jailbroken iOS device.

Users should understand that:

* jailbreaking weakens portions of the normal iOS security model;
* behavior may change significantly between iOS versions;
* private frameworks and undocumented interfaces can change without notice;
* VPN applications may implement forwarding differently;
* incorrect routing configuration can interrupt device networking;
* VPN compatibility cannot be guaranteed universally.

Use TetherM3 only on devices you own or are explicitly authorized to administer.

---

# Compatibility

Compatibility depends on:

```text
iOS version
jailbreak
tweak injection framework
device architecture
Personal Hotspot implementation
VPN provider
VPN protocol
NetworkExtension configuration
```

A compatibility matrix should be maintained as testing expands.

Example:

| Environment       | Status             |
| ----------------- | ------------------ |
| Dopamine rootless | Testing            |
| ElleKit           | Testing            |
| arm64e            | Testing            |
| USB tethering     | Development        |
| Wi-Fi tethering   | Development        |
| Cellular override | Development        |
| Wi-Fi override    | Development        |
| VPN override      | Active Development |
| Tor VPN           | Experimental       |

---

# Project Goals

TetherM3 is not intended to simply reproduce a legacy tweak byte-for-byte.

The project aims to modernize the concept for contemporary iOS networking.

Major goals include:

```text
Rootless Compatibility

Modern iOS Support

VPN Source Override

Tor/VPN Compatibility

Dynamic Tunnel Discovery

IPv6 Safety

DNS Leak Protection

VPN Kill Switch

Routing Verification

Network Diagnostics

Cleaner Architecture

Improved Reliability
```

---

# Non-Goals

TetherM3 is not intended to:

* provide unauthorized access to carrier networks;
* bypass carrier billing or subscription requirements;
* defeat network access controls;
* interfere with third-party network infrastructure;
* modify VPN providers for which the user lacks authorization;
* conceal abusive or unauthorized network activity.

The project's focus is device-owner network control, interoperability, research, and tethering functionality.

---

# Contributing

Contributions related to the following areas are particularly useful:

* iOS routing research
* rootless jailbreak compatibility
* NetworkExtension behavior
* VPN interface discovery
* Personal Hotspot internals
* IPv6 forwarding
* DNS routing
* NAT behavior
* USB tethering
* VPN compatibility testing
* Settings UI
* automated regression testing

When submitting changes, include:

```text
iOS version
device
jailbreak
jailbreak version
VPN provider/protocol
tethering method
expected behavior
actual behavior
reproduction steps
```

Avoid submitting logs containing personal IP addresses, VPN credentials, authentication tokens, or other secrets.

---

# Disclaimer

TetherM3 is experimental software intended for legitimate research, interoperability, development, and administration of devices under the user's control.

It relies on behavior that may include private or undocumented iOS functionality and may stop working after operating-system, jailbreak, or VPN-provider updates.

There is no guarantee of connectivity, privacy, anonymity, VPN compatibility, carrier compatibility, or availability.

Use at your own risk.

---

# Credits

TetherM3 is inspired by the functionality and usability concepts introduced by the original **TetherMe** tweak.

TetherM3 is an independent modernization effort intended to explore how advanced tethering and selectable upstream networking can be implemented in contemporary rootless iOS environments.

Respect the intellectual property and licensing terms of upstream projects.

---

# TetherM3

### Rootless. Modern. Observable. Selectable.

```text
Your device.
Your connection.
Your upstream.
```
