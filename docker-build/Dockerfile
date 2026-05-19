# ==============================================================================
# LAZARUS HOMELAB - ENTERPRISE DNS IMAGE
# Core: Pi-hole v6 + Unbound Hyperlocal
# Architecture: Multi-Arch (linux/amd64, linux/arm64)
# ==============================================================================

# 1. BASE IMAGE PIPELINE
# Using 'latest' ensures GitHub Actions always pulls the absolute newest
# official Pi-hole code every time the automated build triggers.
FROM pihole/pihole:latest

# 2. PACKAGE INJECTION
# Install Unbound, required libraries, and the Lazarus administrative 
# toolkit (nano, net-tools, sqlite) that you previously forced via Compose.
RUN apk update && apk upgrade --no-cache && \
    apk add --no-cache \
    unbound \
    unbound-libs \
    wget \
    nano \
    net-tools \
    sqlite

# 3. UNBOUND CORE CONFIGURATION
# Copy the standard hyperlocal routing logic into the image.
COPY unbound-pihole.conf /etc/unbound/unbound.conf.d/pihole.conf
COPY 99-edns.conf /etc/dnsmasq.d/99-edns.conf

# 4. THE LAZARUS TCP FORTIFICATION (v1.7.1)
# Bakes the Unbound TCP aggression defense directly into the image structure,
# permanently solving the Pi-hole v6 FTL connection clash without entrypoint scripts.
RUN printf 'server:\n  incoming-num-tcp: 1024\n  do-tcp: yes\n  tcp-idle-timeout: 30000\n' > /etc/unbound/unbound.conf.d/99-tcp-fix.conf

# 5. THE LAZARUS UI THERMAL PATCH
# Permanently modifies the Pi-hole Lua dashboard files during the build phase.
# This prevents the container from wasting CPU cycles running 'sed' on every reboot.
RUN sed -i 's|<p>Status</p>||g' /var/www/html/admin/scripts/lua/sidebar.lp && \
    sed -i '/id="temperature"/d' /var/www/html/admin/scripts/lua/sidebar.lp && \
    sed -i 's|<div class="pull-left info">|<div class="pull-left info">\n                    <p>Status</p>\n                    <span id="temperature"><i class="fa-solid fa-fw fa-temperature-three-quarters text-green-light"></i>\&nbsp;\&nbsp;Temp: <%= string.format("%.1f°C", tonumber(io.open("/sys/class/thermal/thermal_zone0/temp"):read("*a")) / 1000) %></span><br/>|' /var/www/html/admin/scripts/lua/sidebar.lp

# 6. DNS ROOT HINTS
# Download the latest top-level domain root servers for Unbound.
RUN wget -S https://www.internic.net/domain/named.root -O /etc/unbound/root.hints

# 7. LOGGING & SECURITY PERMISSIONS
# Secure the Unbound log directory with the correct user ownership.
RUN install -o unbound -g unbound -Dm755 -d /var/log/unbound

# 8. LIFECYCLE MANAGEMENT
# Copy and execute the custom startup script that initializes Unbound alongside FTL.
COPY --chmod=0755 unbound-entrypoint.sh /usr/bin/unbound-entrypoint.sh

ENTRYPOINT ["unbound-entrypoint.sh"]
