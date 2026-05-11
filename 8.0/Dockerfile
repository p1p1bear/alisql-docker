FROM oraclelinux:8-slim

RUN groupadd --system --gid 999 mysql \
 && useradd --system --uid 999 --gid 999 --home-dir /var/lib/mysql --no-create-home mysql

ENV GOSU_VERSION=1.17
RUN set -eux; \
    arch="$(uname -m)"; \
    case "$arch" in \
      aarch64) gosuArch=arm64 ;; \
      x86_64)  gosuArch=amd64 ;; \
      *) echo "unsupported $arch"; exit 1 ;; \
    esac; \
    curl -fL -o /usr/local/bin/gosu \
      "https://github.com/tianon/gosu/releases/download/${GOSU_VERSION}/gosu-${gosuArch}"; \
    chmod +x /usr/local/bin/gosu; gosu nobody true

RUN microdnf install -y bzip2 gzip openssl xz zstd findutils \
      libaio numactl-libs ncurses-compat-libs \
 && microdnf clean all

ENV ALISQL_VERSION=8.0.44-1
ENV MYSQL_MAJOR=8.0

ARG TARGETARCH
COPY alisql-${ALISQL_VERSION}-${TARGETARCH}.tar.xz /tmp/alisql.tar.xz

RUN set -eux; \
    tar -xJf /tmp/alisql.tar.xz -C /opt; \
    if [ ! -d /opt/alisql ]; then \
      mv /opt/alisql-* /opt/alisql; \
    fi; \
    test -x /opt/alisql/bin/mysqld; \
    rm /tmp/alisql.tar.xz

ENV PATH=/opt/alisql/bin:$PATH
RUN mysqld --version

RUN mkdir -p /etc/mysql/conf.d /var/lib/mysql /var/run/mysqld /docker-entrypoint-initdb.d \
 && chown -R mysql:mysql /var/lib/mysql /var/run/mysqld \
 && chmod 1777 /var/lib/mysql /var/run/mysqld \
 && printf '%s\n' \
      '[mysqld]' \
      'basedir=/opt/alisql' \
      'datadir=/var/lib/mysql' \
      'socket=/var/run/mysqld/mysqld.sock' \
      'lc-messages-dir=/opt/alisql/share' \
      '' \
      '[client]' \
      'socket=/var/run/mysqld/mysqld.sock' \
      '' \
      '!includedir /etc/mysql/conf.d/' \
      > /etc/my.cnf

VOLUME /var/lib/mysql
COPY docker-entrypoint.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/docker-entrypoint.sh

EXPOSE 3306 33060
ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["mysqld"]
