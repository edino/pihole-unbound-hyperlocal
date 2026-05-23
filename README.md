# 🛡️ Pi-hole + Unbound + Hyperlocal (Hardened Homelab Edition)

> [!NOTE]  
> **Protocol v4.1 Active:** This is a customized fork of the excellent [sujiba/pihole-unbound-hyperlocal](https://github.com/sujiba/pihole-unbound-hyperlocal) repository, specifically hardened for AMD64/x86_64 environments. It implements strict Layer 2 Privacy (QNAME Minimisation), TCP connection fortification, and native Traefik v3 reverse proxy integration.

## 🚀 Upgrade Notes & Architecture

> [!CAUTION]  
> ## 🚨 V6 ARCHITECTURE & BREAKING CHANGES
> **Pi-hole v6 has been entirely redesigned from the ground up.** It replaces lighttpd with a native webserver and introduces a REST API. This repository preserves the v6 transition while adding severe privacy constraints to the Unbound recursive resolver.
> 
> *Read upstream docs: [Docker Pi-hole v6](https://github.com/pi-hole/docker-pi-hole)*

### The Hardened Upgrades
* **Layer 2 Privacy:** Native integration of RFC 7816 (QNAME Minimisation) and RFC 8020 (NXDOMAIN Hardening).
* **TCP Fortification:** Injected `incoming-num-tcp: 1024` and `tcp-idle-timeout: 30000` to prevent Pi-hole/FTL and Unbound from clashing over dropped connections.
* **Decentralized Recursion:** Fully independent DNS resolution. We do not use Cloudflare or Google. We query the Global Root Servers directly, but obfuscate the payload to prevent tracking.

---

## 📚 Navigation

- [Acknowledgements](#acknowledgements)
- [Introduction](#introduction)
- [Prerequisites](#prerequisites)
- [First Startup (Deployment)](#first-startup-deployment)
  - [Testing the DNSSEC Chain](#testing-the-dnssec-chain)
- [The Privacy Architecture (Unbound)](#the-privacy-architecture-unbound)
- [DNS Problems (Host Resolution)](#dns-problems-host-resolution)
- [Recommended Blocklists](#recommended-blocklists)

---

## 🤝 Acknowledgements
This architecture is built upon the incredible work of the open-source community:
- [sujiba/pihole-unbound-hyperlocal](https://github.com/sujiba/pihole-unbound-hyperlocal) (The primary upstream foundation)
- [Docker Pi-hole](https://github.com/pi-hole/docker-pi-hole)
- [Unbound DNS](https://nlnetlabs.nl/projects/unbound/about/)
- [Pi-hole Unbound Guide](https://docs.pi-hole.net/guides/dns/unbound/)
- [mpgirro/docker-pihole-unbound](https://github.com/mpgirro/docker-pihole-unbound)

---

## 📖 Introduction

**Pi-hole:**
Acts as the network-wide DNS sinkhole, protecting devices from unwanted content, telemetry, and tracking without requiring client-side software.

**Unbound:**
A validating, recursive, and caching DNS resolver. Instead of handing your entire browsing history to an upstream provider (like Google 8.8.8.8), Unbound mathematically traverses the global internet hierarchy to find IP addresses independently.

**Hyperlocal (Root Hints):**
To accelerate resolution, the container is pre-packaged with the DNS Root Zone (`root.hints`). Unbound uses this local cache to contact the global root servers directly without needing a third party to locate them.

---

## 🛠️ Prerequisites
- An AMD64/x86_64 host node.
- [Docker Engine](https://docs.docker.com/get-docker/) & Docker Compose.
- An existing Docker external network (e.g., `pihole_default`) managed by a Reverse Proxy (e.g., Traefik).

---

## 💻 First Startup (Deployment)

To deploy the hardened image, execute the following steps on your host machine.

### 1. Directory Preparation
Create the persistent volume paths on the host to ensure Pi-hole retains its database across container reboots.
```bash
mkdir -p /opt/docker/pihole/etc/{dnsmasq.d,pihole}
sudo chown -R 1000:1000 /opt/docker/pihole
