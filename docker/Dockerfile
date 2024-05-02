# syntax=docker/dockerfile:1
FROM alpine:3.19 as builder

WORKDIR /workdir

ARG RUST_PYTHON_VERSION="0.3.1"
ARG BUILD_DATE
ARG VCS_REF

ENV CARGO_HOME="/workdir"
ENV CARGO_INSTALL_ROOT="/workdir"
# armv6 and armv7 fix
# See: https://github.com/rust-lang/cargo/issues/6513#issuecomment-1440029221
ENV CARGO_NET_GIT_FETCH_WITH_CLI=true

ADD https://github.com/RustPython/RustPython/archive/refs/tags/$RUST_PYTHON_VERSION.tar.gz /tmp/v$RUST_PYTHON_VERSION.tar.gz

RUN tar --strip-components=1 -C /workdir -xzf /tmp/v$RUST_PYTHON_VERSION.tar.gz

# Remove the git mode that triggers an error on armv6 and armv7
# Ref: https://github.com/rust-lang/cargo/issues/2808 (failed to mmap. Could not write data: Out of memory; class=Os (2))
RUN sed -i 's#{ git = .*#{ path = "/workdir/rp__doc__/" }#' /workdir/derive-impl/Cargo.toml

RUN apk add --no-cache rust cargo musl-dev git

RUN cargo fetch

RUN cargo build --features freeze-stdlib --release --locked --offline --bin rustpython


FROM alpine:3.19

ARG VERSION="1.2.6.7"
ARG DIST_URL="https://github.com/leenooks/phpLDAPadmin/archive/refs/tags/${VERSION}.tar.gz"
ARG BUILD_DATE
ARG VCS_REF

# Remember to also change in horust/services/fpm.toml
ARG PHP_VERSION_MAJOR="8"
ARG PHP_VERSION_MINOR="2"

# Set user and group
ARG user=deploy
ARG group=deploy

ENV CONTAINER_SERVICE_DIR="/container/service"
ENV FIRST_START_DONE="/container/first-start-done"

COPY --from=builder /workdir/target/release/rustpython /usr/local/bin/rustpython

RUN apk add --no-cache --update nginx curl bash musl patch \
    openssl libgcc \
    php${PHP_VERSION_MAJOR}${PHP_VERSION_MINOR}-fpm php${PHP_VERSION_MAJOR}${PHP_VERSION_MINOR} php${PHP_VERSION_MAJOR}${PHP_VERSION_MINOR}-session \
    php${PHP_VERSION_MAJOR}${PHP_VERSION_MINOR}-json php${PHP_VERSION_MAJOR}${PHP_VERSION_MINOR}-curl php${PHP_VERSION_MAJOR}${PHP_VERSION_MINOR}-xml php${PHP_VERSION_MAJOR}${PHP_VERSION_MINOR}-gettext php${PHP_VERSION_MAJOR}${PHP_VERSION_MINOR}-mbstring php${PHP_VERSION_MAJOR}${PHP_VERSION_MINOR}-ldap php${PHP_VERSION_MAJOR}${PHP_VERSION_MINOR}-ctype && \
    apk add --no-cache --repository=https://dl-cdn.alpinelinux.org/alpine/edge/testing horust && \
    horust --help && nginx -v && php-fpm${PHP_VERSION_MAJOR}${PHP_VERSION_MINOR} -v && php -v && \
    mkdir -p /var/log/php${PHP_VERSION_MAJOR}${PHP_VERSION_MINOR} && \
    mkdir /run/phpfpm && \
    ln -sf /dev/stdout /var/log/nginx/access.log && \
    ln -sf /dev/stderr /var/log/nginx/error.log && \
    ln -sf /dev/stderr /var/log/php${PHP_VERSION_MAJOR}${PHP_VERSION_MINOR}/error.log && \
    date -R > /build_date && \
    printf '{"build_date": "%s", "timestamp": %i}' "$(date -R)" "$(date +%s)" > /build_date.json && \
    addgroup -S ${group} && \
    adduser -G ${group} -G nginx -s /bin/sh -h /var/www/phpldapadmin -S ${user} && \
    id deploy && \
    chown -R deploy:deploy /var/log/php${PHP_VERSION_MAJOR}${PHP_VERSION_MINOR} /var/lib/nginx /run/nginx /run/phpfpm && \
    chmod -R 770 /var/log/php${PHP_VERSION_MAJOR}${PHP_VERSION_MINOR} /var/lib/nginx /run/nginx /run/phpfpm && \
    mkdir /container && \
    chown deploy:deploy /container && \
    rustpython --help

WORKDIR /var/www/phpldapadmin

COPY ./Build/docker/sambadc-ui/docker-entrypoint.sh /
COPY ./Build/docker/sambadc-ui/log-helper /usr/local/sbin/
COPY ./Build/docker/sambadc-ui/pythontojson.py /usr/local/sbin/
COPY --chown=deploy:deploy ./Build/docker/sambadc-ui/horust/ /etc/horust/
COPY ./Build/docker/sambadc-ui/nginx-default.conf /etc/nginx/http.d/default.conf
COPY ./Build/docker/sambadc-ui/nginx.conf /etc/nginx/nginx.

COPY ./Build/docker/sambadc-ui/php-fpm-www-pool.conf /etc/php${PHP_VERSION_MAJOR}${PHP_VERSION_MINOR}/php-fpm.d/www.conf
COPY --chown=deploy:deploy ./Build/docker/sambadc-ui/config.php ${CONTAINER_SERVICE_DIR}/phpldapadmin/assets/config/config.php
RUN chown -R deploy:deploy /etc/ssl/certs
# Switch to user
USER deploy:deploy

RUN curl -# -L -o phpldapadmin.tar.gz ${DIST_URL} && \
    tar -xzf phpldapadmin.tar.gz -C /var/www/phpldapadmin --strip-components=1 && \
    rm phpldapadmin.tar.gz && \
    wget https://sources.debian.org/data/main/p/phpldapadmin/1.2.6.7-1/debian/patches/Fix-dynamic-property-PHP-8.2.patch && \
    patch -p 1 -f < ./Fix-dynamic-property-PHP-8.2.patch && \
    printf "RELEASE-${VERSION}" > ./VERSION && \
    rm -vr ./doc/ ./.gitignore ./*.md

COPY <<-EOT /usr/local/share/sbom/phpldapadmin.spdx.json
{
    "spdxVersion": "SPDX-2.3",
    "dataLicense": "CC0-1.0",
    "SPDXID": "SPDXRef-DOCUMENT",
    "name": "docker-phpldapadmin",
    "packages": [
        {
            "name": "phpldapadmin",
            "SPDXID": "SPDXRef-Package-phpldapadmin",
            "versionInfo": "${VERSION}",
            "originator": "Person: Deon George",
            "downloadLocation": "${DIST_URL}",
            "sourceInfo": "dowloaded from GitHub releases",
            "licenseConcluded": "GPL-2.0-or-later",
            "licenseDeclared": "GPL-2.0-or-later",
            "copyrightText": "NOASSERTION",
            "description": "phpLDAPadmin"
        }
    ],
    "externalRefs": [
        {
            "referenceCategory": "SECURITY",
            "referenceLocator": "cpe:2.3:a:phpldapadmin_project:phpldapadmin",
            "referenceType": "cpe23Type"
        },
        {
            "referenceCategory": "SECURITY",
            "referenceLocator": "cpe:/a:phpldapadmin_project:phpldapadmin",
            "referenceType": "cpe22Type"
        },
        {
            "referenceCategory": "PACKAGE_MANAGER",
            "referenceLocator": "pkg:deb/debian/phpldapadmin",
            "referenceType": "purl"
        }
    ]
}
EOT

# Metadata

EXPOSE 80

ENTRYPOINT ["/docker-entrypoint.sh"]
