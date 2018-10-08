DOCKER_TAG	?= remote-syslog
DEBRELEASE	?= stretch
PKGCACHE	?= 192.168.100.10:3142
SYSLOG_DIR      ?= /syslog/
BUILD_ARGS      ?= --rm
RUN_ARGS        ?= docker run -it --network="host" --log-driver=syslog -v /dev/log:/dev/log -v $(SYSLOG_DIR):/syslog -h $(DOCKER_TAG) --name $(DOCKER_TAG)

default: run

build: 
	docker build $(BUILD_ARGS) \
		--build-arg DEBRELEASE=$(DEBRELEASE) \
		--build-arg PKGCACHE=$(PKGCACHE) \
		--build-arg SYSLOG_DIR=$(SYSLOG_DIR) \
		--network=host -t $(DOCKER_TAG) .
stop: 
	docker kill $(DOCKER_TAG) || true
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


