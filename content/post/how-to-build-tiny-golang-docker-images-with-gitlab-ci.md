+++
author = "Sergey Nuzhdin"
categories = ["docker", "infrastructure", "ci/cd", "gitlab", "devops", "golang"]
date = "2017-02-06"
description = "Building tiny docker images for Golang applications with Gitlab"
featured = "golang_docker_gitlab.png"
featuredalt = ""
featuredpath = "date"
linktitle = ""
title = "How to build tiny Golang docker images with Gitlab-CI"

+++


This post will show how to build Golang docker containers using Gitlab-CI.
It's the sequel of my previous posts about [building](http://blog.lwolf.org/post/How-to-build-and-test-docker-images-in-gitlab-ci/) and [optimizing](http://blog.lwolf.org/post/how-to-speed-up-builds-and-save-time-and-money/) docker containers.

I’ve written a small microservice in Golang, and I needed to configure my GitLab to auto build it. It uses  [Glide](https://glide.sh/) as a package manager for Go.

My directory tree for this project looks like this:

```

> $ tree
├── Dockerfile
├── build_base.sh
├── glide.lock
├── glide.yaml
├── handlers.go
├── db.go
├── logger.go
├── main.go
├── middlewares.go
├── utils.go
├── deploy
│   └── <kubernetes-deploy-files>
├── release
│   └── Dockerfile
└── vendor
    ├── github.com
    │   └── ...
    ├──gopkg.in
        └── ...
```

Here I have some ***.go** files with my code. **deploy** directory with Kubernetes manifests. **vendor** folder is for dependencies. **glide.lock** and **glide.yaml** is used by the glide.

I have 2 docker files here, one in the top level and one is inside release directory. First one is to install and build application. The second is to run. Having separate docker images for build and run allow me to have tiny container at the end.

# Dockerfile for build

So I needed docker image with go and glide installed to be able to build my application. I found 2 or 3 golang-glide images, but they are based on ubuntu and weighed around 700Mb.

{{< figure src="/img/2017/02/standards.jpg" alt="another one meme" >}}

So I created just another one, but ~~with black jack and …~~  based on alpine and with the latest versions of Golang and glide.

I’m using it as a base image for my builder.

```
FROM lwolf/golang-glide:0.12.3

ENV APP_PATH=/go/src/gitlab-path/repository-name

RUN mkdir -p $APP_PATH
WORKDIR $APP_PATH

COPY glide.yaml glide.yaml
COPY glide.lock glide.lock

RUN glide install -v

VOLUME ["/build"]

ADD . $APP_PATH
CMD GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -o /build/app
```

It’s pretty simple and straightforward:

- set application path
- add glide files
- install dependencies
- build the application and put into the volume.


# Dockerfile for production

This one is really simple. It based on the tiny image from [iron](https://iron.io/) with added binary from the previous step.

```
FROM iron/go
ENTRYPOINT ["/app"]
ADD app /
```

Resulting container is **17Mb** in size. Which is great, comparing to **300-600Mb** python images.


# Automation

All we have to do now is to write **.gitlab-ci.yml** file to do all this on each push.

**.gitlab-ci.yml** and **build_base.sh** are almost as in the [previous post](http://blog.lwolf.org/post/how-to-speed-up-builds-and-save-time-and-money/).


```
# .gitlab-ci.yml

image: docker:latest

services:
  - docker:dind

stages:
  - build

variables:
  REGISTRY: "docker-registry.example.com"
  CONTAINER_IMAGE: ${REGISTRY}/${CI_PROJECT_NAME}:${CI_BUILD_REF_NAME}_${CI_BUILD_REF}

build:
  stage: build
  before_script:
    - docker login -u gitlab-ci -p $DHUB_ROCKS_PASSWORD $REGISTRY
  script:
    - sh build_base.sh gitlab-ci ${DHUB_ROCKS_PASSWORD} ${REGISTRY}
    - docker run --rm -v $PWD/release:/build ${REGISTRY}/${CI_PROJECT_NAME}:${CI_BUILD_REF_NAME}-builder-$(md5sum glide.lock | cut -d' ' -f1)
    - cd release
    - docker build -t ${CONTAINER_IMAGE} .
    - docker push ${CONTAINER_IMAGE}
```

```
# build_base.sh

#!/usr/bin/env bash
username=${1}
password=${2}
registry=${3}

requirements_hash="$(md5sum glide.lock | cut -d' ' -f1)"
curl -u ${username}:${password} https://${registry}/v2/${CI_PROJECT_NAME}/tags/list | grep ${requirements_hash}

if [ $? -ne 0 ]; then
    tag=${registry}/${CI_PROJECT_NAME}:${CI_BUILD_REF_NAME}-builder-${requirements_hash}
    echo "going to build new base image ${tag}"
    docker build -t ${tag} -f Dockerfile .
    docker push ${tag}
fi
```

First I was using hash from the glide.lock file instead of calculation it. But it turned out that it’s not always updated.

# Conclusion

As a result, I have a pretty fast build right now: **1:30m** for usual builds and **3:00m** on requirements update.

{{< figure src="/img/2017/02/build_time.png" alt="build times" >}}

The size of the resulting image is also great - **17mb**. It could be  reduced for another few megs by switching from **iron/go** base image to **scratch**.  But I don’t need it now.
