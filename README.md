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
| VPS | Ubuntu | Nginx reverse proxy, Headscale, Suricata IPS, WireGuard hub, Fail2ban |
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

Two distinct access paths, both HTTPS end-to-end, Vaultwarden VM never directly internet-exposed:

```
# Tailnet members
Device → Tailscale → Vaultwarden VM Nginx → Vaultwarden

# External users (friends/family)
Device → HTTPS → VPS Nginx → WireGuard (wg1) → Vaultwarden VM Nginx → Vaultwarden
```

### TLS Certificate Management

Each service VM provisions its own wildcard TLS certificate via the NixOS `security.acme` module using Cloudflare DNS-01 challenge validation. Covers `*.mesh.mydomain.com`. Fully automated renewal — no manual certificate management.

---

## Security Stack

| Layer | Tool | Mode |
|---|---|---|
| Network IPS | Suricata 7.0.3 | NFQUEUE active blocking on VPS public interface, et/open ruleset |
| Host IDS | Wazuh 4.14.3 | Agents on all hosts, FIM, rootkit detection, SCA |
| SIEM | Elasticsearch + Kibana 8.13.0 | Centralized log aggregation, KQL queries, custom dashboards |
| Log shipping | Fluent Bit | Structured shipping with custom Lua parsers, WireGuard transport, per-machine scoped credentials |
| Auth blocking | Fail2ban | Reactive IP blocking on VPS, nginx-bad-request jail |
| Private mesh | Tailscale + Headscale | Zero trust, ACL-enforced, SSH via tailnet identity |
| Encrypted tunnels | WireGuard | Channel-separated by function |
| Credential management | Vaultwarden | Self-hosted, HTTPS-only |
| Secrets management | sops-nix | Encrypted secrets in version-controlled NixOS config |
| Reverse proxy | Nginx | Per-service TLS termination, public routing via VPS, exploit probe blocking |

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
| Fluent Bit Log Ingestion Health | 🔄 In progress |
| Nginx Attack Traffic | 📋 Planned |
| Tailscale Activity | 📋 Planned |
| Vaultwarden Access | 📋 Planned |

Dashboards are ongoing — new panels and data sources are added as the stack evolves.

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
`WireGuard` `Tailscale` `Zero Trust` `ACL Policy` `NFQUEUE` `iptables`

### Protocols & Standards
`TLS/HTTPS` `ACME/Let's Encrypt` `DNS-01` `Cloudflare` `KQL` `SIEM` `HIDS` `FIM`

### Concepts Demonstrated
`Defense in depth` `Channel separation` `Infrastructure as code` `Declarative configuration` `Zero trust networking` `Log aggregation` `Intrusion detection` `Active threat blocking` `Encrypted private networking` `Secrets management` `Custom log parsing` `SIEM engineering`

---

## Secrets & Security Notice

All secrets (API keys, passwords, private keys, IP addresses) are managed via sops-nix and excluded from version control. No credentials, private keys, or infrastructure-identifying information are present in this repository.

---

## Status

| Component | Status |
|---|---|
| ELK Stack | ✅ Production |
| Wazuh HIDS | ✅ Production |
| Suricata IPS | ✅ Production — NFQUEUE mode, et/open ruleset, 49k+ rules |
| Fail2ban | ✅ Production — VPS, rollout to remaining hosts planned |
| WireGuard tunnels | ✅ Production |
| Tailscale mesh | ✅ Production |
| Vaultwarden | ✅ Production — daily use |
| sops-nix | ✅ Production |
| Fluent Bit parsers | ✅ Production — Tailscale SSH, fail2ban, Suricata |
| Kibana dashboards | 🔄 In progress — ongoing |
| Nginx exploit blocking | ✅ Production — deny rules on VPS and Vaultwarden VM |
| Per-machine Fluent Bit credentials | 🔄 In progress — VPS complete, remaining hosts pending |
| Wazuh custom rules | 📋 Planned |
| Fail2ban — remaining VMs | 📋 Planned |
| Suricata IPS — Vaultwarden VM | 📋 Planned |
| Suricata IPS — Wazuh VM | 📋 Planned |
| Suricata IPS — ELK VM | 📋 Planned |
| Zeek — Daily Driver | 📋 Planned |
| OpenSnitch — Daily Driver | 📋 Planned |
| OPNsense VM | 📋 Planned |
| Attack simulation | 📋 Planned |
| VPS NixOS migration | 📋 Planned |
