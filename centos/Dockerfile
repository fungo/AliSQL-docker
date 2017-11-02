FROM centos:7

# add our user and group first to make sure their IDs get assigned consistently, regardless of whatever dependencies get added
RUN groupadd -r mysql && useradd -r -g mysql mysql

# add gosu for easy step-down from root
ENV GOSU_VERSION 1.10
RUN set -x \
        && curl -o /usr/local/bin/gosu -L https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-amd64 \
        && curl -o /usr/local/bin/gosu.asc -L https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-amd64.asc \
        && export GNUPGHOME="$(mktemp -d)" \
        && gpg --keyserver ha.pool.sks-keyservers.net --recv-keys B42F6819007F00F88E364FD4036A9C25BF357DD4 \
        && gpg --batch --verify /usr/local/bin/gosu.asc /usr/local/bin/gosu \
        && rm -r "$GNUPGHOME" /usr/local/bin/gosu.asc \
        && chmod +x /usr/local/bin/gosu \
        && gosu nobody true

RUN mkdir /docker-entrypoint-initdb.d

RUN set -x \
        && mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup \
        && curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo \
        && yum makecache \
        ## FATAL ERROR: please install the following Perl modules before executing /usr/local/mysql/scripts/mysql_install_db:
        ## File::Basename
        ## File::Copy
        ## Sys::Hostname
        ## Data::Dumper
        && yum install -y perl perl-Data-Dumper\
        ## mysqld: error while loading shared libraries: libaio.so.1: cannot open shared object file: No such file or directory
        && yum install -y libaio \
        && yum clean all \
        && mv /etc/yum.repos.d/CentOS-Base.repo.backup /etc/yum.repos.d/CentOS-Base.repo

ENV ALISQL_VERSION 5.6.32-7

RUN set -x \
        && mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup \
        && curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo \
        && yum makecache \
        && curl -o alisql.tar.gz -L https://github.com/alibaba/AliSQL/archive/AliSQL-$ALISQL_VERSION.tar.gz \
        && tar -zxf alisql.tar.gz && cd AliSQL-AliSQL-$ALISQL_VERSION \
        && yum install -y gcc gcc-c++ cmake make bison git ncurses-devel openssl-devel \
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
        && yum autoremove -y gcc gcc-c++ cmake make bison git ncurses-devel openssl-devel \
        && yum clean all \
        && cd ..  && rm -f alisql.tar.gz && rm -rf AliSQL-AliSQL-$ALISQL_VERSION \
        && rm -rf /usr/local/mysql/mysql-test /usr/local/mysql/sql-bench \
        && rm -rf /usr/local/mysql/bin/mysqltest /usr/local/mysql/bin/mysql_client_test \
        && rm -rf /usr/local/mysql/bin/*-debug /usr/local/mysql/bin/*_embedded \
        && find /usr/local/mysql -type f -name "*.a" -delete \
        && { find /usr/local/mysql -type f -executable -exec strip --strip-all '{}' + || true; } \
        && mv /etc/yum.repos.d/CentOS-Base.repo.backup /etc/yum.repos.d/CentOS-Base.repo

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
