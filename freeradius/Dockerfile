# syntax=docker/dockerfile:1.6
#
# FreeRADIUS + daloRADIUS Home Assistant add-on
# Base: debian:bookworm-slim (NOT HA base images — they ship s6-overlay
# which we don't want and which historically caused startup issues in
# this setup).
#
# Architecture: Apache2 + mod_php (mpm_prefork). No PHP-FPM, no FastCGI,
# no scoreboard, no flock on /dev/shm. That sidesteps the entire AppArmor
# 'k' permission problem.
#
ARG BUILD_FROM=debian:bookworm-slim
FROM ${BUILD_FROM}

ENV DEBIAN_FRONTEND=noninteractive \
    LANG=C.UTF-8 \
    LC_ALL=C.UTF-8 \
    TZ=UTC \
    APACHE_RUN_USER=www-data \
    APACHE_RUN_GROUP=www-data \
    APACHE_PID_FILE=/var/run/apache2/apache2.pid \
    APACHE_RUN_DIR=/var/run/apache2 \
    APACHE_LOCK_DIR=/var/lock/apache2 \
    APACHE_LOG_DIR=/var/log/apache2

# ---------------------------------------------------------------------------
# System packages
# ---------------------------------------------------------------------------
# - freeradius + freeradius-utils: the RADIUS server itself + radclient/radtest
# - apache2 + libapache2-mod-php: web server with mod_php (no FPM)
# - php-* : only the extensions daloRADIUS actually uses
# - sqlite3: CLI used by run.sh to bootstrap and inspect the DB
# - gettext-base: envsubst (used by run.sh for config templating)
# - tini: proper PID 1 that reaps zombies and forwards signals
# - jq: read /data/options.json
# - curl, ca-certificates: fetch daloRADIUS tarball
# ---------------------------------------------------------------------------
RUN apt-get update \
 && apt-get install -y --no-install-recommends \
      freeradius \
      freeradius-utils \
      freeradius-common \
      freeradius-config \
      sqlite3 \
      apache2 \
      libapache2-mod-php \
      php \
      php-cli \
      php-common \
      php-sqlite3 \
      php-gd \
      php-mbstring \
      php-xml \
      php-curl \
      php-zip \
      php-db \
      php-mail \
      php-mail-mime \
      tini \
      jq \
      gettext-base \
      ca-certificates \
      curl \
      bash \
      tzdata \
 && rm -rf /var/lib/apt/lists/*

# ---------------------------------------------------------------------------
# daloRADIUS
# ---------------------------------------------------------------------------
# Pin to a known tag. 2.2 line supports FreeRADIUS 3 and SQLite.
ARG DALORADIUS_VERSION=2.2.2
RUN set -eux; \
    curl -fsSL "https://github.com/lirantal/daloradius/archive/refs/tags/${DALORADIUS_VERSION}.tar.gz" \
      -o /tmp/daloradius.tgz; \
    mkdir -p /var/www; \
    tar -xzf /tmp/daloradius.tgz -C /tmp; \
    mv "/tmp/daloradius-${DALORADIUS_VERSION}" /var/www/daloradius; \
    rm -f /tmp/daloradius.tgz; \
    chown -R www-data:www-data /var/www/daloradius

# ---------------------------------------------------------------------------
# Rootfs overlay (apache vhost, freeradius SQL module, run.sh)
# ---------------------------------------------------------------------------
COPY rootfs/ /

RUN set -eux; \
    chmod +x /usr/local/bin/run.sh; \
    # Enable Apache modules + site
    a2dissite 000-default >/dev/null; \
    a2ensite daloradius >/dev/null; \
    a2enmod rewrite headers >/dev/null; \
    # Enable FreeRADIUS SQL module (link mods-available/sql -> mods-enabled)
    ln -sf ../mods-available/sql /etc/freeradius/3.0/mods-enabled/sql; \
    # Make sure dirs that Apache/FreeRADIUS write to exist and are owned correctly
    mkdir -p /var/run/apache2 /var/lock/apache2 /var/log/apache2; \
    chown -R www-data:www-data /var/run/apache2 /var/lock/apache2 /var/log/apache2; \
    # FreeRADIUS log to stdout via symlink is handled in run.sh; just ensure dir exists
    mkdir -p /var/log/freeradius; \
    chown -R freerad:freerad /var/log/freeradius /etc/freeradius

EXPOSE 80/tcp 1812/udp 1813/udp

# Healthcheck: Apache serves the daloRADIUS login page
HEALTHCHECK --interval=30s --timeout=5s --start-period=30s --retries=3 \
    CMD curl -fsS http://127.0.0.1/ >/dev/null || exit 1

# tini reaps zombies and forwards SIGTERM to run.sh, which then
# shuts down Apache and FreeRADIUS cleanly.
ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["/usr/local/bin/run.sh"]

# ---------------------------------------------------------------------------
# OCI labels (populated at build time by GitHub Actions)
# ---------------------------------------------------------------------------
ARG BUILD_ARCH=amd64
ARG BUILD_DATE
ARG BUILD_VERSION
ARG BUILD_REF
LABEL \
    io.hass.name="FreeRADIUS + daloRADIUS" \
    io.hass.description="FreeRADIUS 3 with daloRADIUS web UI (SQLite)" \
    io.hass.arch="${BUILD_ARCH}" \
    io.hass.type="addon" \
    io.hass.version="${BUILD_VERSION}" \
    maintainer="qlphon" \
    org.opencontainers.image.title="FreeRADIUS + daloRADIUS" \
    org.opencontainers.image.source="https://github.com/qlphon/ha-addons" \
    org.opencontainers.image.created="${BUILD_DATE}" \
    org.opencontainers.image.revision="${BUILD_REF}" \
    org.opencontainers.image.version="${BUILD_VERSION}"
