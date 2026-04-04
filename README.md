# Homelab Security Stack

> Production-grade, fully declarative security monitoring infrastructure built on NixOS, ELK, Wazuh, and WireGuard. All services run as real daily-use systems — not sandboxed demos.

---

## Overview

A self-hosted security stack designed around defense-in-depth, zero trust networking, and complete infrastructure-as-code. Excluding the Headscale VPS which is Ubuntu, every host is declared in version-controlled NixOS configuration. No configuration drift, no undocumented changes — the repository is the system.

Built for hands-on learning and portfolio demonstration across network security, SIEM engineering, host-based intrusion detection, encrypted private networking, and secrets management.

---

## Infrastructure

| Host | OS | Role |
|---|---|---|
| [Daily Driver](https://github.com/impulseSecDev/dailyDriver) | NixOS | Primary workstation, KVM/QEMU VM host, OpenWebUI (local LLM) |
| VPS | Ubuntu | Nginx (Hardened), Headscale, Suricata IPS, WireGuard hub, Fail2ban |
| [ELK VM](https://github.com/impulseSecDev/ELK-NIXVM) | NixOS | Elasticsearch, Kibana, Fluent Bit — SIEM core |
| [Wazuh VM](https://github.com/impulseSecDev/WAZUH-NIXVM) | NixOS | Wazuh Manager, Fluent Bit — HIDS core |
| [Vaultwarden VM](https://github.com/impulseSecDev/VW-NIXVM) | NixOS | Self-hosted password manager, family/friends access |
| Laptop | NixOS | Mobile workstation, log shipping over WireGuard |

---

## Network Architecture

### Zero Trust Mesh — Tailscale + Headscale

All NixOS hosts connect to a private Tailscale mesh coordinated by a self-hosted Headscale server on the VPS. All inter-host admin traffic, SSH access, and internal service communication routes exclusively over the encrypted mesh. ACL policy enforces access control — only authenticated tailnet members can reach services or initiate SSH sessions.

```
Daily Driver  ─┐
ELK VM        ─┤
Wazuh VM      ─┼── Headscale
Vaultwarden   ─┤
Laptop        ─┘
```

### Log Shipping — WireGuard wg0 (10.10.10.0/24)

A dedicated WireGuard interface handles all log shipping traffic, deliberately separated from admin channels. All VMs and the laptop maintain encrypted tunnels to the VPS hub. The VPS forwards traffic between peers using the WireGuard routing table — no traffic is routable outside the `10.10.10.0/24` subnet.

```
Daily Driver  ─┐
ELK VM        ─┤
Wazuh VM      ─┼── VPS hub
Vaultwarden   ─┤
Laptop        ─┘
```

### SSH Access — WireGuard wg2

A dedicated WireGuard interface exclusively for SSH access to the VPS, isolated from log shipping and service traffic. No port 22 exposed publicly.

### Vaultwarden Access Paths

Two distinct access paths, both HTTPS end-to-end, Vaultwarden VM never directly internet-exposed. Recent hardening includes **Backend SSL Verification** and **Proxy Buffering** optimizations.

```
# Tailnet members
Device → Tailscale → Vaultwarden VM Nginx → Vaultwarden

# External users (friends/family)
Device → HTTPS → VPS Nginx (Hardened) → WireGuard (wg1) → Vaultwarden VM Nginx → Vaultwarden
```

### TLS Certificate Management

Each service VM provisions its own wildcard TLS certificate via the NixOS `security.acme` module using Cloudflare DNS-01 challenge validation. Covers `*.mesh.mydomain.com`. Fully automated renewal — no manual certificate management.

---

## Security Stack

| Layer | Tool | Mode |
|---|---|---|
| Network IPS | Suricata 7.0.3 | NFQUEUE active blocking on VPS public interface, et/open ruleset |
| Host IPS | Suricata | NFQ mode on ELK, Wazuh, and Vaultwarden VMs — daily rule updates |
| Host IDS | Wazuh 4.14.3 | Agents on all hosts, FIM, rootkit detection, SCA |
| SIEM | Elasticsearch + Kibana 8.13.0 | Centralized log aggregation, KQL queries, custom dashboards |
| **Edge Hardening** | **Nginx (VPS)** | **Global rate limiting, shared WebSocket upgrade maps, hidden file denial (`/\.`), and error code silencing (400/401/403/404/500 → 444)** |
| **Anti-Recon** | **Nginx Blackhole** | **Default 443 server block using snakeoil certs returns 444 on all requests and all error codes — no SNI/domain leakage, no chatter on probes** |
| **Backend Security** | **Upstream SSL** | **Full proxy SSL verification (`proxy_ssl_verify on`) for secure backend communication** |
| **Firewall Hardening** | **iptables (VPS)** | **Explicit DROP rules for IKE/IPsec (UDP 500/4500), ICMP echo, and ICMP timestamp — silent drop before kernel generates ICMP unreachable responses** |
| Log shipping | Fluent Bit | Structured shipping with custom Lua parsers, WireGuard transport |
| Auth blocking | Fail2ban | VPS, ELK VM, Wazuh VM, and Vaultwarden VM — nftables backend, `blocktype=drop` (no REJECT chatter) |
| Private mesh | Tailscale + Headscale | Zero trust, ACL-enforced, SSH via tailnet identity |
| Secrets management | sops-nix | Encrypted secrets in version-controlled NixOS config |
| Reverse proxy | Nginx | Per-service TLS termination, public routing via VPS, exploit probe blocking |

---

## Channel Separation

A deliberate design decision to separate network traffic by function rather than routing everything over a single interface:

| Channel | Transport | Traffic |
|---|---|---|
| Admin / SSH | Tailscale mesh | SSH, file transfers, service access |
| Log shipping | WireGuard wg0 | Fluent Bit, Wazuh agent comms |
| Vaultwarden routing | WireGuard wg1 | Public → VPS (Hardened Proxy) → Vaultwarden VM |
| VPS SSH | WireGuard wg2 | SSH access to VPS only |

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

```ini
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

```nginx
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

---

## Fluent Bit Log Parsing

Custom Lua parsers enrich log events beyond what standard parsers provide:

- **Tailscale SSH parser** — extracts `tailscale_src_ip`, sets `tailscale_ssh: true` and `event_type: "tailscale_login"` from systemd journal entries on all NixOS hosts
- **Fail2ban parser** — extracts `jail`, `action`, and `src_ip` from fail2ban log lines into queryable Kibana fields
- **Suricata JSON parser** — full eve.json parsing including alert, flow, dns, http, tls, and stats event types

Per-machine Elasticsearch users with scoped index permissions — each host ships logs under its own credentials, limiting blast radius in the event of credential compromise.

---

## Kibana Dashboards

Custom dashboards built manually in Kibana against real production data:

| Dashboard | Status |
|---|---|
| Wazuh Alerts & MITRE ATT&CK | ✅ Complete |
| SSH Authentication Events | ✅ Complete |
| Suricata IDS/IPS Alerts | ✅ Complete |
| System/Journal Logs Overview | ✅ Complete |
| Fail2ban & VPS Security | ✅ Complete |
| Fluent Bit Log Ingestion Health | ✅ Complete |
| Nginx Attack Traffic | ✅ Complete |
| Tailscale Activity | 📋 Planned |
| Vaultwarden Access | ✅ Complete |
| Sudo Activity | 🔄 In progress |

---

## Why NixOS

All VMs run NixOS for reproducibility and auditability. The entire system state is declared in version-controlled configuration files. Rebuilding any VM from scratch produces an identical result — no configuration drift, no undocumented changes, no snowflake servers.

Docker containers for software not in nixpkgs (Elasticsearch, Kibana, Wazuh) are declared via `virtualisation.oci-containers` — managed by systemd, fully reproducible despite running in containers.

The VPS runs Ubuntu and is a candidate for future NixOS + impermanence migration.

> *Why NixOS? In the end, the infrastructure is the documentation.*

---

## Tech Stack

### Core Infrastructure
`NixOS` `Ubuntu` `KVM/QEMU` `WireGuard` `Tailscale` `Headscale` `Nginx` `Docker`

### Security & Monitoring
`Elasticsearch` `Kibana` `Fluent Bit` `Wazuh` `Suricata IDS/IPS` `Fail2ban` `sops-nix`

### Networking & Access
`WireGuard` `Tailscale` `Zero Trust` `ACL Policy` `NFQUEUE` `iptables` `nftables`

### Protocols & Standards
`TLS/HTTPS` `ACME/Let's Encrypt` `DNS-01` `Cloudflare` `KQL` `SIEM` `HIDS` `FIM`

---

## Secrets & Security Notice

All secrets (API keys, passwords, private keys, IP addresses) are managed via sops-nix and excluded from version control. No credentials, private keys, or infrastructure-identifying information are present in this repository.

---

## Status

| Component | Status |
|---|---|
| ELK Stack | ✅ Production |
| Wazuh HIDS | ✅ Production |
| Suricata IPS — VPS | ✅ Production — NFQUEUE mode, et/open ruleset, 49k+ rules |
| Nginx Hardening | ✅ Production — Rate limiting, hidden file blocks, SSL verification, error silencing (→ 444) |
| Anti-Scan Blackhole | ✅ Production — HTTPS default_server returns 444; all error codes silenced |
| Firewall Hardening | ✅ Production — IKE/IPsec DROP, ICMP echo DROP, ICMP timestamp DROP |
| Fail2ban | ✅ Production — nftables backend, blocktype=drop |
| WireGuard tunnels | ✅ Production |
| Tailscale mesh | ✅ Production |
| Vaultwarden | ✅ Production — daily use |
| sops-nix | ✅ Production |
| Fluent Bit parsers | ✅ Production — Tailscale SSH, fail2ban, Suricata |
| Headscale Whitelist | 📋 Planned — Restricting VPS paths to /register, /machine, etc. |
| Access Logging Audit | ✅ Completed — Verified vhost log consistency |
| Wazuh custom rules | 📋 Planned |
| Zeek — Daily Driver | 📋 Planned |
| OpenSnitch — Daily Driver | 📋 Planned |
| Enable Suricata in IPS mode on all interfaces and machines | 📋 Planned |
| OPNsense VM | 📋 Planned |
| Attack simulation | 📋 Planned |
| VPS NixOS migration | 📋 Planned |
