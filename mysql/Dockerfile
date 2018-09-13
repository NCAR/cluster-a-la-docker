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
RUN apt-get -y install supervisor build-essential autoconf automake git cmake git \
 cmake-curses-gui libncurses5-dev libgnutls28-dev bison libjemalloc-dev libzmq3-dev \
 libbz2-dev libsnappy-dev libboost-dev liblzma-dev ruby-dev python-dev libaio-dev 

RUN groupadd -r -g 1000 mysql
RUN useradd -r -u 1000 -M -g mysql mysql

RUN git clone https://github.com/MariaDB/server.git
WORKDIR /server/
RUN git checkout 10.2
RUN mkdir build
WORKDIR /server/build
RUN cmake -DBUILD_CONFIG=mysql_release ../
RUN make -j128
RUN make -j128 install
WORKDIR /
RUN rm -Rf /server
RUN echo 1
COPY my.cnf /etc/my.cnf
#only need to run this once to create the dir...dont call after data is in mysql
#RUN /usr/local/mysql/scripts/mysql_install_db --datadir=/data --basedir=/usr/local/mysql
#RUN chown mysql:mysql -R /data/
RUN ln -s /usr/local/mysql/bin/mysql /usr/bin/mysql
RUN ln -s /usr/local/mysql/bin/mysqldump /usr/bin/mysqldump

COPY root.my.cnf /root/.my.cnf
COPY supervisord.conf /etc/supervisord.conf
CMD /usr/bin/supervisord -c /etc/supervisord.conf


