DOCKER_TAG	?= slurm
DEBRELEASE	?= stretch
PKGCACHE	?= pkg-cache.ssg.ucar.edu:3142
IP		?= 192.168.100.21
DATA_PATH	?= /srv/slurm/
SLURM_PLUGINS_PATH ?= /ssg/git/SSG_SLURM_Plugins/
RUN_ARGS        ?= docker run -it --ulimit core=99999999999:99999999999 --network="bridge-nat" --ip "$(IP)" --log-driver=syslog -v $(DATA_PATH):/data/ -v /dev/log:/dev/log -h slurm.ssg.ucar.edu --name $(DOCKER_TAG)
BUILD_ARGS      ?= --rm
RUNNING 	:= $(shell docker ps -f "name=$(DOCKER_TAG)" | wc -l)

default: run

build:
	rsync --delete --cvs-exclude -avp /ssg/git/SSG_SLURM_Plugins SSG_SLURM_Plugins
	docker build $(BUILD_ARGS) \
		--build-arg DEBRELEASE=$(DEBRELEASE) \
		--build-arg PKGCACHE=$(PKGCACHE) \
		--build-arg IP=$(IP) \
		--network=host -t $(DOCKER_TAG) .
stop: 
ifeq "$(RUNNING)" "2"
	docker exec -it $(DOCKER_TAG) supervisorctl stop slurmctld
	docker exec -it $(DOCKER_TAG) supervisorctl stop slurmdbd
	docker exec -it $(DOCKER_TAG) mysqladmin --shutdown_timeout 60 shutdown
	docker exec -it $(DOCKER_TAG) supervisorctl stop all
	docker stop $(DOCKER_TAG)
	docker rm $(DOCKER_TAG)
else
	echo "skipping stop as $(DOCKER_TAG) is already stopped"
endif

nocache:
	$(eval BUILD_ARGS := $(BUILD_ARGS) --no-cache)

clean: nocache build

run: build stop
	$(RUN_ARGS) -d --restart unless-stopped $(DOCKER_TAG) 

debug: build stop
	$(RUN_ARGS) --rm $(DOCKER_TAG) 

bash: build stop
	$(RUN_ARGS) --rm $(DOCKER_TAG) /bin/bash

download:
	rm -Rvf src
	git clone --recurse-submodules --shallow-submodules --single-branch git://git.sv.gnu.org/autoconf src/autoconf -b master
	git clone --depth 1 --recurse-submodules --shallow-submodules --single-branch https://github.com/Kitware/CMake.git src/cmake -b v3.11.4
	git clone --depth 1 --recurse-submodules --shallow-submodules --single-branch https://github.com/dun/munge.git src/munge -b master
	git clone --depth 1 --recurse-submodules --shallow-submodules --single-branch https://github.com/ganglia/monitor-core.git src/ganglia -b release/3.6
	git clone --depth 1 --recurse-submodules --shallow-submodules --single-branch https://github.com/open-mpi/hwloc.git src/hwloc -b hwloc-1.11.8
	git clone --depth 1 --recurse-submodules --shallow-submodules --single-branch https://github.com/open-mpi/netloc.git src/netloc -b v0.5
	git clone --depth 1 --recurse-submodules --single-branch https://github.com/MariaDB/server.git src/mysql -b 10.3
	git clone --depth 1 --recurse-submodules --shallow-submodules --single-branch https://github.com/pmix/pmix.git src/pmix -b v2.0.3
	git clone --depth 1 --recurse-submodules --shallow-submodules --single-branch https://github.com/cronie-crond/cronie.git src/cron -b cronie-1.5.2
	git clone --depth 1 --recurse-submodules --shallow-submodules --single-branch https://github.com/lz4/lz4.git src/lz4 -b v1.8.1.2
	git clone --depth 1 --recurse-submodules --shallow-submodules --single-branch https://github.com/libssh2/libssh2.git src/libssh2 -b libssh2-1.8.0
	git clone --depth 1 --recurse-submodules --shallow-submodules --single-branch https://github.com/facebook/zstd.git src/zstd -b zstd-0.4.2
	git clone --depth 1 --recurse-submodules --shallow-submodules --single-branch https://github.com/SchedMD/slurm.git src/slurm -b slurm-17-11-7-1


