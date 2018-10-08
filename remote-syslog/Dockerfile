FROM ppc64le/debian:stretch
ARG DEBRELEASE
ENV DEBRELEASE=${DEBRELEASE}
ARG PKGCACHE
ENV PKGCACHE=${PKGCACHE}
ARG SYSLOG_DIR
ENV SYSLOG_DIR=${SYSLOG_DIR}

RUN bash -c 'echo "Acquire::http::proxy \"http://${PKGCACHE}/\";" > /etc/apt/apt.conf.d/00Proxy '
COPY apt.sources.list /etc/apt/sources.list
RUN bash -c 'sed "s#\${DEBRELEASE}#${DEBRELEASE}#g" -i /etc/apt/sources.list'
RUN apt-get -y update && apt-get dist-upgrade -y
RUN apt-get -y install rsyslog supervisor
RUN groupadd -r -g 69547 syslog

RUN bash -c "mkdir -p /var/spool/ /syslog"

COPY example.rsyslog.conf /etc/rsyslog.conf
COPY supervisord.conf /etc/supervisord.conf
RUN ln -nfs /usr/share/zoneinfo/America/Denver /etc/localtime

CMD /usr/bin/supervisord -c /etc/supervisord.conf
