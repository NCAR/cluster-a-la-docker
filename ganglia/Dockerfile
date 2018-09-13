FROM ppc64le/debian:stretch
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
RUN apt-get -y install supervisor git build-essential automake cmake libtool-bin m4 texi2html \
    autoconf autopoint pkg-config wget tar gzip help2man libapr1-dev libconfuse-dev \
    libexpat1-dev libpcre3-dev libz-dev libmemcached-dev librrd-dev rrdtool rrdcached \
    nginx php-fpm php-mysql php-xml memcached xinetd libck-dev texinfo aspell

RUN echo 'export PATH=/usr/local/bin:$PATH' >> /etc/profile.d/local.sh
RUN chmod +x /etc/profile.d/local.sh
RUN cat /etc/profile.d/local.sh

RUN mkdir -p /tmp/ganglia
WORKDIR /
RUN git clone --single-branch https://github.com/ganglia/monitor-core.git /tmp/ganglia -b 'release/3.6'
WORKDIR /tmp/ganglia
ENV CFLAGS='-Wl,-rpath /usr/local/lib/ -Wl,-rpath .' 
RUN ./bootstrap
#RUN autoreconf -i
RUN ./configure --enable-status --with-memcached --with-gmetad
RUN make -j140
RUN make install

#COPY apache2.conf /etc/apache2/apache2.conf
#RUN mkdir -m 0770 -p /var/run/apache2 && chown www-data:www-data /var/run/apache2
RUN usermod -s /bin/bash www-data
RUN mkdir -p /run/php; chown www-data /run/php
RUN ln -s /usr/sbin/php-fpm7.0 /usr/sbin/php-fpm
COPY fpm-php.ini /etc/php/7.0/fpm/php.ini

RUN mkdir -m 0755 -p /srv/
WORKDIR /srv
RUN git clone https://github.com/ganglia/ganglia-web.git ganglia
WORKDIR /srv/ganglia
COPY ganglia-web-config.php conf_default.php
RUN cp version.php.in version.php
RUN mkdir -m 0770 -p /srv/ganglia/conf /tmp/dwoo/compiled /tmp/dwoo/cache
RUN chown -R www-data:www-data /srv/ganglia/conf /tmp/dwoo

COPY config /etc/ganglia
RUN groupadd -r -g 1000 ganglia
RUN useradd -r -u 1000 -g 1000 ganglia

COPY nginx.conf /etc/nginx/nginx.conf
RUN rm -Rvf /etc/xinetd.d; mkdir -m 0755 /etc/xinetd.d
COPY rrdtool.xinetd /etc/xinetd.d/rrdsrv
COPY xinetd.conf /etc/xinetd.conf
RUN echo 'rrdsrv 13900/tcp' >> /etc/services

COPY supervisord.conf /etc/supervisord.conf
CMD /usr/bin/supervisord -c /etc/supervisord.conf
