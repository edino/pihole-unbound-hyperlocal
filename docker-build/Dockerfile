# ===============================================================================
# LAZARUS HOMELAB - ENTERPRISE DNS IMAGE
# Core: Pi-hole v6 + Unbound Hyperlocal
# Architecture: Multi-Arch (linux/amd64, linux/arm64)
# ===============================================================================

FROM pihole/pihole:latest

# 1. OCI IMAGE METADATA (ITIL Asset Management)
LABEL org.opencontainers.image.source="https://github.com/edino/pihole-unbound-hyperlocal"
LABEL org.opencontainers.image.description="Lazarus Homelab Enterprise DNS - Pi-hole v6 + Unbound"
LABEL maintainer="Edino De Souza Carvalho"

# 2. PACKAGE INJECTION
RUN apk update && apk upgrade --no-cache && \
    apk add --no-cache \
    unbound \
    unbound-libs \
    wget \
    nano \
    net-tools \
    sqlite \
    bind-tools

# 3. UNBOUND CORE CONFIGURATION
COPY unbound-pihole.conf /etc/unbound/unbound.conf.d/pihole.conf
COPY 99-edns.conf /etc/dnsmasq.d/99-edns.conf

# 4. LAZARUS TCP FORTIFICATION
RUN printf 'server:\n  incoming-num-tcp: 1024\n  do-tcp: yes\n  tcp-idle-timeout: 30000\n' > /etc/unbound/unbound.conf.d/99-tcp-fix.conf

# 5. UI THERMAL PATCH
RUN sed -i 's|<p>Status</p>||g' /var/www/html/admin/scripts/lua/sidebar.lp && \
    sed -i '/id="temperature"/d' /var/www/html/admin/scripts/lua/sidebar.lp && \
    sed -i 's|<div class="pull-left info">|<div class="pull-left info">\n                    <p>Status</p>\n                    <span id="temperature"><i class="fa-solid fa-fw fa-temperature-three-quarters text-green-light"></i>\&nbsp;\&nbsp;Temp: <%= string.format("%.1f°C", tonumber(io.open("/sys/class/thermal/thermal_zone0/temp"):read("*a")) / 1000) %></span><br/>|' /var/www/html/admin/scripts/lua/sidebar.lp

# 6. DNS ROOT HINTS
RUN wget -S https://www.internic.net/domain/named.root -O /etc/unbound/root.hints

# 7. LOGGING & PERMISSIONS
RUN install -o unbound -g unbound -Dm755 -d /var/log/unbound

# 8. LIFECYCLE MANAGEMENT
COPY --chmod=0755 unbound-entrypoint.sh /usr/bin/unbound-entrypoint.sh

# 9. SELF-HEALING HEALTHCHECK (ITIL Service Availability)
# This checks if DNS is responding every 60 seconds.
HEALTHCHECK --interval=60s --timeout=10s --start-period=30s \
    CMD dig +short +norecurse @127.0.0.1 pi.hole || exit 1

ENTRYPOINT ["unbound-entrypoint.sh"]
