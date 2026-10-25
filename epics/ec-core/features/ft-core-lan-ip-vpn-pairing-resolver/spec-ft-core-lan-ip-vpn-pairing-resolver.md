---
id: spec-ft-core-lan-ip-vpn-pairing-resolver
type: spec
aico_skills_version: "0.1.2"
epic: "ec-core"
order: 19
title: "Local LAN IP Resolution, VPN Adapter Exclusion & Multi-Interface Pairing"
status: "Completed"
derives_from: ["brain-ec-core", "ADR-0001", "ADR-0004"]
created_at: "2026-10-25T19:10:00.000Z"
---

# Feature Specification: Local LAN IP Resolution, VPN Adapter Exclusion & Multi-Interface Pairing

## 1. Problem Statement & User Value

When a developer operates with an active Virtual Private Network (such as ProtonVPN, WireGuard, OpenVPN, Tailscale, or corporate tunnels), the HostServer's mobile pairing URL generator resolves the wrong IP address:

1. **Default Gateway Hijacking by VPNs**: The current IP detection mechanism (`get_lan_ip`) connects an outbound UDP socket to `8.8.8.8:80`. When a VPN is active, the operating system routing table directs all default internet traffic through the virtual tunnel adapter (e.g. `10.2.0.2` on ProtonVPN), rather than the physical local Wi-Fi or Ethernet adapter (e.g. `192.168.178.190` on `Ethernet 5`).
2. **Unreachable QR Codes on Mobile**: When a mobile phone on the developer's home/office Wi-Fi network scans the desktop QR code, it attempts to connect to the VPN tunnel's internal IP address (`http://10.2.0.2:4242/?token=...`), which is unreachable from external LAN devices. Pairing fails completely.
3. **No Interface Selection or Escape Hatch**: The mobile pairing modal provides no UI to see alternative detected network interfaces or override the IP address if a machine has multiple active adapters (e.g. Wi-Fi, Ethernet, WSL virtual switches).

Accurate local LAN IP discovery with VPN exclusion and interactive adapter selection ensures instant, reliable mobile companion pairing regardless of whether VPN tunnels, virtual switches, or multiple network adapters are active.

---

## In-Scope

- Cross-platform network interface enumeration using `get_if_addrs` in `aico-server`.
- Filtering out loopbacks (`127.0.0.0/8`), link-local APIPA addresses (`169.254.0.0/16`), and virtual/VPN adapters (`vpn`, `proton`, `wireguard`, `nord`, `tailscale`, `tun`, `tap`, `vethernet`, `wsl`, `vbox`, `virtualbox`, `vmware`, `docker`, `bridge`, `bluetooth`, `loopback`, `dummy`).
- Smart RFC 1918 priority ranking: Class C (`192.168.0.0/16`) > Class A (`10.0.0.0/8` non-VPN) > Class B (`172.16.0.0/12` non-virtual) > other IPv4.
- Deterministic environment variable override (`AICO_LAN_IP`).
- Extending `GET /api/v1/auth/pairing` with `candidate_ips: Vec<LanCandidate>`.
- Updating `MobilePairingModal.tsx` with an interactive network adapter switcher when multiple candidates exist, dynamically regenerating the QR code SVG and updating the copyable pairing URL.

---

## Out-of-Scope

- Automatic dynamic DNS / mDNS daemon publication (`.local` zero-conf).
- Cloud relay or TURN/STUN traversal across separate internet networks.
- Modifying host operating system routing tables or VPN configuration.

---

## 2. User Stories

1. **US-01** (Priority: P1): As a developer with ProtonVPN or another VPN running on my desktop, I want the Pair Mobile QR code to encode my physical home Wi-Fi/Ethernet LAN IP address (`192.168.x.x`), so that my phone can connect to my desktop without turning off my VPN.
   - *Independent Test*: Simulate active interfaces with a `ProtonVPN` virtual adapter (`10.2.0.2`) alongside an `Ethernet` adapter (`192.168.178.190`); verify `get_lan_ip()` and `GET /api/v1/auth/pairing` select `192.168.178.190` as the preferred `lan_ip`.
2. **US-02** (Priority: P2): As a developer working in a complex multi-network environment (e.g. Ethernet + Wi-Fi + Docker/WSL), I want the Mobile Pairing Modal to show an interface selector when multiple candidate IPs exist, so that I can choose which network interface my phone should connect through.
   - *Independent Test*: Render `MobilePairingModal` with multiple `candidate_ips`; verify the interface selector dropdown appears, switching interfaces changes the selected IP, and the QR code SVG updates reactively.
3. **US-03** (Priority: P2): As a developer running in a container, VM, or custom network, I want to set `AICO_LAN_IP=<ip>` in the environment, so that the pairing URL deterministically uses my designated address without automatic heuristics.
   - *Independent Test*: Set `AICO_LAN_IP=192.168.1.99`; verify `GET /api/v1/auth/pairing` returns `192.168.1.99` as `lan_ip`.

---

## 3. Functional Requirements (EARS)

### Ubiquitous Requirements
- **REQ-001**: The `aico-server` runtime shall enumerate network interface addresses across platforms and rank private IPv4 addresses, prioritizing RFC 1918 Class C addresses (`192.168.0.0/16`) on physical network adapters over virtual or tunnel interfaces.
- **REQ-002**: The `aico-server` runtime shall exclude loopback addresses (`127.0.0.0/8`), link-local APIPA addresses (`169.254.0.0/16`), and interfaces whose names match virtual or VPN patterns (including `vpn`, `proton`, `wireguard`, `nord`, `tailscale`, `tun`, `tap`, `vethernet`, `wsl`, `vbox`, `virtualbox`, `vmware`, `docker`, `bridge`, `bluetooth`, `loopback`, `dummy`) from primary LAN IP selection.

### State-Driven Requirements
- **REQ-003**: While an environment variable `AICO_LAN_IP` is defined and contains a valid IPv4 address, the `aico-server` runtime shall select that address as the primary LAN IP regardless of adapter enumeration or routing tables.
- **REQ-006**: While no physical Class C adapter is detected and no `AICO_LAN_IP` is configured, the `aico-server` runtime shall rank non-VPN Class A (`10.0.0.0/8`) and Class B (`172.16.0.0/12`) private addresses above virtual interfaces, falling back to connect socket probing (`8.8.8.8:80`) and ultimately `127.0.0.1` if no network interfaces are found.

### Event-Driven Requirements
- **REQ-004**: When a client sends `GET /api/v1/auth/pairing`, the HostServer shall return a `PairingResponse` containing the preferred `lan_ip`, the `port`, the pairing `token`, the primary `pairing_url`, and an array `candidate_ips` listing all detected non-loopback IPv4 candidates with their interface names and preferred flags.
- **REQ-005**: When the `MobilePairingModal` receives `candidate_ips` containing multiple network interfaces, the modal shall render an interactive interface selector allowing the user to select any candidate IP, dynamically regenerating the QR code SVG and updating the copyable pairing URL to encode the selected host IP.

---

## 4. Designated Public Test Seams

### Primary Seams:
1. **Network Interface Detection & Pairing API Seam (`crates/aico-server/tests/auth_test.rs` & `crates/aico-server/tests/server_test.rs`)**:
   - Signature: `aico_server::server::detect_lan_ip_candidates()` and `GET /api/v1/auth/pairing`
   - Verifies: VPN adapter exclusion (`REQ-001`, `REQ-002`), `AICO_LAN_IP` override (`REQ-003`), `candidate_ips` response structure (`REQ-004`), and fallback ranking (`REQ-006`).
2. **Mobile Pairing Modal UI Seam (`frontend/tests/MobilePairingAndSync.test.tsx`)**:
   - Signature: `<MobilePairingModal isOpen={true} onClose={...} windowId="win-1" />`
   - Verifies: Interactive interface selector rendering and dynamic QR code / URL updates on adapter selection (`REQ-005`).

---

## 5. Open Clarifications

None. Ratified in [ADR-0001](../../../../adr/0001-host-runtime-and-network-topology.md) and [ADR-0004](../../../../adr/0004-react-frontend-and-reusable-design-system.md).
