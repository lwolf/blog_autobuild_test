+++
author = "Sergey Nuzhdin"
categories = ["glusterfs", "storage", "kubernetes", "docker", "infrastructure", "k8s"]
date = "2017-02-09"
description = ""
featured = "glusterkube.png"
featuredalt = ""
featuredpath = "date"
linktitle = ""
title = "Two days of pain or how I deployed GlusterFS cluster to Kubernetes"

+++


I spent last two days installing GlusterFS storage on top of my Kubernetes.
It took much more time and effort than it should.
I faced all kinds of problems, some if which were not obvious and took a lot of googling.
So I decided to write this post. Hopefully it will save some time for somebody.


I was playing with [helm](https://github.com/kubernetes/helm). Trying to assemble a complex application with several dependencies from official chart repository.
I tried to install PostgreSQL chart with persistence enabled, but it didn't work.
After inspecting manifests it became clear that it needs dynamic storage provision.
Since I'm running bare-metal cluster it has no proper storage solution. Everything is running NFS.
So I cloned this chart and changed it to use NFS. Not the best solution but it worked.

But then I tried to install Minio and faced the same problem.
But in this case it it was impossible to do the same hack because of StatefullSet.


## Time for a proper distributed storage

I read about dynamic storage provisioning and new StorageClass entity in Kubernetes. But since I had only NFS storages I didn’t try it.

After some googling, I  had two choices for my storage:  [GlusterFS](https://www.gluster.org/) and [Ceph](http://ceph.com/).

They were both OK for me until I found [`heketi`](https://github.com/heketi/heketi) .


> RESTful based volume management framework for GlusterFS
>
> Heketi provides a RESTful management interface which can be used to manage the life cycle of GlusterFS volumes. With Heketi, cloud services like OpenStack Manila, Kubernetes, and OpenShift can dynamically provision GlusterFS volumes with any of the supported durability types. Heketi will automatically determine the location for bricks across the cluster, making sure to place bricks and its replicas across different failure domains. Heketi also supports any number of GlusterFS clusters, allowing cloud services to provide network file storage without being limited to a single GlusterFS cluster.

So I decided to go with GlusterFS.
Heketi even has the [guide on Kubernetes integration](https://github.com/heketi/heketi/wiki/Kubernetes-Integration).
I will go through the guide here with all the problems and solutions/hacks I had to do to make it work.

Following this guide I installed heketi-cli and started to follow the steps:


    > $ git clone https://github.com/heketi/heketi
    > $ cd heketi/extras/kubernetes
    # create glusterfs daemonset
    > $ kubectl create -f glusterfs-daemonset.json

    # mark nodes which will be used as glusterfs
    > $ kubectl label node server-node-01 storagenode=glusterfs
    > $ kubectl label node server-node-02 storagenode=glusterfs

    > $ kubectl create -f deploy-heketi-deployment.json
    service "deploy-heketi" created
    deployment "deploy-heketi" created

    > $ kubectl get pods
    NAME                                          READY     STATUS    RESTARTS   AGE
    deploy-heketi-3195612485-s87qd                1/1       Running   0          1m
    glusterfs-04dk7                               1/1       Running   0          1m
    glusterfs-l23jq                               1/1       Running   0          1m

At this point, we have 2 GlusterFS pods and heketi deployer running.


    > $ kubectl port-forward deploy-heketi-3195612485-s87qd :8080
    Forwarding from 127.0.0.1:54969 -> 8080
    Forwarding from [::1]:54969 -> 8080


    > curl http://localhost:54969/hello
    Handling connection for 54969
    Hello from Heketi

    > $ export HEKETI_CLI_SERVER=http://localhost:54969

## Creating topology

Next step in the manual was to create GlusterFS topology.
Topology is JSON manifest with the list of all nodes, disks, and clusters used by GlusterFS.
More information about topology is in [documentation](https://github.com/heketi/heketi/wiki/Setting-up-the-topology).
Sample topology is in repository we cloned:  [heketi/client/cli/go/topology-sample.json](https://github.com/heketi/heketi/blob/master/client/cli/go/topology-sample.json)

Here is a cut of my topology file.


    {
        "clusters": [
            {
                "nodes": [
                    {
                        "node": {
                            "hostnames": {
                                "manage": [
                                    "server-node-01" // <- IMPORTANT! Should be name
                                ],
                                "storage": [
                                    "192.168.11.11" // <- IMPORTANT! Should be IP
                                ]
                            },
                            "zone": 1
                        },
                        "devices": [
                            "/dev/sdb"
                        ]
                    },
                    ...
                ]
            }
        ]
    }

We need to create topology from the file. Make sure you edited it and set your nodes and storages .  To create it run:


    > $ heketi-cli topology load --json=topology-sample.json


______

### Issue: hostnames

After the first run, I’ve got this error.


    > $ heketi-cli topology load --json=topology-sample.json
    Creating cluster ... ID: 453f4948ea32888b7925b7a0ae144fdc
         Creating node 192.168.11.11 ... ID: d666452f025e805c6ff0644cc1363fd8
              Adding device /dev/sdb ... Unable to add device: Unable to find a GlusterFS pod on host 192.168.11.11 with a label key glusterfs-node
         Found node 192.168.11.12 on cluster b4b5e2fe39273e9250cc307794d28bd1
              Adding device /dev/sdb ... Unable to add device: Unable to find a GlusterFS pod on host 192.168.11.12 with a label key glusterfs-node

This actually was my fault. I missed the NOTE in the documentation and put IP address instead of the hostname in `hostnames.manage`  field.

**Workaround/Solution**: read manuals carefully :)


> NOTE: Make sure that `hostnames/manage` points to the exact name as shown under `kubectl get nodes`, and `hostnames/storage` is the IP address of the storage network.


______

### Issue: Adding device hangs…

Next time I run the command it hanged. I tried to restart the command and all pods. I tried to change the order of nodes in topology file hoping that it was some buggy node. Nothing. I tried waiting 10-15 minutes to get timeout and error as was suggested in some thread. Nothing.


    > $ heketi-cli topology load --json=topology-sample.json
    Creating cluster ... ID: 763d9c28bb1cbfbda46c9888f0398936
         Creating node server-node-02 ... ID: cc0b10651a082a324395ffa12818744e
              Adding device /dev/sdb ... ^C


Finally, after some test and trial, I found a workaround. It’s definitely not a solution, but at least it worked.

**Workaround/Solution**:  attach to all GlusterFS pods and run `pvcreate` manually.


    > $ kubectl get pods  | grep glusterfs
    glusterfs-631p3                             1/1       Running   0          2m
    glusterfs-fdxgr                             1/1       Running   0          2m

    > kubectl exec -it glusterfs-631p3 /bin/bash
    [root@server-node-02 /]# ps lax
    ...
    4373 ... pvcreate --metadatasize=128M --dataalignment=256K /dev/sdb
    ...
    [root@server-node-02 /]# kill 4373
    [root@server-node-02 /]# pvcreate --metadatasize=128M --dataalignment=256K /dev/sdb
      Physical volume "/dev/sdb" successfully created.

After I got it to work I tried to delete everything and run from scratch, but it hanged the same way. And again running pvcreate manually fixed the issue.

Anyway, after this   `heketi-cli topology load --json=topology-sample.json` successfully created my topology.

## Create heketi data volume

Next step was to run  `heketi-cli setup-openshift-heketi-storage` . Which should  provision a volume for heketi’s database. The name of the command is a bit confusing. I spent some time searching to make sure that I need to run on non-openshift platform. But it should be run.

As you may guess, it also didn’t go smooth.

______
### Issue: No space error

    > $ heketi-cli setup-openshift-heketi-storage
    Error: No space

From this error response, it should be “obvious” that you have too few nodes right?!  :)

Apparently, this command has default replication factor of 3 and it cannot be changed.

**Workaround/Solution**: add 3rd node to cluster.

______
### Issue: modprobe failed

After I added the third node and run the command again I’ve got this:


    > $ heketi-cli setup-openshift-heketi-storage
    Error: Unable to execute command on glusterfs-cjn49:   /usr/sbin/modprobe failed: 1
      thin: Required device-mapper target(s) not detected in your kernel.
      Run `lvcreate --help' for more information.

Quick googling lead me to this [issue](https://github.com/gluster/gluster-kubernetes/issues/19).

**Workaround/Solution**: To fix this we need to run  `modprobe dm_thin_pool` on all nodes.

______
### Issue: Host not connected

I had one more issue, but it was due to a misconfiguration in my DNS server.


    > $ heketi-cli setup-openshift-heketi-storage
    Error: Unable to execute command on glusterfs-szljx: volume create: heketidbstorage: failed: Staging failed on server-node-01. Error: Host server-node-03 not connected

**Workaround/Solution**: Make sure that all GlusterFS pods can resolve and ping each other.

## Deploying heketi storage

The previous command produced a file called `heketi-storage.json` . Now we need to deploy it to Kubernetes and wait until the job is finished.


    > $ kubectl create -f heketi-storage.json

After several minutes I noticed that container stuck in `ContainerCreating` state.


    > $ kubectl get pods
    NAME                                        READY     STATUS
    heketi-storage-copy-job-j9x09               0/1       ContainerCreating


In pod description, we can see that it can’t mount glusterfs filesystem.


    > $ kubectl describe pod heketi-storage-copy-job-j9x09
    Name:                heketi-storage-copy-job-j9x09
    ...
    Events:
      FirstSeen     LastSeen     Count     From                    SubObjectPath     Type          Reason          Message
      ---------     --------     -----     ----                    -------------     --------     ------          -------
      12s          12s          1     {default-scheduler }                    Normal          Scheduled     Successfully assigned heketi-storage-copy-job-j9x09 to server-node-02
      10s          2s          5     {kubelet server-node-02}               Warning          FailedMount     MountVolume.SetUp failed for volume "kubernetes.io/glusterfs/3f3dd514-edfe-11e6-80fa-000c29c84f22-heketi-storage" (spec.Name: "heketi-storage") pod "3f3dd514-edfe-11e6-80fa-000c29c84f22" (UID: "3f3dd514-edfe-11e6-80fa-000c29c84f22") with: glusterfs: mount failed: mount failed: exit status 32
    Mounting command: mount
    Mounting arguments: 192.168.11.12:heketidbstorage /var/lib/kubelet/pods/3f3dd514-edfe-11e6-80fa-000c29c84f22/volumes/kubernetes.io~glusterfs/heketi-storage glusterfs [log-level=ERROR log-file=/var/lib/kubelet/plugins/kubernetes.io/glusterfs/heketi-storage/heketi-storage-copy-job-j9x09-glusterfs.log]
    Output: mount: unknown filesystem type 'glusterfs'

    the following error information was pulled from the glusterfs log to help diagnose this issue: glusterfs: could not open log file for pod: heketi-storage-copy-job-j9x09


This error was fixed by installing `glusterfs-client`  on all nodes and restarting the job.

**Workaround/Solution**: apt-get install glusterfs-client

After the job is completed we need to delete everything used for bootstrap and deploy actual heketi.


    # wait for job status "Completed"
    > $ kubectl get pods
    NAME                                        READY     STATUS
    heketi-storage-copy-job-j9x09               0/1      Completed
    ...

    # delete all entities used for bootstrap
    > $ kubectl delete all,service,jobs,deployment,secret --selector="deploy-heketi"
    service "deploy-heketi" deleted
    job "heketi-storage-copy-job" deleted
    deployment "deploy-heketi" deleted
    secret "heketi-storage-secret" deleted

    # deploy heketi
    > $ kubectl create -f heketi-deployment.json
    service "heketi" created
    deployment "heketi" created


# Create StorageClass

To use our new GlusterFS cluster for dynamic provisioning we need to create StorageClass.
More details about StorageClass entity could be found [here](https://kubernetes.io/docs/user-guide/persistent-volumes/#storageclasses) and [here](http://blog.kubernetes.io/2016/10/dynamic-provisioning-and-storage-in-kubernetes.html)


    # storage-class.yml
    apiVersion: storage.k8s.io/v1beta1
    kind: StorageClass
    metadata:
      name: slow
    provisioner: kubernetes.io/glusterfs
    parameters:
      resturl: "http://IP-ADDRESS-OF-HEKETI-SERVICE:8080"

    > $ kubectl create -f storage-class.yml

# Create dynamic PersistentVolumeClaim

Finally, we can create PVC to test that everything works.


    # test-pvc.yml
    kind: PersistentVolumeClaim
    apiVersion: v1
    metadata:
      name: myclaim
      annotations:
        volume.beta.kubernetes.io/storage-class: "slow"
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 10Gi




    > $ kubectl create -f test-pvc.yml
    persistentvolumeclaim "myclaim" created

    > $ kubectl get pvc
    NAME      STATUS    VOLUME            CAPACITY    ACCESSMODES   AGE
    myclaim   Bound     pvc-9dc2cf51-..   10Gi        RWO           1m


# Conclusion

After two days of struggle, I finally got it working.
It was definitely worth it.

I already tried to use it to create dynamic volumes for different helm charts and it works pretty well.
Now I can forget about manual creating of PersistentVolumes.

With this features bare-metal cluster became closer to cloud-based ones, at least storage-wise.

