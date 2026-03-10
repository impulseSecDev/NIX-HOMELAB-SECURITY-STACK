# Homelab Security Stack

> Production-grade, fully declarative security monitoring infrastructure built on NixOS, ELK, Wazuh, and WireGuard. All services run as real daily-use systems — not sandboxed demos.

---

## Overview

A self-hosted security stack designed around defense-in-depth, zero trust networking, and complete infrastructure-as-code. Every host is declared in version-controlled NixOS configuration. No configuration drift, no undocumented changes — the repository is the system.

Built for hands-on learning and portfolio demonstration across network security, SIEM engineering, host-based intrusion detection, encrypted private networking, and secrets management.

---

## Infrastructure

| Host | OS | Role |
|---|---|---|
| Daily Driver | NixOS | Primary workstation, KVM/QEMU VM host, OpenWebUI (local LLM) |
| VPS | Ubuntu | Nginx reverse proxy, Headscale, Suricata IPS, WireGuard hub |
| ELK VM | NixOS | Elasticsearch, Kibana, Fluent Bit — SIEM core |
| Wazuh VM | NixOS | Wazuh Manager, Fluent Bit — HIDS core |
| Vaultwarden VM | NixOS | Self-hosted password manager, family/friends access |
| Laptop | NixOS | Mobile workstation, log shipping over WireGuard |

---

## Network Architecture

### Zero Trust Mesh — Tailscale + Headscale

All NixOS hosts connect to a private Tailscale mesh coordinated by a self-hosted Headscale server on the VPS. All inter-host admin traffic, SSH access, and internal service communication routes exclusively over the encrypted mesh. ACL policy enforces access control — only authenticated tailnet members can reach services or initiate SSH sessions.

```
Daily Driver (100.64.0.1) ─┐
ELK VM       (100.64.0.3) ─┤
Wazuh VM     (100.64.0.4) ─┼── Headscale (tails.loranjennings.com)
Vaultwarden  (100.64.0.5) ─┤
Laptop                    ─┘
```

### Log Shipping — WireGuard wg0 (10.10.10.0/24)

A dedicated WireGuard interface handles all log shipping traffic, deliberately separated from admin channels. All VMs and the laptop maintain encrypted tunnels to the VPS hub. The VPS forwards traffic between peers using the WireGuard routing table — no traffic is routable outside the `10.10.10.0/24` subnet.

```
Daily Driver (10.10.10.4) ─┐
ELK VM       (10.10.10.2) ─┤
Wazuh VM     (10.10.10.3) ─┼── VPS hub (10.10.10.1)
Vaultwarden  (10.10.10.5) ─┤
Laptop       (10.10.10.6) ─┘
```

### SSH Access — WireGuard wg2

A dedicated WireGuard interface exclusively for SSH access to the VPS, isolated from log shipping and service traffic. No port 22 exposed publicly.

### Vaultwarden Access Paths

Two distinct access paths, both HTTPS end-to-end, Vaultwarden VM never directly internet-exposed:

```
# Tailnet members
Device → Tailscale → Vaultwarden VM Nginx → Vaultwarden

# External users (friends/family)
Device → HTTPS → VPS Nginx → WireGuard (wg1) → Vaultwarden VM Nginx → Vaultwarden
```

### TLS Certificate Management

Each service VM provisions its own wildcard TLS certificate via the NixOS `security.acme` module using Cloudflare DNS-01 challenge validation. Covers `*.mesh.loranjennings.com`. Fully automated renewal — no manual certificate management.

---

## Security Stack

| Layer | Tool | Mode |
|---|---|---|
| Network IPS | Suricata 7.0.3 | NFQUEUE active blocking on VPS public interface |
| Host IDS | Wazuh 4.14.3 | Agents on all hosts, FIM, rootkit detection, SCA |
| SIEM | Elasticsearch + Kibana 8.13.0 | Centralized log aggregation, KQL queries, alerting |
| Log shipping | Fluent Bit | Structured shipping with custom parsers, WireGuard transport |
| Auth blocking | Fail2ban | Reactive IP blocking on VPS |
| Private mesh | Tailscale + Headscale | Zero trust, ACL-enforced, SSH via tailnet identity |
| Encrypted tunnels | WireGuard | Channel-separated by function |
| Credential management | Vaultwarden | Self-hosted, HTTPS-only |
| Secrets management | sops-nix | Encrypted secrets in version-controlled NixOS config (rollout in progress) |
| Reverse proxy | Nginx | Per-service TLS termination, public routing via VPS |

---

## Channel Separation

A deliberate design decision to separate network traffic by function rather than routing everything over a single interface:

| Channel | Transport | Traffic |
|---|---|---|
| Admin / SSH | Tailscale mesh | SSH, file transfers, service access |
| Log shipping | WireGuard wg0 | Fluent Bit, Wazuh agent comms |
| Vaultwarden routing | WireGuard wg1 | Public → VPS → Vaultwarden VM |
| VPS SSH | WireGuard wg2 | SSH access to VPS only |

This ensures a compromise of one channel cannot affect others. WireGuard's ChaCha20-Poly1305 authenticated encryption means the VPS hub cannot read or manipulate log traffic in transit even though it forwards packets between peers.

---

## Why NixOS

All VMs run NixOS for reproducibility and auditability. The entire system state is declared in version-controlled configuration files. Rebuilding any VM from scratch produces an identical result — no configuration drift, no undocumented changes, no snowflake servers.

Docker containers for software not in nixpkgs (Elasticsearch, Kibana, Wazuh) are declared via `virtualisation.oci-containers` — managed by systemd, fully reproducible despite running in containers.

The VPS runs Ubuntu and is a candidate for future NixOS + impermanence migration.

In the end, the infrastructure is the documentation.

---

## Tech Stack

### Core Infrastructure
`NixOS` `Ubuntu` `KVM/QEMU` `WireGuard` `Tailscale` `Headscale` `Nginx` `Docker`

### Security & Monitoring
`Elasticsearch` `Kibana` `Fluent Bit` `Wazuh` `Suricata IDS/IPS` `Fail2ban` `sops-nix`

### Networking & Access
`WireGuard` `Tailscale` `Zero Trust` `ACL Policy` `NFQUEUE` `iptables` `UFW`

### Protocols & Standards
`TLS/HTTPS` `ACME/Let's Encrypt` `DNS-01` `Cloudflare` `KQL` `SIEM` `HIDS` `FIM`

### Concepts Demonstrated
`Defense in depth` `Channel separation` `Infrastructure as code` `Declarative configuration` `Zero trust networking` `Log aggregation` `Intrusion detection` `Active threat blocking` `Encrypted private networking` `Secrets management`

---

## Secrets & Security Notice

All secrets (API keys, passwords, private keys, IP addresses) are managed via sops-nix and excluded from version control. Public WireGuard keys are safe to commit and appear in configs as-is. No credentials, private keys, or infrastructure-identifying information are present in this repository.

---

## Status

| Component | Status |
|---|---|
| ELK Stack | ✅ Production |
| Wazuh HIDS | ✅ Production |
| Suricata IPS | ✅ Production — NFQUEUE mode |
| WireGuard tunnels | ✅ Production |
| Tailscale mesh | ✅ Production |
| Vaultwarden | ✅ Production — daily use |
| sops-nix | 🔄 Rollout in progress |
| Kibana dashboards | 🔄 In progress |
| Wazuh custom rules | 📋 Planned |
| OPNsense VM | 📋 Planned |
| Attack simulation | 📋 Planned |
