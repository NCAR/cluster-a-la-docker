FROM ppc64le/debian:stretch
ARG DEBRELEASE
ENV DEBRELEASE=${DEBRELEASE:-stretch}
ARG PKGCACHE
ENV PKGCACHE=${PKGCACHE:-pkg-cache.ssg.ucar.edu:3142}

RUN bash -c 'echo "deb http://${PKGCACHE}/ftp.debian.org/debian ${DEBRELEASE} main contrib non-free" > /etc/apt/sources.list'
RUN bash -c 'echo "deb-src http://${PKGCACHE}/ftp.debian.org/debian ${DEBRELEASE} main contrib non-free" >> /etc/apt/sources.list' 
RUN bash -c 'echo "deb http://${PKGCACHE}/ftp.debian.org/debian ${DEBRELEASE}-updates main contrib non-free" >> /etc/apt/sources.list' 
RUN bash -c 'echo "deb-src http://${PKGCACHE}/ftp.debian.org/debian ${DEBRELEASE}-updates main contrib non-free" >> /etc/apt/sources.list' 
RUN bash -c 'echo "deb http://${PKGCACHE}/security.debian.org ${DEBRELEASE}/updates main contrib non-free" >> /etc/apt/sources.list' 
RUN bash -c 'echo "deb-src http://${PKGCACHE}/security.debian.org ${DEBRELEASE}/updates main contrib non-free" >> /etc/apt/sources.list'
RUN echo 1
RUN apt-get -y update && apt-get dist-upgrade -y
RUN apt-get -y install monitoring-plugins-basic git make autoconf gcc unzip gzip libgd-dev apache2 supervisor cron php libtool libgettextpo-dev libgettextpo0 gettext m4 dnsutils libmariadbclient-dev-compat libssl-dev wget libmcrypt-dev libmcrypt4 python-argh python python-setuptools python-lxml clustershell
RUN a2enmod cgi.load
RUN groupadd -r nagios 
RUN useradd -g nagios -m -r -s /bin/bash nagios

RUN git clone https://github.com/NagiosEnterprises/nagioscore.git
WORKDIR /nagioscore
RUN git checkout nagios-4.3.4
RUN ./configure
RUN make -j 120 all
RUN make fullinstall

WORKDIR /
RUN git clone https://github.com/nagios-plugins/nagios-plugins.git
WORKDIR /nagios-plugins
RUN bash autogen.sh
RUN make -j 120 all
RUN make install

WORKDIR /
RUN git clone https://github.com/nateucar/nsca.git
#RUN git clone https://github.com/NagiosEnterprises/nsca.git
WORKDIR /nsca
RUN ./configure --host=ppc64le-unknown-linux-gnu --build=ppc64le-unknown-linux-gnu --with-nsca-user=nagios --with-nsca-grp=nagios --with-nsca-port=5667
RUN make -j 120 all
RUN cp src/nsca /usr/sbin/
RUN cp src/send_nsca /usr/bin/
WORKDIR /
RUN rm -Rf /nsca

WORKDIR /
RUN git clone https://github.com/NagiosEnterprises/nsca.git
WORKDIR /nsca
RUN git checkout nsca-2-7-2
RUN ./configure --host=ppc64le-unknown-linux-gnu --build=ppc64le-unknown-linux-gnu --with-nsca-user=nagios --with-nsca-grp=nagios --with-nsca-port=5667
RUN make -j 120 all
RUN cp src/send_nsca /usr/bin/send_nsca-2.7.2
WORKDIR /
RUN rm -Rf /nsca

WORKDIR /
COPY apache2.conf /etc/apache2/apache2.conf
RUN mkdir -m 0770 -p /var/run/apache2 && chown www-data:www-data /var/run/apache2
COPY nagios_config /etc/nagios
RUN ln -s /etc/nagios /usr/local/nagios/etc
RUN mkdir -m 0770 -p /var/log/nagios/spool/checkresults /var/spool/nagios/cmd/ && chown nagios:nagios -R /var/log/nagios /etc/nagios /var/spool/nagios
RUN htpasswd -c -b -s /etc/apache2/htpasswd.users nagiosadmin 'PASSWORD'
RUN htpasswd -b -s /etc/apache2/htpasswd.users ssg 'PASSWORD'
RUN chown www-data:www-data /etc/apache2/htpasswd.users
RUN usermod -s /bin/bash www-data
RUN usermod -s /bin/bash nagios

WORKDIR /
RUN git clone https://github.com/larsks/check_ganglia
WORKDIR /check_ganglia
RUN  python setup.py install

WORKDIR /
RUN git clone https://github.com/cronie-crond/cronie.git
WORKDIR /cronie
RUN bash autogen.sh
RUN ./configure --prefix=/ --enable-syscrontab 
RUN make -j120
RUN make install

WORKDIR /
RUN git clone https://github.com/asuknath/Nagios-Status-JSON.git nathstatus
RUN sed 's#/usr/local/nagios/var/status.dat#/var/log/nagios/status.dat#g' /nathstatus/nath_status.php > /usr/local/nagios/share/nath_status.php

COPY root.crontab /etc/cron.d/crontab
RUN chmod 0644 /etc/cron.d/crontab 
COPY nagios-now-check.py /usr/bin/nagios-now-check.py 
COPY nagios-now-config.json /etc/nagios-now-config.json
COPY send_nsca_central.cfg send_nsca_ssgnagios.cfg /etc/nagios/

#disable mail since nagios just spams
RUN ln -s /bin/true /bin/mail

WORKDIR /

#cloned from ops_mgt
RUN mkdir -m 0700 /root/.ssh
COPY root.id_ecdsa /root/.ssh/id_ecdsa 
COPY root.id_ecdsa.pub /root/.ssh/id_ecdsa.pub
COPY root.id_rsa /root/.ssh/id_rsa 
COPY root.id_rsa.pub /root/.ssh/id_rsa.pub
COPY root.known_hosts /root/.ssh/known_hosts

COPY generate_ssg_cfg.sh /usr/bin/generate_ssg_cfg.sh
RUN /usr/bin/generate_ssg_cfg.sh > /etc/nagios/clusters.cfg

COPY supervisord.conf /etc/supervisord.conf
CMD /usr/bin/supervisord -c /etc/supervisord.conf
