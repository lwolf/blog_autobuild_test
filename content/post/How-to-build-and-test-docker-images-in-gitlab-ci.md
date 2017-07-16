+++
author = "Sergey Nuzhdin"
categories = ["docker", "infrastructure", "ci/cd", "gitlab", "devops"]
date = "2016-09-21"
description = "How to configure GitLab CI to build,test and release python containers with javascript and webpack."
featured = "gitlab-ci.png"
featuredalt = ""
featuredpath = "date"
linktitle = ""
title = "How to build and test docker images in GitLab CI."
aliases = [
    "/post/How-to-build-and-test-docker-images-in-gitlab-ci/"
]

+++

In the [previous post](/post/how-to-easily-deploy-gitlab-on-kubernetes/) I described how to run own GitLab server with CI runner.
In this one, I'm going to walk through my experience of configuring GitLab-CI for one of my projects.
I faced few problems during this process, which I will highlight in this post.

Some words about the project:

  * Python/Flask backend with PostgreSQL as a database, with the bunch of unittests.
  * React/Reflux in frontend with Webpack for bundling. No JS tests.
  * Frontend and backend are bundled in a single docker container and deployed to Kubernetes


## Building the first container

First thing after importing repository into gitlab we need to create `.gitlab-ci.yml`.
Gitlab itself has a lot of useful information about CI configuration,
for example [this](https://docs.gitlab.com/ce/ci/quick_start/README.html) and
[this](https://docs.gitlab.com/ce/ci/yaml/README.html).

Previously I used to do following steps to test/compile/deploy manually:

* npm run build - to build all js/css
* build.sh - to build docker container
* docker-compose -f docker-compose-test.yml run --rm api-test nosetests
* deploy.sh - to deploy it

Everything is pretty common for most of the web projects.
So, I started with a simple `.gitlab-ci.yml` file, trying to build my python code:

```
image: docker:latest

services:
  - docker:dind

stages:
- build

variables:
  CONTAINER_TEST_IMAGE: my-docker-hub/$CI_PROJECT_ID:$CI_BUILD_REF_NAME_test

before_script:
  - docker login -u gitlab-ci-token -p $CI_BUILD_TOKEN my-docker-hub

build:
  stage: build
  script:
    - docker build --pull -t $CONTAINER_TEST_IMAGE .
    - docker push $CONTAINER_TEST_IMAGE

```

- `$CI_BUILD_TOKEN` - is a password for my docker registry, saved in projects variables in GitLab project settings


And it actually worked fine. I got my docker container built and pushed to the registry.
Next step was to be able to run tests. This is where things became not so obvious.
For testing, we need to add another stage to our gitlab-ci file.


## Testing. First try

After reading posts like [this](https://docs.gitlab.com/ce/ci/docker/using_docker_images.html)
I thought that it will be as easy as declaring service and adding test stage:

```
image: docker:latest

stages:
  - build
  - test

services:
  - docker:dind
  - postgres:9.5

test:
  stage: test
  script:
    - docker run --link=postgres:postgres $CONTAINER_TEST_IMAGE nosetests --with-coverage --cover-erase --cover-package=${CI_PROJECT_NAME} --cover-html
```

But it didn't work. As far as I understood it, it's because of docker-in-docker runner.
Basically, we have 2 containers - docker:latest and postgresql:9.5, and they are linked perfectly fine.
But then we're bootstrapping our own container, inside this docker container.
And this container can't use docker links to access postgres, because its outside of its scope.


## Testing. Second try

Then I tried to use my container as and image for the test stage and service declared in test stage itself, like this:

```
test:
  stage: test
  image: $CONTAINER_TEST_IMAGE
  services:
    - postgres:9.5
  script:
    - nosetests --with-coverage --cover-erase --cover-package=${CI_PROJECT_NAME} --cover-html
```

But it also didn't work because of authentication.

```
ERROR: Preparation failed: unauthorized: authentication required
```

My docker registry needs authentication which I do in `before_script` stage. But before_script is being called after the services.
I assume that this method should work for public images.


## Testing. Third try. Kinda working solution

So I decided to try to use docker-compose in tests as I was doing manually since.
Docker-compose should be able to run and link everything together.
And since this [page](https://docs.gitlab.com/ce/ci/docker/using_docker_build.html) in documentation says:


> # Using Docker Build
> GitLab CI allows you to use Docker Engine to build and test docker-based projects.

> **This also allows to you to use docker-compose and other docker-enabled tools.**


I was very confused when I was not able to use docker-compose, since docker:latest image has no docker-compose installed.
I spent some time googling and trying to install compose inside the container,
and ended up using image [jonaskello/docker-and-compose](https://hub.docker.com/r/jonaskello/docker-and-compose/) instead of the recommended one.

So my test stage changed to this:

```
image: jonaskello/docker-and-compose:1.12.1-1.8.0
...
test:
  stage: test
  before_script:
    - docker-compose -f docker-compose-test.yml pull
    - docker-compose -f docker-compose-test.yml up -d
  script:
    - docker-compose -f docker-compose-test.yml run --rm api-test nosetests --with-coverage
  after_script:
    - docker-compose -f docker-compose-test.yml stop
    - docker-compose -f docker-compose-test.yml rm -f
```

This actually worked, but from time to time I was seeing weird race conditions during database provisioning.
It's not a big problem, and could be fixed easily. But I decided to try one more approach.

## Testing. Final version

This time I decided to run postgres container during the test stage and link it to my test container.
This requires to provide additional configuration to postgres container, but still this way is the closest to the original services approach.

```
test:
  stage: test
  script:
    - docker run -d --env-file=.postgres-env postgres:9.5
    - docker run --env-file=.environment --link=postgres:db $CONTAINER_TEST_IMAGE nosetests --with-coverage --cover-erase --cover-package=${CI_PROJECT_NAME} --cover-html
```


# Compiling static

Now, when we have our python container built and tested we need to add one more thing.
We need to compile our javascript and css and put it into release container.
This stage is actually going before the build.
To be able to use files between stages we need to use some cache.
In my case we need to cache only one directory.
The one where webpack produces compiled files - `src/static/dist`.

```

cache:
  key: "$CI_BUILD_REF"
  paths:
    - src/static/dist


compile:
  stage: compile
  image: iteamdev/node-webpack:latest
  script:
    - npm run deploy

```

Cache is described in more details in [official docs](https://docs.gitlab.com/ce/ci/yaml/README.html#cache)


# Release stage and the final config

As for release stage I'm simply going to tag container with build number and push it back to the registry.

```
image: docker:latest

services:
  - docker:dind

stages:
  - compile
  - build
  - test
  - release

cache:
  key: "$CI_BUILD_REF"
  paths:
    - src/static/dist

variables:
  STAGING_REGISTRY: "my-docker-hub"
  CONTAINER_TEST_IMAGE: ${STAGING_REGISTRY}/${CI_PROJECT_NAME}:${CI_BUILD_REF_NAME}_${CI_BUILD_REF}_test
  CONTAINER_RELEASE_IMAGE: ${STAGING_REGISTRY}/${CI_PROJECT_NAME}:${CI_BUILD_REF_NAME}_${CI_BUILD_REF}

before_script:
  - docker login -u gitlab-ci -p $CI_BUILD_TOKEN $STAGING_REGISTRY

compile:
  stage: compile
  image: iteamdev/node-webpack:latest
  script:
    - npm run deploy

build:
  stage: build
  script:
    - cd src
    - docker build --pull -t $CONTAINER_TEST_IMAGE .
    - docker push $CONTAINER_TEST_IMAGE

test:
  stage: test
  script:
    - docker run -d --env-file=.postgres-env postgres:9.5
    - docker run --env-file=.environment --link=postgres:db $CONTAINER_TEST_IMAGE nosetests --with-coverage --cover-erase --cover-package=${CI_PROJECT_NAME} --cover-html

release:
  stage: release
  script:
    - cd src
    - docker pull $CONTAINER_TEST_IMAGE
    - docker tag $CONTAINER_TEST_IMAGE $CONTAINER_RELEASE_IMAGE
    - docker push $CONTAINER_RELEASE_IMAGE

```

Now we have a working CI pipeline which can build a container, run tests against *this* container and then push it to the registry.
So far so good. The only problem is that all of this takes very long time. For my project, it takes about 20 minutes to finish all these stages.
Most of the time is spent on building docker layers, downloading python and npm packages and installing it.
Next post will probably be about reducing this time by using some local caching services.