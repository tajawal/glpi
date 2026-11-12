ARG PHP_VERSION=latest

FROM ghcr.io/glpi-project/glpi-development-env:$PHP_VERSION

USER root

# CVE-2026-119618 (mod_http2 heap corruption / RCE, Apache 2.4.55-2.4.70,
# fixed in 2.4.71): the upstream dev image still ships 2.4.69, so take the
# fixed apache2 from the distro security repo, then fail the build if an
# affected version is still installed. Replacing the binary drops the
# cap_net_bind_service file capability the base image sets so Apache can bind
# port 80 as www-data; setcap restores it.
# HTTP/2 stays off in the dev image (a2enmod http2 / Protocols h2): the guard
# below enforces it. Remove the guard deliberately if HTTP/2 is ever needed.
RUN apt-get update \
  && apt-get install --yes --no-install-recommends --only-upgrade apache2 apache2-bin apache2-data apache2-utils \
  && rm -rf /var/lib/apt/lists/* \
  && setcap cap_net_bind_service=+ep /usr/sbin/apache2 \
  && dpkg --compare-versions "$(dpkg-query -W -f='${Version}' apache2-bin)" ge "2.4.71-0~" \
  || { echo "apache2 upgrade failed or apache2-bin is still < 2.4.71 (CVE-2026-119618)" >&2; exit 1; }

# Keep HTTP/2 off and fail loudly if anything has enabled it. Only enabled
# config is checked, and -R follows the *-enabled symlinks into *-available:
# mods-available/http2.conf ships "Protocols h2 ..." and is inert unless the
# module is enabled.
RUN a2dismod -f http2 >/dev/null 2>&1 || true; \
  if [ -e /etc/apache2/mods-enabled/http2.load ] \
    || apache2ctl -M 2>/dev/null | grep -qi http2_module \
    || grep -RqiE '^\s*Protocols\b.*\bh2c?\b' /etc/apache2/apache2.conf /etc/apache2/ports.conf /etc/apache2/*-enabled/; then \
    echo "HTTP/2 is enabled in Apache; keep it off (CVE-2026-119618)" >&2; exit 1; \
  fi

# Allow extra PHP configuration to be placed in a dedicated folder than can serve
# as a volume
ENV PHP_INI_SCAN_DIR "/usr/local/etc/php/conf.d/:/usr/local/etc/php/custom_conf.d/"
RUN mkdir /usr/local/etc/php/custom_conf.d/

# Workaround to make apache use same UID/GID as host user.
ARG HOST_GROUP_ID=1000
RUN groupmod --gid ${HOST_GROUP_ID} www-data
ARG HOST_USER_ID=1000
RUN usermod --uid ${HOST_USER_ID} www-data

# Allow login as www-data user and allow it to execute sudo
RUN usermod --shell /bin/bash www-data
RUN echo "www-data ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers

# Make www-data user home persistent for cache purpose.
RUN mkdir --parents /home/www-data \
  && chown www-data:www-data /home/www-data \
  && usermod --home /home/www-data www-data
VOLUME /home/www-data

USER www-data
