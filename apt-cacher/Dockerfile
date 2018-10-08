FROM ppc64le/debian:latest
RUN apt-get -y update && apt-get upgrade -y
RUN apt-get -y install apt-cacher-ng
ADD apt-cacher-ng-etc /etc/apt-cacher-ng
RUN chown apt-cacher-ng:apt-cacher-ng -R /etc/apt-cacher-ng
CMD bash -c '/usr/sbin/apt-cacher-ng -c /etc/apt-cacher-ng && tail -qF /var/log/apt-cacher-ng/*'
