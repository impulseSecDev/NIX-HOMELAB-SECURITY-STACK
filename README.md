# Homelab Security Stack

> Production-grade, fully declarative security monitoring infrastructure built on NixOS, ELK, Wazuh, and WireGuard. All services run as real daily-use systems — not sandboxed demos.

---

## Overview

A self-hosted security stack designed around defense-in-depth, zero trust networking, VLAN segmentation, and complete infrastructure-as-code. Every service runs on its own dedicated host. Excluding the Headscale VPS (Ubuntu) and the OPNsense router/firewall, every host is declared in version-controlled NixOS configuration. No configuration drift, no undocumented changes — the repository is the system.

Built for hands-on learning and portfolio demonstration across network security, SIEM engineering, host-based intrusion detection, network segmentation, encrypted private networking, and secrets management.

---

## Infrastructure

| Host                                                         | OS       | Role                                                                                                         |
| ------------------------------------------------------------ | -------- | ------------------------------------------------------------------------------------------------------------ |
| [Daily Driver](https://github.com/impulseSecDev/dailyDriver) | NixOS    | Primary workstation and admin node, OpenWebUI and llama.cpp (local LLM interface and inference)             |
| OPNsense Router                                              | OPNsense | Primary gateway and firewall, CrowdSec, inter-VLAN default deny, Tailscale exit node, Vaultwarden ingress inspection |
| VPS                                                          | Ubuntu   | Nginx (Hardened), Headscale, Suricata IPS, WireGuard hub, Fail2ban, layer 4 forwarding                       |
| [ELK Host](https://github.com/impulseSecDev/ELK-NIXVM)       | NixOS    | Elasticsearch, Kibana, Fluent Bit — SIEM core                                                                |
| [Wazuh Host](https://github.com/impulseSecDev/WAZUH-NIXVM)   | NixOS    | Wazuh Manager, Fluent Bit — HIDS core                                                                        |
| [Vaultwarden Host](https://github.com/impulseSecDev/VW-NIXVM) | NixOS   | Self-hosted password manager, family/friends access                                                          |
| NetBox Host                                                  | NixOS    | NetBox — source of truth for devices, IPs, VLANs and subnets                                                 |
| Laptop                                                       | NixOS    | Mobile workstation, log shipping over WireGuard                                                              |
| Netgear Switch                                               | —        | LAN switching                                                                                                |
| Nighthawk AP                                                 | —        | Wireless access                                                                                              |

---

## Network Architecture

### Home Network Segmentation — VLANs

The home network is divided into per-function VLANs, each on its own small subnet (/29 per VLAN, /26 for guests). Every former VM now runs on its own dedicated host, and each host lives in the VLAN that matches its role, so a compromise in one segment is contained away from the others. OPNsense sits in the dedicated management VLAN as the router/firewall for the segmented network.

| VLAN | Name                   | Hosts                                  | Subnet |
| ---- | ---------------------- | -------------------------------------- | ------ |
| 10   | Control Center         | Daily Driver                           | /29    |
| 20   | Guests                 | Guest devices (up to 20)               | /26    |
| 30   | Monitoring and Logging | ELK Host, Wazuh Host                   | /29    |
| 40   | Auth                   | Vaultwarden Host                       | /29    |
| 50   | NetBox                 | NetBox Host                            | /29    |
| 60   | Cloud                  | —                                      | /29    |
| 70   | Micro Services         | —                                      | /29    |
| 80   | Coordinator            | Reserved                               | /29    |
| 90   | P2P                    | Reserved                               | /29    |
| 100  | Business               | Reserved                               | /29    |
| 110  | Backups                | Reserved                               | /29    |
| 200  | Management             | OPNsense router/firewall, access point | /29    |

#### Inter-VLAN Hardening

OPNsense enforces strict isolation between segments. A firewall group designates every VLAN as private, with a default deny for access from any VLAN into any other. The native VLAN (VLAN 1, LAN) is not enabled on any switch port and has no access to any other VLAN, so an untagged or misconfigured port reaches nothing.

### NetBox — Source of Truth

NetBox is the authoritative inventory for the entire network. It holds all device information, IP addresses, LANs, VLANs and subnets, and is consumed directly by the deployment workflow when new machines are provisioned — addressing and placement are looked up from NetBox rather than hand-assigned, keeping the declarative NixOS configuration and the network documentation consistent.

### Zero Trust Mesh — Tailscale + Headscale

All NixOS hosts connect to a private Tailscale mesh coordinated by a self-hosted Headscale server on the VPS. OPNsense participates in the mesh as both a Headscale node and a Tailscale exit node. All inter-host admin traffic, SSH access, and internal service communication routes exclusively over the encrypted mesh. ACL policy enforces access control — only authenticated tailnet members can reach services or initiate SSH sessions.

```
Daily Driver     ─┐
OPNsense         ─┤
ELK Host         ─┤
Wazuh Host       ─┤
Vaultwarden Host ─┼── Headscale
NetBox Host      ─┤
Cloud Host       ─┤
Sprint Host      ─┤
Laptop           ─┘
```

### Headscale ACL Policy

Tag-based ACL policy enforced across the tailnet. `tag:admin` machines have unrestricted access to all destinations including LAN and internet via the exit node. SSH requires re-authentication every 6 hours. A separate restricted user is limited to HTTPS (port 443) on two specific service nodes and nothing else. Usernames and node addresses below are anonymized.

```
{
  "tagOwners": {
    "tag:server": ["user1@"],
    "tag:admin": ["user1@"],
    "tag:exit-node": ["user1@"],
    "tag:gf": ["user1@", "user2@"]
  },
  "acls": [
    {
      "action": "accept",
      "src": ["tag:admin"],
      "dst": ["*:*"]
    },
    {
      "action": "accept",
      "src": ["user1@"],
      "dst": ["*:*"]
    },
    {
      "action": "accept",
      "src": ["tag:admin", "user1@"],
      "dst": ["0.0.0.0/0:*", "::/0:*", "192.168.1.0/24:*"]
    },
    {
      "action": "accept",
      "src": ["user2@"],
      "dst": ["100.64.0.x:443", "100.64.0.y:443"]
    }
  ],
  "ssh": [
    {
      "action": "check",
      "check_period": "6h",
      "src": ["tag:admin"],
      "dst": ["tag:server"],
      "users": ["autogroup:nonroot"]
    },
    {
      "action": "check",
      "check_period": "6h",
      "src": ["tag:admin"],
      "dst": ["tag:admin", "tag:exit-node"],
      "users": ["autogroup:nonroot"]
    }
  ]
}
```

### Log Shipping — WireGuard wg0 (10.10.10.0/24)

A dedicated WireGuard interface handles all log shipping traffic, deliberately separated from admin channels. The hosts, the laptop, and OPNsense maintain encrypted tunnels to the VPS hub. OPNsense logs and Wazuh agent data ship over this tunnel to the dedicated ELK and Wazuh hosts. The VPS forwards traffic between peers using the WireGuard routing table — no traffic is routable outside the `10.10.10.0/24` subnet.

```
Daily Driver     ─┐
OPNsense         ─┤
ELK Host         ─┤
Wazuh Host       ─┼── VPS hub
Vaultwarden Host ─┤
Laptop           ─┘
```

### SSH Access — WireGuard wg2

A dedicated WireGuard interface exclusively for SSH access to the VPS, isolated from log shipping and service traffic. No port 22 exposed publicly.

### Vaultwarden Access Paths

Two distinct access paths, both HTTPS end-to-end with TLS terminated on the Vaultwarden host itself. The Vaultwarden host is never directly internet-exposed in either path. The VPS routes external traffic at layer 4 only — it never terminates TLS and cannot read the traffic it forwards — and holds no CA authentication tokens. External traffic is inspected on the OPNsense router/firewall, where CrowdSec applies its blocklists, before it reaches the host.

```
# Tailnet members
Device → Tailscale → Vaultwarden Host Nginx → Vaultwarden

# External users (friends/family)
Device → HTTPS → VPS (layer 4 routing) → OPNsense (inspection) → Vaultwarden Host Nginx (TLS termination) → Vaultwarden
```

### TLS Certificate Management

Each service host provisions its own wildcard TLS certificate via the NixOS `security.acme` module using Cloudflare DNS-01 challenge validation. Covers `*.mesh.mydomain.com`. Fully automated renewal — no manual certificate management. Certificate authority credentials live only on the hosts that terminate TLS; the VPS stores none.

---

## Security Stack

| Layer                    | Tool                         | Mode                                                                                                                                                 |
| ------------------------ | ---------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| Network IPS              | Suricata 7.0.3               | NFQUEUE active blocking on VPS public interface, et/open ruleset                                                                                     |
| Host IPS                 | Suricata                     | NFQ mode on ELK, Wazuh, and Vaultwarden hosts — daily rule updates                                                                                   |
| Host IDS                 | Wazuh 4.14.3                 | Agents on all hosts including OPNsense, FIM, rootkit detection, SCA                                                                                  |
| SIEM                     | Elasticsearch + Kibana 9.3.2 | Centralized log aggregation, KQL queries, custom dashboards                                                                                          |
| **Network Segmentation** | **VLANs + OPNsense**         | **Per-function VLANs on /29 subnets, one dedicated host per service — compromise containment**                                                       |
| **Inter-VLAN Isolation** | **OPNsense firewall group**  | **All VLANs private with default deny into other VLANs; native VLAN 1 unused on switch ports and isolated**                                          |
| **Ingress Inspection**   | **OPNsense**                 | **Inbound Vaultwarden traffic from the VPS is inspected at the firewall before reaching the host**                                                   |
| **Threat-Intel Blocking** | **CrowdSec (OPNsense)**     | **Blocklist decisions at the firewall, including a list for a newly disclosed PHP vulnerability**                                                    |
| Edge Hardening           | Nginx (VPS)                  | Global rate limiting, shared WebSocket upgrade maps, hidden file denial (`/\.`), and error code silencing (400/401/403/404/500 → 444)                |
| Anti-Recon               | Nginx Blackhole              | Default 443 server block using snakeoil certs returns 444 on all requests and all error codes — no SNI/domain leakage, no chatter on probes          |
| Backend Security         | Upstream SSL                 | Full proxy SSL verification (`proxy_ssl_verify on`) for secure backend communication on proxied services                                             |
| Firewall Hardening       | iptables (VPS)               | Explicit DROP rules for IKE/IPsec (UDP 500/4500), ICMP echo, and ICMP timestamp — silent drop before kernel generates ICMP unreachable responses     |
| Log shipping             | Fluent Bit                   | Structured shipping with custom Lua parsers, WireGuard transport                                                                                     |
| Auth blocking            | Fail2ban                     | VPS, ELK Host, Wazuh Host, and Vaultwarden Host — nftables backend, `blocktype=drop` (no REJECT chatter)                                              |
| Private mesh             | Tailscale + Headscale        | Zero trust, ACL-enforced, SSH via tailnet identity; OPNsense as exit node                                                                            |
| Secrets management       | sops-nix                     | Encrypted secrets in version-controlled NixOS config                                                                                                 |
| Source of truth          | NetBox                       | Devices, IPs, VLANs and subnets feed the deployment of new machines                                                                                  |
| Reverse proxy            | Nginx                        | Per-service TLS termination on service hosts, hardened proxy and layer 4 forwarding on the VPS, exploit probe blocking                               |

---

## Channel Separation

A deliberate design decision to separate network traffic by function rather than routing everything over a single interface:

| Channel             | Transport      | Traffic                                                      |
| ------------------- | -------------- | ------------------------------------------------------------ |
| Admin / SSH         | Tailscale mesh | SSH, file transfers, service access                          |
| Log shipping        | WireGuard wg0  | Fluent Bit, Wazuh agent comms, OPNsense logs                 |
| Vaultwarden routing | WireGuard wg1  | Public → VPS (layer 4) → OPNsense → Vaultwarden Host         |
| VPS SSH             | WireGuard wg2  | SSH access to VPS only                                       |

This ensures a compromise of one channel cannot affect others. WireGuard's ChaCha20-Poly1305 authenticated encryption means the VPS hub cannot read or manipulate log traffic in transit even though it forwards packets between peers.

---

## VPS Firewall Hardening

### Background

Suricata flagged an inbound IKE/IPsec probe (`invalid_proposal`) on UDP 500. Investigation revealed the VPS was responding to port scans with ICMP unreachable responses rather than silently dropping traffic — leaking host presence to scanners.

### Approach

Explicit DROP rules inserted at position 1 of the INPUT chain intercept traffic before the kernel UDP stack can generate ICMP unreachable responses. All rules persisted via `netfilter-persistent`.

```
# Silent drop — IKE/IPsec probes (UDP 500/4500)
iptables -I INPUT 1 -p udp --dport 500 -j DROP
iptables -I INPUT 1 -p udp --dport 4500 -j DROP

# Silent drop — OS fingerprinting via ICMP timestamp
iptables -I INPUT 1 -p icmp --icmp-type timestamp-request -j DROP

# Silent drop — ICMP echo (ping)
iptables -I INPUT 1 -p icmp --icmp-type echo-request -j DROP
```

### Known Limitations

The VPS remains discoverable via TCP SYN probes on ports 80/443 — unavoidable while running a public web server. Suricata remains in IPS mode via NFQUEUE with `exception-policy: auto` (drop-flow in IPS mode).

---

## Fail2ban — Silent Banning

Fail2ban was previously banning via `REJECT with icmp-port-unreachable` (iptables-multiport action), generating response traffic toward attackers. Switched to the nftables backend with `blocktype=drop` across all jails — banned IPs now receive no response.

```
# /etc/fail2ban/jail.local [DEFAULT]
banaction = nftables[blocktype=drop]
banaction_allports = nftables[type=allports,blocktype=drop]
```

Applied consistently in both `jail.local` and `jail.d/defaults-debian.conf`.

---

## Nginx — Anti-Recon & Error Silencing

### Blackhole Default Server

The default HTTPS server block returns Nginx's connection-close code (`444`) for all requests — no TLS handshake, no response body, no domain or SNI leakage during IP-range scans. Snakeoil certificates are presented to complete enough of the handshake to trigger the drop.

All error codes (400, 401, 403, 404, 500) are mapped to `444` on the blackhole server, eliminating response chatter from malformed or probe requests:

```
server {
    listen 443 ssl default_server;
    ssl_certificate     /path/to/snakeoil.crt;
    ssl_certificate_key /path/to/snakeoil.key;

    error_page 400 401 403 404 500 = @blackhole;
    location @blackhole { return 444; }
    location / { return 444; }
}
```

This same error-silencing pattern is applied to the default server config, ensuring no upstream errors produce informative responses to unauthenticated scanners.

Vaultwarden traffic is the exception to Nginx proxying on the VPS: it is forwarded at layer 4 and never terminated there. TLS is terminated on the Vaultwarden host after OPNsense inspection.

---

## Fluent Bit Log Parsing

Custom Lua parsers enrich log events beyond what standard parsers provide:

- **Tailscale SSH parser** — extracts `tailscale_src_ip`, sets `tailscale_ssh: true` and `event_type: "tailscale_login"` from systemd journal entries on all NixOS hosts
- **Fail2ban parser** — extracts `jail`, `action`, and `src_ip` from fail2ban log lines into queryable Kibana fields
- **Suricata JSON parser** — full eve.json parsing including alert, flow, dns, http, tls, and stats event types
- **OPNsense parser** — structured parsing of OPNsense firewall and system logs shipped over WireGuard

Per-machine Elasticsearch users with scoped index permissions — each host ships logs under its own credentials, limiting blast radius in the event of credential compromise.

---

## Kibana Dashboards

Custom dashboards built manually in Kibana against real production data:

| Dashboard                       | Status        |
| ------------------------------- | ------------- |
| Wazuh Alerts & MITRE ATT&CK     | ✅ Complete    |
| SSH Authentication Events       | ✅ Complete    |
| Suricata IDS/IPS Alerts         | ✅ Complete    |
| System/Journal Logs Overview    | ✅ Complete    |
| Fail2ban & VPS Security         | ✅ Complete    |
| Fluent Bit Log Ingestion Health | ✅ Complete    |
| Nginx Attack Traffic            | ✅ Complete    |
| Tailscale Activity              | 📋 Planned     |
| Vaultwarden Access              | ✅ Complete    |
| Sudo Activity                   | 🔄 In progress |
| OPNsense Firewall               | 📋 Planned     |

---

## Why NixOS

All hosts run NixOS for reproducibility and auditability. The entire system state is declared in version-controlled configuration files, and NetBox supplies the device, IP, VLAN and subnet data used when deploying new machines. Rebuilding any host from scratch produces an identical result — no configuration drift, no undocumented changes, no snowflake servers.

Docker containers for software not in nixpkgs (Elasticsearch, Kibana, Wazuh) are declared via `virtualisation.oci-containers` — managed by systemd, fully reproducible despite running in containers.

The VPS runs Ubuntu and is a candidate for future NixOS + impermanence migration.

> *Why NixOS? In the end, the infrastructure is the documentation.*

---

## Tech Stack

### Core Infrastructure

`NixOS` `Ubuntu` `OPNsense` `NetBox` `WireGuard` `Tailscale` `Headscale` `Nginx` `Docker`

### Security & Monitoring

`Elasticsearch` `Kibana` `Fluent Bit` `Wazuh` `Suricata IDS/IPS` `CrowdSec` `Fail2ban` `sops-nix`

### Networking & Access

`WireGuard` `Tailscale` `Zero Trust` `ACL Policy` `VLAN Segmentation` `Default-Deny Firewalling` `Layer 4 Routing` `NFQUEUE` `iptables` `nftables`

### Protocols & Standards

`TLS/HTTPS` `ACME/Let's Encrypt` `DNS-01` `Cloudflare` `KQL` `SIEM` `HIDS` `FIM` `IPAM`

---

## Secrets & Security Notice

All secrets (API keys, passwords, private keys, IP addresses) are managed via sops-nix and excluded from version control. No credentials, private keys, or infrastructure-identifying information are present in this repository.

---

## Status

| Component                 | Status                                                                                      |
| ------------------------- | ------------------------------------------------------------------------------------------- |
| ELK Stack                 | ✅ Production                                                                                |
| Wazuh HIDS                | ✅ Production                                                                                |
| Suricata IPS — VPS        | ✅ Production — NFQUEUE mode, et/open ruleset, 49k+ rules                                    |
| Nginx Hardening           | ✅ Production — Rate limiting, hidden file blocks, SSL verification, error silencing (→ 444) |
| Anti-Scan Blackhole       | ✅ Production — HTTPS default_server returns 444; all error codes silenced                   |
| Firewall Hardening        | ✅ Production — IKE/IPsec DROP, ICMP echo DROP, ICMP timestamp DROP                          |
| Fail2ban                  | ✅ Production — nftables backend, blocktype=drop                                             |
| WireGuard tunnels         | ✅ Production                                                                                |
| Tailscale mesh            | ✅ Production                                                                                |
| OPNsense Router           | ✅ Production — Primary gateway, Headscale node, Tailscale exit node                         |
| VLAN Segmentation         | ✅ Production — Per-function VLANs, one dedicated host per service                           |
| Inter-VLAN Isolation      | ✅ Production — Private firewall group, default deny, native VLAN 1 unused                   |
| CrowdSec                  | ✅ Production — OPNsense, includes blocklist for new PHP vulnerability                       |
| NetBox                    | ✅ Production — Source of truth for deployments                                              |
| Vaultwarden               | ✅ Production — daily use, layer 4 routed via OPNsense, TLS terminated on host               |
| Headscale ACL Policy      | ✅ Production — Tag-based, exit node routing, 6h SSH re-auth                                 |
| sops-nix                  | ✅ Production                                                                                |
| Fluent Bit parsers        | ✅ Production — Tailscale SSH, fail2ban, Suricata, OPNsense                                  |
| Access Logging Audit      | ✅ Completed — Verified vhost log consistency                                                |
| OPNsense log shipping     | ✅ Production — WireGuard transport to ELK                                                   |
| Wazuh custom rules        | 📋 Planned                                                                                   |
| OPNsense Kibana Dashboard | 📋 Planned                                                                                   |
| Zeek — Daily Driver       | 📋 Planned                                                                                   |
| Attack simulation         | 📋 Planned                                                                                   |
| VPS NixOS migration       | 📋 Planned                                                                                   |
