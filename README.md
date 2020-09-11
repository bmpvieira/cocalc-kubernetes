# Resurgo instructions
```bash
# Install gcloud on OSX
brew cask install gcloud
gcloud init
gcloud components install kubectl

# Set PATH if needed in fish shell
set -x PATH $PATH:/usr/local/Caskroom/google-cloud-sdk/latest/google-cloud-sdk/bin

# Go to Google Cloud and create a cluster named resurgo or use something like command below
gcloud container clusters create resurgo \
  --zone eu-west2-a \
  --node-locations eu-west2-a \
  --num-nodes 1 --enable-autoscaling --min-nodes 0 --max-nodes 6

# Get resurgo cluster credentials for kubectl tool
gcloud container clusters get-credentials resurgo

# Create service account
kubectl create serviceaccount cocalc-kubernetes-server
kubectl create rolebinding cocalc-kubernetes-server-binding --clusterrole=admin --serviceaccount=default:cocalc-kubernetes-server

# Git clone cocalc-kubernetes
git clone git@gitlab.com:resurgo/cocalc-kubernetes.git
cd cocalc-kubernetes/server

# Deploy code to cluster
kubectl apply -f deployment.yaml

# Expose service privately and local port forward
kubectl expose deployment cocalc-kubernetes-server
kubectl port-forward --address 0.0.0.0 service/cocalc-kubernetes-server 4043:443

# Or, expose service to the world (TODO: improve security)
kubectl expose deployment cocalc-kubernetes-server --type=LoadBalancer --name=cocalc

# Deploy NFS for shared state across instances
kubectl apply -f nfs-service.yaml

# Get some info about service (like availability, IP, ports, status, etc)
kubectl get nodes --output wide
watch kubectl get pods
kubectl get services
```

# cocalc-kubernetes

This is a free open source AGPL licensed slightly modified version of [cocalc-docker](https://github.com/sagemathinc/cocalc-docker), but for running CoCalc on a Kubernetes cluster.  There is one pod for the server and one pod for each project.

## STATUS

I've updated the images on DockerHub that this uses to the latest version of CoCalc as of **May 12, 2020**.

## Installation

See [server/README.md](./server/README.md) to get going!

## LICENSE AND SUPPORT

  - Much of this code is licensed [under the AGPL](https://en.wikipedia.org/wiki/Affero_General_Public_License). If you would instead like a business-friendly MIT license, please contact [help@cocalc.com](mailto:help@cocalc.com), and we will sell you a 1-year license for $1499.  This also includes some support, though with no guarantees (that costs more).
  - Join the [CoCalc Docker mailing list](https://groups.google.com/a/sagemath.com/group/cocalc-docker) for news, updates and more.
  - Use the [CoCalc mailing list](https://groups.google.com/forum/?fromgroups#!forum/cocalc) for general community support.

## SECURITY STATUS

  - If you setup everything as explained in [server/README.md](./server/README.md), including appropriate network restrictions on the project pods, then there are no known security vulnerabilities.  In particular, this is much safer to run than [cocalc-docker](https://github.com/sagemathinc/cocalc-docker), if you are going to expose this to untrusted (or uncareful) users.

## Discussion

cocalc-kubernetes is an open source (but AGPL) version of CoCalc that can be run on an existing generic Kubernetes cluster.

It is as similar as possible to [cocalc-docker](https://github.com/sagemathinc/cocalc-docker), with the following changes:
  1. It runs in a Kubernetes cluster rather than a plain Docker install, and
  2. Projects run as separate pods on the cluster rather than all in the same Docker container.

The benefits of this architecture include:
  - Projects can have individual resource limitations imposed by Kubernetes.
  - The security issues with cocalc-docker (involving projects connecting to other project services on localhost) can be addressed by blocking outgoing network connections from projects.
  - Projects run across a cluster, so the number of projects that one can run at once is a function of the number (and size) of nodes in the cluster, rather than cocalc-docker host.
  - This can be done with just a slight modification and addition to the entirely open source (AGPL) codebase that cocalc-docker uses.
  
The drawbacks of this architecture over a more complicated architecture like the closed-source KuCalc (what https://cocalc.com uses) include:
  - It is unclear to what extent it can handle a large number of *simultaneous users*, since the entire server component (the database, hub, NFS file server, etc.) are all served from a single pod.
  - Project storage is just a single NFS server, so disk iops may be lower for client projects, which may or may not be an issue depending on the network and use of projects.
  - Filesystem level snapshots and other backups have to be handled outside CoCalc.  This is the responsibility of the admin and is not part of cocalc-kubernetes at present.  However, the [TimeTravel](https://doc.cocalc.com/time-travel.html) functionality, which records every version of a file or notebook while you work on it, and lets you browse all past versions, does **fully work** in cocalc-kubernetes.
  - The project image is much more minimal than the one provided by https://cocalc.com -- it has to be small enough to reasonably run in a normal way without pull taking too long.  The image in KuCalc is hundreds of gigabytes but mounts quickly using some sophisticated tricks.
