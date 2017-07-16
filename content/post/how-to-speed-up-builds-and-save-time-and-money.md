+++
author = "Sergey Nuzhdin"
categories = ["docker", "infrastructure", "kubernetes", "k8s", "ci/cd", "gitlab", "devops"]
date = "2016-12-01"
description = ""
featured = "buildtime.png"
featuredalt = ""
featuredpath = "date"
linktitle = ""
title = "How to speed up builds and save time and money"

+++


In one of the [previous posts](/post/How-to-build-and-test-docker-images-in-gitlab-ci/), I described the way I'm using GitLab to build and test images.
Despite the fact that it's pretty simple configuration and actual tests are run for less than a minute,
complete pipeline takes around 20 minutes. Which seems very unreasonable.
I spent some time digging out the problem and was able to reduce build time to around 3 minutes. 

Here are few things, that should be considered to optimize pipeline.
They are listed in the order I was applying them and not by effectiveness. 

# Caching everything

The first thing that came to my mind when I started optimisation was **slow package download speed**.
So I deployed my caching mirrors for [**pip**](https://hub.docker.com/r/muccg/devpi/) and [**npm**](https://hub.docker.com/r/apicht/npm_lazy/).

But it almost did not help. All I got is speedup of around 1-2 minutes.

Also, I tried to deploy my docker mirroring registry.
But it turned out that it works only for official docker registry and do not support push.

# Reduce docker size images

Next thing I noticed was the amount of time spent to pull/push docker images.
It was about 80% of the time. 

It could sound obvious, but sometimes you do not look at the size of docker images.
Especially when you develop locally. Time does not matter for a one-time pull.
But in the build system, when it has to be done on each commit it starts to matter.

I'm trying to use official docker images for most of the stuff.
Since I faced some strange network-related issues with alpine-based images some time ago,
I was trying to avoid it where it's possible, until now.

I knew that python image is 685mb, but I never thought that the difference 
between it and alpine-based version is almost 8 times - 88mb

So, I switched all my Dockerfiles to alpine-based images. And this reduced build time in a half.  

Still, it was not fast enough.

# Reduce number of ci steps

Originally I configured my [gitlab-ci file](/post/How-to-build-and-test-docker-images-in-gitlab-ci/) to have one stage for each logical action:

  * compile - to compile all js and css
  * build - to build test image
  * test - test the image we've got on previous step
  * release - push release image

It looks logical and organized, but in fact build-test-release stages
 are spending most of the time pulling and pushing the same image.

So, I merged build/test/release steps into one. The image is pushed if and only if tests are passed.
In this case, I don't need to pull and push images several times.

After this change, build time reduced to around 5 minutes.

Which is really great after 20 minutes. But I still thought that it could be reduced 
by avoiding installation of the same packages on each build.

# Rebuild base image only when list of packages changed

The idea was to have 2 images for the project, in my case python backend. But it could be used everywhere.

The first **base** image with all the requirements/dependencies installed.
It should be rebuilt only when the list of packages was changed.

The second one is based on the first one. It should just add application code to the container.
It should be rebuilt on each build.

My Dockerfile-base looks like this.

```
FROM python:3.5-alpine

MAINTAINER Sergii Nuzhdin <ipaq.lw@gmail.com>

RUN apk add --update ca-certificates \
    && apk add postgresql-dev libjpeg tiff-dev zlib-dev libwebp-dev gcc musl-dev linux-headers \
    && rm /var/cache/apk/*

ARG PIP_INDEX_URL
ENV PIP_INDEX_URL=${PIP_INDEX_URL}

ARG PIP_TRUSTED_HOST
ENV PIP_TRUSTED_HOST=${PIP_TRUSTED_HOST}

ADD requirements.txt /opt/requirements.txt
RUN pip install -r /opt/requirements.txt
```

And my Dockerfile looks like this

```
FROM lwolf/project:master-base-latest

ENV INSTALL_DIR=/app

RUN mkdir -p $INSTALL_DIR

WORKDIR $INSTALL_DIR

ADD . $INSTALL_DIR

EXPOSE 9000

CMD python manage.py runserver

```

The only thing left was to actually trigger the rebuild of the base image on requirements change. 
I came up with the next bash script:

```
#!/usr/bin/env bash
username=${1}
password=${2}
registry=${3}

requirements_hash="$(md5sum requirements.txt | cut -d' ' -f1)"
curl -u ${username}:${password} https://${registry}/v2/${CI_PROJECT_NAME}/tags/list | grep ${requirements_hash}
if [ $? -ne 0 ]; then
    tag=${registry}/${CI_PROJECT_NAME}:${CI_BUILD_REF_NAME}-base-${requirements_hash}
    tag_latest=${registry}/${CI_PROJECT_NAME}:${CI_BUILD_REF_NAME}-base-latest
    docker build -t ${tag} -f Dockerfile-base .
    docker tag ${tag} ${tag_latest}
    docker push ${tag}
    docker push ${tag_latest}
fi
```

It's pretty straight-forward.
Get the hash of the requirements.txt file, check that base image with this version exists.
If it does not - build and push it.
  
I added this 2 lines to .gitlab-ci.yml to use the corresponding base image instead of the latest.

```
build:
  stage: build
  script:
    ...
    - sh build_base.sh gitlab-ci ${REGISTRY_PASSWORD} ${REGISTRY}
    - sed -i -e "s/master-base-latest/master-base-$(md5sum requirements.txt | cut -d' ' -f1)/g" Dockerfile
    ...

```


With this last change I've achieved build time in the range of 2-3 minutes.
With the increase to around 5 minutes when requirements are changed.

# Conclusion

Just reducing the docker images size and the number of CI steps will save you a lot of time.
In my case, it saved me around 70% of build time. 

Next one, in order of value it provides, is to not reinstall all packages on each build.

The last one is local mirrors of packages.

Using this advice you'll be able to drastically reduce build time.
And save a lot of money, both on pay-per-minute costs of infrastructure and development time. 
