DOCKER_TAG	?= pxemaster
DEBRELEASE	?= stretch
PKGCACHE	?= pkg-cache.ssg.ucar.edu:3142
MGT_DEV		?= cloudeth
DHCP_SUBNET	?= 10.20.3.0
DHCP_NETMASK	?= 255.255.255.0
DHCP_POOL_START	?= 10.20.3.100
DHCP_POOL_STOP	?= 10.20.3.200
SRV_IP		?= 10.20.3.249
SYSLOG_IP	?= $(SRV_IP):514
CENTOS_RELEASE	?= 7.4.1708
MOFED_TARBALL	?= MLNX_OFED_LINUX-4.1-1.0.2.0-rhel7.4-x86_64.tgz
MSMTP_TARBALL	?= msmtp-1.6.6.tar.xz
RUN_ARGS        ?= docker run --network=host --cap-add NET_ADMIN -it --log-driver=syslog -h $(DOCKER_TAG) -v /dev/log:/dev/log --name $(DOCKER_TAG)
BUILD_ARGS      ?= --rm 

default: run

build: 
	docker build $(BUILD_ARGS) \
		--build-arg DEBRELEASE=$(DEBRELEASE) \
		--build-arg PKGCACHE=$(PKGCACHE) \
		--build-arg MGT_DEV=$(MGT_DEV) \
		--build-arg DHCP_SUBNET=$(DHCP_SUBNET) \
		--build-arg DHCP_NETMASK=$(DHCP_NETMASK) \
		--build-arg DHCP_POOL_START=$(DHCP_POOL_START) \
		--build-arg DHCP_POOL_STOP=$(DHCP_POOL_STOP) \
		--build-arg SRV_IP=$(SRV_IP) \
		--build-arg SYSLOG_IP=$(SYSLOG_IP) \
		--build-arg CENTOS_RELEASE=$(CENTOS_RELEASE) \
		--build-arg MOFED_TARBALL=$(MOFED_TARBALL) \
		--build-arg MSMTP_TARBALL=$(MSMTP_TARBALL) \
		--network=host -t $(DOCKER_TAG) .

stop: 
	docker stop $(DOCKER_TAG) || true
	docker rm $(DOCKER_TAG) || true

nocache:
	$(eval BUILD_ARGS := $(BUILD_ARGS) --no-cache)

clean: nocache build

run: build stop
	$(RUN_ARGS) -d --restart unless-stopped $(DOCKER_TAG) 

debug: build stop
	$(RUN_ARGS) --rm $(DOCKER_TAG) 

bash: build stop
	$(RUN_ARGS) --rm $(DOCKER_TAG) /bin/bash

regen:
	rm -vf client/ssh/ssh_host_dsa_key client/ssh/ssh_host_dsa_key.pub client/ssh/ssh_host_ecdsa_key \
	    client/ssh/ssh_host_ecdsa_key.pub client/ssh/ssh_host_ed25519_key client/ssh/ssh_host_ed25519_key.pub \
	    client/ssh/ssh_host_rsa_key client/ssh/ssh_host_rsa_key.pub
	ssh-keygen -C '' -N '' -b 1024 -t dsa -f client/ssh/ssh_host_dsa_key
	ssh-keygen -C '' -N '' -b 4096 -t rsa -f client/ssh/ssh_host_rsa_key
	ssh-keygen -C '' -N '' -b 521 -t ecdsa -f client/ssh/ssh_host_ecdsa_key
	ssh-keygen -C '' -N '' -t ed25519 -f client/ssh/ssh_host_ed25519_key
	ssh-keygen -y -f client/ssh/ssh_host_dsa_key > client/ssh/ssh_host_dsa_key.pub
	ssh-keygen -y -f client/ssh/ssh_host_rsa_key > client/ssh/ssh_host_rsa_key.pub
	ssh-keygen -y -f client/ssh/ssh_host_ed25519_key > client/ssh/ssh_host_ed25519_key.pub
	ssh-keygen -y -f client/ssh/ssh_host_ecdsa_key > client/ssh/ssh_host_ecdsa_key.pub
	dd if=/dev/random bs=1 count=1024 client/munge.key

download:
	mkdir -p src/
	wget 'http://mirrors-usa.go-parts.com/gcc/releases/gcc-6.4.0/gcc-6.4.0.tar.xz' -O src/gcc.640.tar.xz
	wget 'http://mirrors-usa.go-parts.com/gcc/releases/gcc-7.3.0/gcc-7.3.0.tar.xz' -O src/gcc.730.tar.xz
	wget 'ftp://gcc.gnu.org/pub/gcc/infrastructure/isl-0.18.tar.bz2' -O src/isl-0.18.tar.bz2
	wget 'http://prdownloads.sourceforge.net/scons/scons-3.0.1.tar.gz' -O src/scons-3.0.1.tar.gz
	git clone --single-branch https://github.com/martinh/libconfuse.git src/confuse -b v3.2.1
	git clone --single-branch https://github.com/ganglia/monitor-core.git src/ganglia -b release/3.6
	git clone --single-branch git://git.sv.gnu.org/autoconf src/autoconf/2.69 -b v2.69
	git clone --single-branch git://git.sv.gnu.org/autoconf src/autoconf/2.64 -b v2.64
	git clone --single-branch https://github.com/Kitware/CMake.git src/cmake -b v3.1.3
	git clone --single-branch https://github.com/mej/nhc.git src/nhc -b 1.4.2
	git clone --single-branch https://github.com/ruby/ruby.git src/ruby -b v2_4_4
	git clone --single-branch https://github.com/OpenNebula/one.git src/opennebula -b release-5.4.6 

