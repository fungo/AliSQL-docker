FROM debian:jessie

# add our user and group first to make sure their IDs get assigned consistently, regardless of whatever dependencies get added
RUN groupadd -r mysql && useradd -r -g mysql mysql

# add gosu for easy step-down from root
ENV GOSU_VERSION 1.10
RUN set -x \
        && sed -i 's/deb.debian.org/mirrors.aliyun.com/g' /etc/apt/sources.list \
        && apt-get update && apt-get install -y --no-install-recommends ca-certificates wget && rm -rf /var/lib/apt/lists/* \
        && wget -O /usr/local/bin/gosu "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$(dpkg --print-architecture)" \
        && wget -O /usr/local/bin/gosu.asc "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$(dpkg --print-architecture).asc" \
        && export GNUPGHOME="$(mktemp -d)" \
        && gpg --keyserver ha.pool.sks-keyservers.net --recv-keys B42F6819007F00F88E364FD4036A9C25BF357DD4 \
        && gpg --batch --verify /usr/local/bin/gosu.asc /usr/local/bin/gosu \
        && rm -r "$GNUPGHOME" /usr/local/bin/gosu.asc \
        && chmod +x /usr/local/bin/gosu \
        && gosu nobody true \
        && apt-get purge -y --auto-remove ca-certificates wget \
        && sed -i 's/mirrors.aliyun.com/deb.debian.org/g' /etc/apt/sources.list

RUN mkdir /docker-entrypoint-initdb.d

RUN set -x \
        && sed -i 's/deb.debian.org/mirrors.aliyun.com/g' /etc/apt/sources.list \
        && apt-get update \
        ## FATAL ERROR: please install the following Perl modules before executing /usr/local/mysql/scripts/mysql_install_db:
        ## File::Basename
        ## File::Copy
        ## Sys::Hostname
        ## Data::Dumper
        && apt-get install -y perl --no-install-recommends \
        ## mysqld: error while loading shared libraries: libaio.so.1: cannot open shared object file: No such file or directory
        && apt-get install -y libaio1 pwgen \
        && rm -rf /var/lib/apt/lists/* \
        && sed -i 's/mirrors.aliyun.com/deb.debian.org/g' /etc/apt/sources.list

ENV ALISQL_VERSION 5.6.32-6

RUN set -x \
        && sed -i 's/deb.debian.org/mirrors.aliyun.com/g' /etc/apt/sources.list \
        && apt-get update && apt-get install -y ca-certificates wget --no-install-recommends && rm -rf /var/lib/apt/lists/* \
        && wget "https://github.com/alibaba/AliSQL/archive/AliSQL-$ALISQL_VERSION.tar.gz" -O alisql.tar.gz \
        && apt-get purge -y --auto-remove ca-certificates wget \
        && tar -zxf alisql.tar.gz && cd AliSQL-AliSQL-$ALISQL_VERSION \
        && apt-get update && apt-get install -y git gcc g++ cmake bison libncurses5-dev zlib1g-dev libssl-dev && rm -rf /var/lib/apt/lists/* \
        && cmake .                             \
           -DCMAKE_BUILD_TYPE="Release"        \
           -DWITH_EMBEDDED_SERVER=0            \
           -DWITH_EXTRA_CHARSETS=all           \
           -DWITH_MYISAM_STORAGE_ENGINE=1      \
           -DWITH_INNOBASE_STORAGE_ENGINE=1    \
           -DWITH_PARTITION_STORAGE_ENGINE=1   \
           -DWITH_CSV_STORAGE_ENGINE=1         \
           -DWITH_ARCHIVE_STORAGE_ENGINE=1     \
           -DWITH_BLACKHOLE_STORAGE_ENGINE=1   \
           -DWITH_FEDERATED_STORAGE_ENGINE=1   \
           -DWITH_PERFSCHEMA_STORAGE_ENGINE=1  \
           -DWITH_TOKUDB_STORAGE_ENGINE=1      \
        && make -j `cat /proc/cpuinfo | grep processor| wc -l` && make install \
        && apt-get purge -y --auto-remove git gcc g++ cmake bison libncurses5-dev zlib1g-dev libssl-dev \
        && cd ..  && rm -f alisql.tar.gz && rm -rf AliSQL-AliSQL-$ALISQL_VERSION \
        && rm -rf /usr/local/mysql/mysql-test /usr/local/mysql/sql-bench \
        && rm -rf /usr/local/mysql/bin/mysqltest /usr/local/mysql/bin/mysql_client_test \
        && rm -rf /usr/local/mysql/bin/*-debug /usr/local/mysql/bin/*_embedded \
        && find /usr/local/mysql -type f -name "*.a" -delete \
        && apt-get update && apt-get install -y binutils && rm -rf /var/lib/apt/lists/* \
        && { find /usr/local/mysql -type f -executable -exec strip --strip-all '{}' + || true; } \
        && apt-get purge -y --auto-remove binutils \
        && sed -i 's/mirrors.aliyun.com/deb.debian.org/g' /etc/apt/sources.list

ENV PATH $PATH:/usr/local/mysql/bin:/usr/local/mysql/scripts

# make a default configuration
RUN mkdir -p /etc/mysql/conf.d \
        && { \
                echo '[mysqld]'; \
                echo 'skip-host-cache'; \
                echo 'skip-name-resolve'; \
                echo 'datadir = /var/lib/mysql'; \
                echo '!includedir /etc/mysql/conf.d/'; \
        } > /etc/mysql/my.cnf

RUN mkdir -p /var/lib/mysql /var/run/mysqld \
        && chown -R mysql:mysql /var/lib/mysql /var/run/mysqld \
# ensure that /var/run/mysqld (used for socket and lock files) is writable regardless of the UID our mysqld instance ends up having at runtime
        && chmod 777 /var/run/mysqld

VOLUME /var/lib/mysql

COPY docker-entrypoint.sh /usr/local/bin/
RUN ln -s usr/local/bin/docker-entrypoint.sh /entrypoint.sh # backwards compat
ENTRYPOINT ["docker-entrypoint.sh"]

EXPOSE 3306
CMD ["mysqld"]
