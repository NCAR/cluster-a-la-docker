DOCKER_TAG	?= mysql
DEBRELEASE	?= stretch
PKGCACHE		?= 192.168.100.10:3142
IP					?= 192.168.100.17
MYSQL_DB        ?= /srv/mysql
RUN_ARGS        ?= docker run -it --network="bridge-nat" --ip "$(IP)" --log-driver=syslog -v /dev/log:/dev/log -v $(MYSQL_DB):/data -h $(DOCKER_TAG) --name $(DOCKER_TAG)
BUILD_ARGS      ?= --rm

default: run

build: 
	docker build $(BUILD_ARGS) \
		--build-arg DEBRELEASE=$(DEBRELEASE) \
		--build-arg PKGCACHE=$(PKGCACHE) \
		--build-arg IP=$(IP) \
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


