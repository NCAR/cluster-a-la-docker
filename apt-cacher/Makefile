DOCKER_TAG	?= apt-cacher-ng
IP		?= 192.168.100.10 
RUN_ARGS        ?= docker run --network="bridge-nat" --ip $(IP) -h $(DOCKER_TAG) --log-driver=syslog -v /dev/log:/dev/log --name $(DOCKER_TAG) 
BUILD_ARGS      ?= --rm 

default: run

build: 
	docker build $(BUILD_ARGS) \
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


