+++
author = "Sergey Nuzhdin"
categories = ["nginx", "docker", "infrastructure", "kubernetes"]
date = "2016-06-08"
description = "Deploying docker registry with UI and tls for kubernetes cluster"
featured = "private-containers.png"
featuredalt = ""
featuredpath = "date"
linktitle = ""
title = "Hosting own docker registry with UI and tls"

+++
First thing you need if you're using Kubernetes - Docker registry. Because its all about containers.
So in this post I will show how to deploy your own registry inside Kubernetes cluster,
with UI and tls, with basic http authentication.
<!-- more -->

I'm going to use cluster I deployed in previous [post](/post/migrate-infrastructure-to-kubernetes-building-baremetal-cluster/).
As short recap - we have Kubernetes cluster with few nodes, and external loadbalancer (ubuntu based machine with nginx)
{{< figure src="/img/2016/05/kube-schema-simple.png" alt="Kubernetes cluster schema" >}}

# Get ssl certificates from Let's Encrypt
To have proper registry opened to the web, we need to get ssl certificates.
For this we're going to obtain free ssl certificate from [Let's Encrypt](https://letsencrypt.org/).
In this post I'm going to use two domains: `example.com` for registry itself and `www.example.com` for UI.

Getting certificates from Let's Encrypt is described in official [getting started guide](https://letsencrypt.org/getting-started/)

After completing it we should have several `pem` files, but we need only two of them: `fullchain1.pem` and `privkey1.pem`.

# Deploy docker registry to Kubernetes
Deploying docker registry into Kubernetes is easy, mostly because Kubernetes provides examples in its own repository.
We are going to use this examples to deploy our registry.
In [kubernetes/cluster/addons/registry](https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/registry) we can find different ways of doing this:

* plain registry without tls or auth
* registry with lts
* registry with http auth protection

We're going to use registry with tls but with some changes.
We need to create secret with our certificates.

```bash
kubectl --namespace=kube-system create secret generic registry-tls-secret --from-file=domain.crt=fullchain1.pem --from-file=domain.key=privkey1.pem
```

Next step is to create replication controller. Since we want to have some frontend, we need to add another container to the POD - `registry-tls-rc.yaml`:

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: kube-registry-v0
  namespace: kube-system
  labels:
    k8s-app: kube-registry
    version: v0
spec:
  replicas: 1
  selector:
    k8s-app: kube-registry
    version: v0
  template:
    metadata:
      labels:
        k8s-app: kube-registry
        version: v0
    spec:
      containers:
      - name: registry
        image: registry:2
        resources:
          # keep request = limit to keep this container in guaranteed class
          limits:
            cpu: 100m
            memory: 100Mi
          requests:
            cpu: 100m
            memory: 100Mi
        env:
        - name: REGISTRY_HTTP_ADDR
          value: :5000
        - name: REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY
          value: /var/lib/registry
        - name: REGISTRY_HTTP_TLS_CERTIFICATE
          value: /certs/domain.crt
        - name: REGISTRY_HTTP_TLS_KEY
          value: /certs/domain.key
        volumeMounts:
        - name: image-store
          mountPath: /var/lib/registry
        - name: cert-dir
          mountPath: /certs
        ports:
        - containerPort: 5000
          name: registry
          protocol: TCP
      - name: registry-ui
        image: konradkleine/docker-registry-frontend:v2
        env:
        - name: ENV_DOCKER_REGISTRY_HOST
          value: "localhost"
        - name: ENV_DOCKER_REGISTRY_PORT
          value: "5000"
        - name: ENV_DOCKER_REGISTRY_USE_SSL
          value: "1"
        ports:
        - containerPort: 80
          name: registry
          protocol: TCP
      volumes:
      - name: image-store
        hostPath:
          path: /data/docker_registry
      - name: cert-dir
        secret:
          secretName: registry-tls-secret
```

We added registry-ui container to our POD with needed settings:

* `ENV_DOCKER_REGISTRY_HOST` can be `localhost` since we will run it on the same pod as registry
* `ENV_DOCKER_REGISTRY_USE_SSL` set to 1 to enable ssl
* `ENV_DOCKER_REGISTRY_PORT` actual port of registry

Also we mounted 2 volumes, one with tls secret and one to store actual data on the host.

And now we need to create Kubernetes Service description `registry-tls-svc.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kube-registry
  namespace: kube-system
  labels:
    k8s-app: kube-registry
    kubernetes.io/name: "KubeRegistry"
spec:
  selector:
    k8s-app: kube-registry
  type: NodePort
  ports:
  - name: registry
    port: 5000
    protocol: TCP
    nodePort: 30299
  - name: registry-ui
    port: 80
    protocol: TCP
    nodePort: 30949
```

We're going to use NodePort type of the service to expose the same ports on all nodes.
Configuring ingress controllers and proper loadbalancing is out of scope for this post.

Lets deploy what we have:

```bash
kubectl create -f registry-tls-svc.yaml
kubectl create -f registry-tls-rc.yaml
```

# Configure nginx
At this point we should have registry up and running.
I spent some time trying to configure nginx to work with ssl and docker registry.
And then I found awesome [post](http://container-solutions.com/running-secured-docker-registry-2-0/) describing it.

Since we want our registry to be accessible only for authorized users we need to create  `.htpasswd` file.
For this we can use `htpasswd` command, available in ubuntu (my external loadbalancer runs it) after installing `apache-utils`

```bash
apt-get install apache2-utils
mkdir /etc/nginx/registry
cd /etc/nginx/registry
htpasswd -c .htpasswd exampleuser
```

Also we need to put our ssl certificates to `/etc/nginx/ssl/dhub`.

After this we can create nginx config with 2 server blocks to serve traffic to our registry and UI

```
upstream registry {
    server NODE1-IP;
    server NODE2-IP;
    server NODE3-IP;
}
server {
    listen 443 ssl;

    server_name example.com;

    add_header Docker-Distribution-Api-Version registry/2.0 always;

    ssl on;
    ssl_certificate /etc/nginx/ssl/dhub/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/dhub/privkey.pem;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Original-URI $request_uri;
    proxy_set_header Docker-Distribution-Api-Version registry/2.0;

    # Allow large uploads
    client_max_body_size 600M;

    location / {
        auth_basic "Restricted";
        auth_basic_user_file /var/www/dhub/.htpasswd;
        proxy_pass https://registry:30299;
        proxy_read_timeout 900;
    }
    access_log  /var/log/nginx/docker.access.log;
    error_log   /var/log/nginx/docker.error.log;
}

server {
        listen 443 ssl;
        server_name www.example.com;

        ssl on;
        ssl_certificate /etc/nginx/ssl/dhub/fullchain.pem;
        ssl_certificate_key /etc/nginx/ssl/dhub/privkey.pem;
        location / {
            auth_basic "Restricted";
            auth_basic_user_file /var/www/dhub/.htpasswd;
            proxy_pass http://registry:30949;
        }
        access_log  /var/log/nginx/docker.access.log;
        error_log   /var/log/nginx/docker.error.log;
}
```

We can check nginx config for errors before applying changes with `nginx -t` and if everything is fine we need to reload it.

```
/etc/init.d/nginx reload
```

Thats it. Now you should be able to use your docker registry.

```
docker pull alpine
docker tag alpine:latest example.com/alpine

docker login -u exampleuser -p "YOURPASSWORD" -e your.email@gmail.com example.com
docker push example.com/alpine

```

Also you should be able to access your registry through frontend on https://www.example.com
