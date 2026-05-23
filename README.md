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
```

### 2. The Master Docker Compose File
Save the following configuration as `/opt/docker/pihole/docker-compose.yml`. Ensure you replace placeholders like `<your-username>`, `<Your/Timezone>`, and `example.com` with your deployment details.

```yaml
# ==============================================================================
# PI-HOLE + UNBOUND: HYPERLOCAL RECURSIVE DNS CONFIGURATION
# Verified Deployment: Pi-hole v6 Native + Strict Decentralized Recursion
# Protocol v4.1 Applied: Strict Traefik Routing
# ==============================================================================

services:

  # ============================================================================
  # 1. PI-HOLE + UNBOUND HYPERLOCAL
  # ============================================================================
  pihole: # Internal identifier for the Pi-hole service.
    
    ################################################################
    # Image Configuration                                          #
    ################################################################
    image: ghcr.io/<your-username>/pihole-unbound-hyperlocal:latest 
    
    ################################################################
    # Container Identity                                           #
    ################################################################
    container_name: pihole
    hostname: pihole
    domainname: example.com # Replace with your FQDN suffix
    restart: unless-stopped

    ################################################################
    # Resource Allocation                                          #
    ################################################################
    # Prevents memory exhaustion on massive FTL databases (13M+ rows).
    shm_size: 512mb 

    ################################################################
    # Traefik Configuration (Reverse Proxy)                        #
    ################################################################
    labels: 
      - "traefik.enable=true" 
      - "traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https" 
      - "traefik.http.routers.pihole-http.entrypoints=web" 
      - "traefik.http.routers.pihole-http.middlewares=redirect-to-https" 
      - "traefik.http.routers.pihole-http.rule=Host(`pihole.example.com`)" # Replace domain
      - "traefik.http.routers.pihole.entrypoints=websecure" 
      - "traefik.http.routers.pihole.rule=Host(`pihole.example.com`)" # Replace domain
      - "traefik.http.routers.pihole.tls.certresolver=letsencrypt" 
      - "traefik.http.services.pihole-http.loadbalancer.server.port=80" 

    ################################################################
    # Ports                                                        #
    ################################################################
    ports: 
      - "53:53/tcp" 
      - "53:53/udp" 
      # SECURITY NOTE: Web access is strictly routed via Traefik. Host ports hidden.

    ################################################################
    # Volumes (Persistent Data)                                    #
    ################################################################
    volumes: 
      - "/opt/docker/pihole/etc/dnsmasq.d:/etc/dnsmasq.d" 
      - "/opt/docker/pihole/etc/pihole:/etc/pihole" 

    ################################################################
    # Security Permissions                                         #
    ################################################################
    cap_add: 
      - NET_ADMIN         # Required for network stack management.
      - CAP_SYS_TIME      # Allows container to adjust system time for DNSSEC validation.
      - CAP_SYS_NICE      # Allows Unbound to set process priority for performance.
      - CHOWN             # Allows changing file ownership inside the container.
      - NET_BIND_SERVICE  # Allows binding to privileged ports (<1024).
      - NET_RAW           # Allows use of raw sockets for network probing.
      - SETGID            # Allows setting arbitrary group IDs for security context.
      - SETUID            # Allows setting arbitrary user IDs for security context.

    ################################################################
    # Environment Variables                                        #
    ################################################################
    environment:
      # --- System & Permissions ---
      - PUID=1000 # Map to your standard user account
      - PGID=1000 # Map to your standard user group
      - TZ=<Your/Timezone> # e.g., America/New_York
      - S6_KEEP_ENV=1 # Preserves environment variables through init stages
      - S6_BEHAVIOUR_IF_STAGE2_FAILS=2 
      - S6_CMD_WAIT_FOR_SERVICES_MAXTIME=0 

      # --- Web Interface & API ---
      - FTLCONF_webserver_port=80 
      - FTLCONF_webserver_api_theme=default-dark 
      - TEMPERATUREUNIT=c 
      - FTLCONF_webserver_api_app_sudo=true 

      # --- DNS Core Logic ---
      - FTLCONF_dns_upstreams=127.0.0.1#5335 # Forwards Pi-hole to internal Unbound
      - DNSMASQ_USER=pihole 
      - FTLCONF_dns_dnssec=true 
      - FTLCONF_dns_domainNeeded=true 
      - FTLCONF_dns_bogusPriv=true 
      - FTLCONF_dns_listeningMode=all 

      # --- FTL Database & Cache Optimization ---
      - FTL_CMD=no-daemon 
      - FTLCONF_database_maxDBdays=7 
      - FTLCONF_database_DBinterval=90 
      - FTLCONF_dns_cache_size=10000 

      # --- Unbound Performance Tuning ---
      - UNBOUND_VERBOSITY=3 
      - UNBOUND_NUM_THREADS=4 
      - UNBOUND_SO_RCVBUF=512m 
      - UNBOUND_MSG_CACHE_SIZE=512m 
      - UNBOUND_RRSET_CACHE_SIZE=512m 

      # --- TCP Connection Fortification ---
      - FTLCONF_misc_tcp_timeout=0 

    ################################################################
    # Logging Policies                                             #
    ################################################################
    logging: 
      driver: json-file
      options:
        max-size: 50m 
        max-file: 3   

    ################################################################
    # Networking                                                   #
    ################################################################
    networks:
      pihole_default:
        ipv4_address: 172.20.0.2 # Example static IP
    
    dns: 
      - 172.20.0.2   
      - 1.1.1.1 # Fallback local or public DNS for container self-resolution

# ==============================================================================
# EXTERNAL NETWORKS
# ==============================================================================
networks:
  pihole_default:
    external: true 
```

### 3. Execution
Start the container and monitor the logs to ensure successful DNSSEC initialization.
```bash
cd /opt/docker/pihole
docker compose up -d
docker compose logs -f
```

---

### 🧪 Testing the DNSSEC Chain
Ensure Unbound is properly resolving and cryptographically validating domains. Execute these commands inside the container:

```bash
docker compose exec -it pihole sh

# 1. Standard Resolution Test (Should return IPs)
dig github.com @127.0.0.1 -p 5335 +short

# 2. DNSSEC Failure Test (MUST return 'status: SERVFAIL')
dig sigfail.verteiltesysteme.net @127.0.0.1 -p 5335 | grep status 

# 3. DNSSEC Success Test (MUST return 'status: NOERROR')
dig sigok.verteiltesysteme.net @127.0.0.1 -p 5335 | grep status
```

---

## 🔒 The Privacy Architecture (Unbound)

This fork natively bakes **Layer 2 Protocol Privacy** directly into the image via `unbound-pihole.conf`. 

Because pure DNS recursion requires contacting global root servers over plain text UDP (Port 53), mathematical encryption is impossible without using a centralized upstream provider. To maintain decentralization *and* privacy, this image uses:

1.  **QNAME Minimisation (RFC 7816):** Unbound strips the subdomains from outgoing queries. The global `.com` server never learns you are looking for `app.example.com`; it only learns you are looking for the `example` nameservers.
2.  **EDNS Client Subnet Stripping:** Drops any ECS metadata, preventing authoritative servers from fingerprinting your local network subnet.
3.  **NXDOMAIN Hardening (RFC 8020):** Halts aggressive subdomain probing if the parent domain is cryptographically proven not to exist.

---

## ⚠️ DNS Problems (Host Resolution)
If you are running other docker containers on the same host and cannot use name resolution within these containers, you have to modify `/etc/resolvconf.conf` on your host system:

```text
# If you run a local name server, you should uncomment the below line and
# configure your subscribers configuration files below.
name_servers=127.0.0.1
```
Write the changes to your `resolv.conf`:
```bash
sudo resolvconf -u
```
*See also [StackExchange](https://unix.stackexchange.com/questions/647996/docker-container-dns-not-working-with-pihole)*

---

## 🛑 Recommended Blocklists
Add these to your Pi-hole Adlists via the Web UI:
- [Firebog Non-crossed lists](https://v.firebog.net/hosts/lists.php?type=nocross)
- [Perflyst SmartTV](https://raw.githubusercontent.com/Perflyst/PiHoleBlocklist/master/SmartTV.txt)
- [mmotti Pi-hole RegEx](https://raw.githubusercontent.com/mmotti/pihole-regex/master/regex.list)
- [Hagezi Multi PRO](https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/pro.txt)
