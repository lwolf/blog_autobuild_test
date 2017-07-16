+++
author = "Sergey Nuzhdin"
categories = ["docker", "infrastructure", "kubernetes", "k8s", "ci/cd", "gitlab", "devops"]
date = "2016-10-10"
description = ""
featured = "gitlab-ci-environments.png"
featuredalt = ""
featuredpath = "date"
linktitle = ""
title = "Continuous deployment to Kubernetes from GitLab-CI."

+++

In my previous posts I described how to [deploy GitLab to Kubernetes](/post/how-to-easily-deploy-gitlab-on-kubernetes/) and [configure GitLab-CI to build and test
docker containers](/post/How-to-build-and-test-docker-images-in-gitlab-ci/).
In this one, I'm going to write about continues deployments.
I assume that you already have Kubernetes cluster and application running,
and you have some manual way of deploying your application to it.
So I will not touch the basics of writing Kubernetes manifests.
But I will briefly describe my own scripts and show how I configured GitLab-CI to automate deployment.

<!-- more -->

My usual project directory looks like this:

```
/project-root
├── src
│   ├── ...
│   ├── Dockerfile
├── deploy
│   ├── app-configmap.yaml
│   ├── app-svc.yaml
│   ├── postgresql
│   │   ├── rc.yaml
│   │   ├── configmap.yaml
│   │   └── svc.yaml
│   ├── rabbitmq
│   │   ├── deployment.yaml
│   │   └── svc.yaml
│   └── tmpl
│       ├── app-deployment.yaml
│       └── celery-deployment.yaml
├── deploy.sh
```

Previously I used to have build.sh script to build and test my container.
Now I don't, since it's done in GitLab-CI.

Deploy directory contains all my Kubernetes manifests, grouped by service name.
For example directory for postgresql with svc,rc,configmap in it.
The same for redis,rabbitmq or whatever. All manifests are plain yaml, and could be deployed anytime.

The interesting part is `tmpl` directory. It contains templates for deployments which should be redeployed very often.
Usually, it's the main application.

Here is a short example of such deployment template.

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: my-new-app
spec:
  replicas: 3
  template:
    metadata:
      labels:
        component: app
        app: my-new-app
    spec:
      containers:
      - name: my-new-app
        image: my-docker-hub/my-new-app:${BUILD_NUMBER}
        imagePullPolicy: Always
```

Basically its plain yaml file with variables. Variables declared the same way as in any bash script.
It will be populated with actual values later, using [envsubst](https://www.gnu.org/software/gettext/manual/html_node/envsubst-Invocation.html)

I do not deploy the application to more than 2 environments (staging, production) so there was no need to automate initial deployment.
That's why for the first time we need to create all our dependent services and databases (all our static manifests) manually.

After this, we can use deploy.sh script to deploy application itself:

```
#!/usr/bin/env bash

TAG=${1}
export BUILD_NUMBER=${TAG}
for f in ./deploy/tmpl/*.yaml
do
  envsubst < $f > "./deploy/.generated/$(basename $f)"
done

kubectl apply -f ./deploy/.generated/

```

It's pretty straightforward: it gets deployment tag as an argument,
iterate through all template manifests and replace all variables with its corresponding env values.
After this, it applies new manifests.


## Updating GitLab-CI

To automate this step we need to add 2 stages to our `.gitlab-ci` file. Original file could be found in my previous [post](/post/How-to-build-and-test-docker-images-in-gitlab-ci/)

```
variables:
  KUBECONFIG: /home/core/deployment-config


deploy_production:
  stage: deploy
  image: lwolf/kubectl_deployer:latest
  script:
    - kubectl config use-context production
    - /bin/sh deploy.sh ${CI_BUILD_REF_NAME}_${CI_BUILD_REF}
  environment: production
  when: manual

deploy_staging:
  stage: deploy
  image: lwolf/kubectl_deployer:latest
  script:
    - kubectl config use-context staging
    - /bin/sh deploy.sh ${CI_BUILD_REF_NAME}_${CI_BUILD_REF}
  environment: staging
```

The first one is for deployment to production and it could be triggered from GitLab web interface.

{{< figure src="/img/2016/10/deploy-production.gif" alt="GitLab deploy to production" >}}

The second one is for staging and it's run after each successful build.
Both stages are using the same docker image, which is simple alpine linux with with kubectl and gettext installed.

I also added KUBECONFIG environment variable which points to kube config with
two configured contexts and mounted as a volume to the GitLab runner.

```
    [[runners]]
      name = "docker-runner"
      ...
      executor = "docker"
      [runners.docker]
        ...
        volumes = ["/home/core/deployment-config:/home/core/deployment-config"]
```

I didn't find a way to use Kubernetes secrets or mount a directory from the pod to the GitLab-CI runner service.
So, I ended up adding volume from the host itself.

That's it. Now all successful builds will be automatically deployed to the staging environment.
Also, you can do redeploy and rollbacks, which is basically just a re-run of the deployment stage.

