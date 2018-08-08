FROM debian:stretch
#FROM ppc64le/debian:stretch
ARG DEBRELEASE
ENV DEBRELEASE=${DEBRELEASE}
ARG PKGCACHE
ENV PKGCACHE=${PKGCACHE}
ARG IP
ENV IP=${IP}

RUN bash -c 'echo "Acquire::http::proxy \"http://${PKGCACHE}/\";" > /etc/apt/apt.conf.d/00Proxy '
COPY apt.sources.list /etc/apt/sources.list
RUN bash -c 'sed "s#\${DEBRELEASE}#${DEBRELEASE}#g" -i /etc/apt/sources.list'
RUN apt-get -y update && apt-get dist-upgrade -y
RUN echo "America/Denver" > /etc/timezone
RUN cat /usr/share/zoneinfo/America/Denver > /etc/localtime

RUN /usr/sbin/groupadd -r lsfadmin -g 101 && \
    /usr/sbin/useradd -c "schduler daemon" -d "/etc/slurm" -g lsfadmin -s /bin/bash -u 210 -r lsfadmin 

RUN /usr/sbin/groupadd -r munge && \
    /usr/sbin/useradd -c "MUNGE authentication service" -d "/usr/local/etc/munge" -g munge -s /bin/false -r munge && \
    mkdir -m 0700 -p /etc/munge/


RUN apt-get -y install supervisor build-essential autoconf automake git cmake git libgcrypt20-dev  \
    libtool libpam0g-dev libhdf5-dev man2html libnuma-dev libglib2.0-dev libreadline-dev \
    libcurl4-openssl-dev libssh2-1-dev librrd-dev rrdtool libfreeipmi-dev libjson-c-dev  \
    freeipmi wget libgtk2.0-dev libncurses5-dev libmariadbclient-dev-compat liblua5.3-dev \
    libjansson-dev libevent-dev libgettextpo-dev liblz4-dev gettext flex librrd8 libipmimonitoring-dev \
    libibumad-dev libibmad-dev texi2html texinfo help2man libaio-dev bison libxml2-dev \
    libzmq3-dev libboost-graph-dev libmsgpack-dev libjudy-dev bzip2 libssl-dev libjemalloc-dev \
    mailutils msmtp libdbi-perl libterm-readkey-perl libdbd-mysql-perl libapr1-dev  libconfuse-dev \
    libexpat1-dev libpcre3-dev libz-dev libmemcached-dev librrd-dev rrdtool rrdcached python-pyldap

ENV PKGCONFIG=/usr/local/mysql/share/:/usr/local/lib/pkgconfig/:/usr/lib/powerpc64le-linux-gnu/pkgconfig
ENV PATH=/usr/local/mysql/sbin:/usr/local/mysql/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
ENV CFLAGS="-DGPL_LICENSED -Wl,-rpath=/usr/local/lib -Wl,-rpath=/usr/local/mysql/lib/ -Wl,-rpath=/usr/local/lib64 -L/usr/local/mysql/lib/ -L/usr/local/lib64/ -L/usr/lib64/ "

COPY src /src/
RUN ls -la /src/autoconf
RUN bash -c '    \
	cd /src/autoconf && \
	git checkout e5f5e535adec83b146bfc921d9005ecf6a846464 && \
	autoreconf -i && ./configure && make -j60 && make install && \
	cd /src/ganglia && \
	./bootstrap && ./configure --without-memcached --enable-status && make -j60 && make install && \
	mkdir -p /src/cmake/build && \
	cd /src/cmake/build && \
	cmake .. && make -j40 && make install  && \
	cd /src/munge && \
	autoreconf -i && \
	env CFLAGS="-D_GNU_SOURCE " ./configure --with-munge-socket=/var/run/munge/munge.socket.2 --with-crypto-lib=openssl && \
	make -j 60 && \
	make install && \
	\
	cd /src/cron && \
	autoreconf -i && \
	./configure --prefix=/ --enable-syscrontab && \
	make -j 60 && \
	make install && \
	mkdir -m 0744 -p /etc/cron.d/ && \
	chown root:root /etc/cron.d/ && \
	\
	cd /src/hwloc && \
	autoreconf -i && \
	./configure && \
	make -j 60 && \
	make install && \
	\
	cd /src/netloc && \
	autoreconf -i && \
	./configure --with-hwloc=/usr/local/ && \
	make -j 60 && \
	make install && \
	\
	cd /src/pmix && \
	./autogen.sh  && \
	./configure && \
	make -j 60 && \
	make install && \
	cd /src/libssh2 && \
	mkdir bin && \
	cd bin && \
	/usr/local/bin/cmake -D BUILD_SHARED_LIBS=ON .. && \
	/usr/local/bin/cmake -D BUILD_SHARED_LIBS=ON --build . && \
	make -j 60 && \
	make install && \
	cd /src/lz4 && \
	make -j 60 && \
	make install '
RUN bash -c '    \
	cd /src/zstd && make && make install && \
	\
	groupadd -r -g 1000 mysql && \
	useradd -r -u 1000 -M -g mysql mysql && \
	mkdir -p /src/mysql/build  && \
	cd /src/mysql/build && \
	cmake -DMYSQL_DATADIR=/data -DWITH_SSL="system" -DBUILD_CONFIG=mysql_release ../ && \
	cmake ../ -LH ;  false ; \
	make -j128 && \
	make install \
	'
RUN bash -c '    \
	cd /src/slurm && \
	./configure --with-munge=/usr/local/ --sysconfdir=/etc/slurm/ \
	--enable-static  --disable-gtktest --enable-pam --with-hdf5=no --with-freeipmi=no \
	--enable-really-no-cray --enable-salloc-kill-cmd && \
	make -j128 && \
	make install \
    '
COPY SSG_SLURM_Plugins /src/
WORKDIR /src/SSG_SLURM_Plugins/submit_filter/
RUN env slurm_b=/src/slurm slurm_p=/usr/local/ ./build_plugin.sh

RUN ldconfig 

#configure munge
COPY munge.key /etc/munge/munge.key
RUN touch /etc/munge/munge.seed && \
    chown munge:munge -R /etc/munge && \
    chmod 0400 /etc/munge/munge.key /etc/munge/munge.seed 


WORKDIR /
COPY my.cnf /etc/my.cnf
COPY root.my.cnf /root/.my.cnf
COPY slurm /etc/slurm/
RUN chown lsfadmin:lsfadmin -R /etc/slurm 
RUN mkdir -p /data /var/lib/mysql/
RUN chown mysql:mysql /var/lib/mysql/
RUN touch /var/run/slurmdbd.pid && chown lsfadmin:lsfadmin /var/run/slurmdbd.pid

RUN apt-get -y install libnet-ldap-perl
#Setup upd for LDAP
COPY upd /tmp/upd/
RUN \
    cp -vf /tmp/upd/ldapprocess.pl /usr/sbin/ldapprocess.pl && \
    cp -vf /tmp/upd/upd /usr/sbin/upd && \
    mkdir -m 0700 -p /root/ldap && \
    chmod 0750 /usr/sbin/upd /usr/sbin/ldapprocess.pl && \
    bash /usr/sbin/upd 
COPY crontab /etc/cron.d/crontab
RUN chown root:root /etc/cron.d/crontab && \
    chmod 0644 /etc/cron.d/crontab

COPY gmond.conf /usr/local/etc/gmond.conf
COPY get_slurm_stat_clusters /usr/local/sbin/get_slurm_stat_clusters
#####only need to run this once to create the dir...dont call after data is in mysql
#####RUN /usr/local/mysql/scripts/mysql_install_db --datadir=/data --basedir=/usr/local/mysql
#####RUN chown mysql:mysql -R /data/
####
####COPY root.my.cnf /root/.my.cnf
COPY sam2slurm.py /usr/local/sbin/sam2slurm.py 
COPY supervisord.conf /etc/supervisord.conf
CMD /usr/bin/supervisord -c /etc/supervisord.conf
