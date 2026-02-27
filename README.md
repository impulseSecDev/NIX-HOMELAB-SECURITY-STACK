# homelab-security-stack

**Self-hosted security monitoring infrastructure built on NixOS, ELK, and Wazuh.**

A fully declarative, reproducible security stack built for learning and portfolio purposes. Covers centralized log aggregation, host-based intrusion detection, network traffic analysis, zero trust access, and encrypted private networking — without relying on any third party for data visibility.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        DAILY DRIVER (NixOS)                     │
│   Fluent Bit ──────────────────────────────────────────────┐    │
│   Wazuh Agent                                              │    │
│                                                            │    │
│   ┌─────────────┐  ┌─────────────┐  ┌──────────────────┐   │    │
│   │  Parrot OS  │  │   ELK VM    │  │    Wazuh VM      │   │    │
│   │     VM      │  │  (NixOS)    │  │    (NixOS)       │   │    │
│   │             │  │             │  │                  │   │    │
│   │  Pen Test   │  │Elasticsearch│  │ Wazuh Manager    │   │    │
│   │  WireGuard  │  │   Kibana    │◄─┤ Wazuh Agent      │   │    │
│   │  Isolated   │  │ Fluent Bit  │  │ Fluent Bit       │   │    │
│   └─────────────┘  └──────▲──────┘  └────────▲─────────┘   │    │
│                            │                  │            │    │
└────────────────────────────┼──────────────────┼────────────┼────┘
                             │   TAILSCALE MESH │            │
                    ─────────┴──────────────────┴────────────┘
                             │
                    ┌────────┴────────┐
                    │   VPS (Ubuntu)  │
                    │                 │
                    │ Nginx           │
                    │ Headscale       │
                    │ Cloudflare      │
                    │   Tunnel        │
                    │ Fluent Bit      │
                    │ Wazuh Agent     │
                    │ Suricata IDS    │
                    │ Fail2ban        │
                    └────────┬────────┘
                             │
                    WireGuard Tunnel
                    10.10.10.0/24
                    VPS       → 10.10.10.1
                    ELK VM    → 10.10.10.2
                    Wazuh VM  → 10.10.10.3
```

---

## Log Flow

```
VPS (Nginx, auth, fail2ban, Suricata, Docker)
  └── Fluent Bit → WireGuard (10.10.10.2:9200) → Elasticsearch
      index: vps-YYYY.MM.DD

Daily Driver (systemd journal)
  └── Fluent Bit → Tailscale → Elasticsearch
      index: dailydriver-YYYY.MM.DD

ELK VM (systemd journal)
  └── Fluent Bit → localhost → Elasticsearch
      index: elkbox-YYYY.MM.DD

Wazuh VM (systemd journal)
  └── Fluent Bit → Tailscale → Elasticsearch
      index: wazuhbox-YYYY.MM.DD

Wazuh Agents (all hosts)
  └── Wazuh Manager → internal Filebeat → Elasticsearch
      index: wazuh-alerts-YYYY.MM.DD
```

---

## Hosts

| Host | OS | Role | Repo |
|---|---|---|---|
| Daily Driver | NixOS | Primary workstation, VM host | Private |
| VPS | Ubuntu | Nginx, Headscale, Cloudflare Tunnel, Suricata IDS | [VPS-CONFIG](https://github.com/impulseSecDev/VPS-CONFIG) |
| ELK VM | NixOS | Elasticsearch, Kibana, Fluent Bit | [ELK-NIXVM](https://github.com/impulseSecDev/ELK-NIXVM) |
| Wazuh VM | NixOS | Wazuh Manager, Fluent Bit | [WAZUH-NIXVM](https://github.com/impulseSecDev/WAZUH-NIXVM) |
| Parrot OS VM | Parrot OS | Penetration testing, isolated network | - |

---

## Full Stack

| Component | Version | Host | Purpose |
|---|---|---|---|
| Elasticsearch | 8.13.0 | ELK VM | Log and alert storage |
| Kibana | 8.13.0 | ELK VM | Dashboards, KQL queries, alerts |
| Fluent Bit | 4.x / 3.x | All hosts | Log shipping |
| Wazuh Manager | 4.14.3 | Wazuh VM | HIDS, FIM, alert correlation |
| Wazuh Agents | 4.14.3 | All hosts | Host monitoring and event shipping |
| Suricata | 7.0.3 | VPS | Network traffic inspection, IDS mode |
| Fail2ban | - | VPS | Reactive IP blocking |
| WireGuard | - | VPS, ELK VM, Wazuh VM | Encrypted private tunnel |
| Tailscale | - | All NixOS hosts | Private mesh network |
| Headscale | - | VPS | Self-hosted Tailscale coordination |
| Cloudflare Tunnel | - | VPS | Zero trust SSH access |
| Nginx | - | VPS | Reverse proxy, TLS termination |

---

## Network Design

### Private mesh — Tailscale with self-hosted Headscale

All NixOS hosts join a private Tailscale mesh coordinated by a self-hosted Headscale server on the VPS. No services are exposed publicly — everything is accessible only over Tailscale.

### VPS connectivity — WireGuard

The VPS runs Headscale and therefore cannot join the Tailscale mesh as a client. A dedicated WireGuard tunnel connects the VPS to both the ELK VM and Wazuh VM on the `10.10.10.0/24` subnet. Both VMs initiate outbound connections to the VPS — no port forwarding is needed on the home router.

### Zero trust SSH — Cloudflare Tunnel

Port 22 is firewalled on the VPS. SSH access is only possible via a Cloudflare Tunnel running as a Docker Compose service. If this container goes down, SSH access is lost — it is monitored in Kibana with a dedicated alert.

### Defense in depth

| Layer | Tool | Type |
|---|---|---|
| Network perimeter | Suricata IDS | Passive detection on public interface |
| Host access control | Fail2ban | Reactive blocking on auth failures |
| Host monitoring | Wazuh agents | HIDS, FIM, rootkit detection |
| Log visibility | ELK + Fluent Bit | Centralized aggregation and analysis |
| Network access | Tailscale + WireGuard | Zero trust private networking |
| Remote access | Cloudflare Tunnel | No exposed SSH port |

---

## Secrets Management

All secrets stored in `/etc/secrets/` on each host — outside version-controlled directories to prevent accidental commits. Permissions are `700` on the directory and `600` on each file. Credentials are injected into Docker containers and systemd services via `EnvironmentFile` — never hardcoded in NixOS configuration files or Docker Compose files.

Public WireGuard keys are safe to commit. Private keys, passwords, IPs, and API tokens are never committed.

---

## Why NixOS

All VMs run NixOS for reproducibility. The entire system state is declared in version-controlled configuration files. Rebuilding a VM from scratch produces an identical result. Docker containers for software not in nixpkgs (Elasticsearch, Kibana, Wazuh) are declared via `virtualisation.oci-containers` — managed by systemd and fully reproducible despite running in containers.

---

## Resume Keywords

Elasticsearch · Kibana · Fluent Bit · Wazuh · Suricata IDS · Fail2ban · WireGuard · Tailscale · Headscale · NixOS · Docker · KQL · SIEM · Log aggregation · Host-based intrusion detection · File integrity monitoring · Defense in depth · Zero trust · Cloudflare Tunnel · Declarative infrastructure · Reproducible systems

