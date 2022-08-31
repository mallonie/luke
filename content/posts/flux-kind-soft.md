---
title: Flux with Kind and Soft Serve
description: Setup a local Kubernetes Cluster with Flux and SoftServe
date: 2022-02-28
tags:
- flux
- fluxcd.io
- cd
- continuous delivery
- charm.sh
- soft-serve
- kubernetes
- k8s
- docker
---

> Originally post to [hackmd.io](https://hackmd.io/@nalum/flux-with-kind-and-soft-serve)

This article covers getting a local [Kind](https://kind.sigs.k8s.io "Kind Website") cluster
setup with [Flux](https://fluxcd.io "Flux Website") and the self hosted git server
[Soft Serve](https://github.com/charmbracelet/soft-serve "Soft Serve GitHub").

## Requirements

You will need to have the following installed:

- `docker` [installation](https://docs.docker.com/get-docker/)
- `kubectl` [installation](https://kubernetes.io/docs/tasks/tools/#kubectl)
- `kustomize` [installation](https://kubectl.docs.kubernetes.io/installation/kustomize/)
- `flux` [installation](https://fluxcd.io/docs/installation/ "Install Flux")
- `kind` [installation](https://kind.sigs.k8s.io/#installation-and-usage "Install Kind")
- `soft` [installation](https://github.com/charmbracelet/soft-serve#installation "Install Soft Serve")

### Assumptions

I am assuming the following things:

- Running on Linux
- Familiar with:
  - Docker
  - YAML
  - Git
  - Kind
  - kubectl

## Let's get Started

We need to run two copies of Soft Serve in order to be able to access and configure
the software.

### Docker Run Soft Serve

We're going to start a local copy of Soft Serve in preparation for running a copy
within the Kind cluster. Let's assume that your Flux repository is located in `/home/nalum/code/flux`.
You will want to setup a new environment variable: `export PATH_TO_CODE="/home/nalum/code"`.
This will be used to mount your local code to both the local copy and the copy running
in the Kind cluster.

Let's get a running instance of Soft Serve going with Docker:

```bash
❯ docker run -d \
  --name=soft-serve \
  --volume=${PATH_TO_CODE}:/soft-serve \
  --publish=23231:23231 \
  --restart=unless-stopped \
  --user=$(id -u):$(id -g) \
  --add-host=host.docker.internal:host-gateway \
  charmcli/soft-serve:latest
```

This will start the container and then create a few directories
that are used by Soft Serve as follows:

```bash
❯ cd ${PATH_TO_CODE}
❯ tree -d -L 1
.
├── flux
├── repos # Created by Soft Serve
└── ssh   # Created by Soft Serve
```

You should also be able to ssh to the Docker Soft Serve:

```bash
❯ ssh localhost -p 23231
```

And you should be presented with a TUI looking something like this:

![Soft Serve SSH TUI showing Home and brief message to clone the configuration repository](/images/ss-home.png "Soft Serve SSH TUI showing Home and brief message to clone the configuration repository")

At this point we have the basics of what we need. You will want to clone the configuration
repository from Soft Serve:

```bash
❯ git clone ssh://localhost:23231/config
Cloning into 'config'...
remote: Enumerating objects: 38, done.
remote: Counting objects: 100% (38/38), done.
remote: Compressing objects: 100% (38/38), done.
remote: Total 38 (delta 11), reused 0 (delta 0), pack-reused 0
Receiving objects: 100% (38/38), 11.98 KiB | 5.99 MiB/s, done.
Resolving deltas: 100% (11/11), done.
```

You can now configure (we'll get to this later) Soft Serve to be ready for when it's
deployed on the Kind cluster. The configuration is done with YAML and the following
is the starting config created by running the software:

```yaml=
# The name of the server to show in the TUI.
name: Soft Serve

# The host and port to display in the TUI. You may want to change this if your
# server is accessible from a different host and/or port that what it's
# actually listening on (for example, if it's behind a reverse proxy).
host: localhost
port: 23231

# Access level for anonymous users. Options are: read-write, read-only and
# no-access.
anon-access: read-write

# You can grant read-only access to users without private keys. Any password
# will be accepted.
allow-keyless: false

# Customize repo display in the menu. Only repos in this list will appear in
# the TUI.
repos:
- name: Home
  repo: config
  private: true
  note: "Configuration and content repo for this server"

# users:
#   - name: Admin
#     admin: true
#     public-keys:
#       - KEY TEXT
#   - name: Example User
#     collab-repos:
#       - REPO
#     public-keys:
#       - KEY TEXT
```

### Setting Up Kind

For the Kind cluster we need to mount the `$PATH_TO_CODE` onto one or all of the
nodes running in the cluster. The following config will setup a cluster one node
and the path mounted for use by containers deployed on that node:

```yaml=
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: dev
nodes:
- role: control-plane
  extraMounts:
  - hostPath: /home/nalum/code # Same value that is in $PATH_TO_CODE
    containerPath: /soft-serve
    readOnly: false
    selinuxRelabel: false
    propagation: None
```

Get your cluster up by running the following (this will change your `kubectl` context):

```bash
❯ kind create cluster --config dev-config.yaml
Creating cluster "dev" ...
 ✓ Ensuring node image (kindest/node:v1.21.1) 🖼
 ✓ Preparing nodes 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
Set kubectl context to "kind-dev"
You can now use your cluster with:

kubectl cluster-info --context kind-dev

Have a question, bug, or feature request? Let us know! https://kind.sigs.k8s.io/#community 🙂
```

### Kind Run Soft Serve

Now that we've got the a default setup of Soft Serve and the Kind cluster running
it's time to run Soft Serve on the Kind cluster. We will define a `Namespace`,
a `Service` and a `Deployment` to get this running on the Kind cluster.

The `Namespace`:

```yaml=
apiversion: v1
kind: namespace
metadata:
  name: soft-serve
  labels:
    app: soft-serve
```

The `Service`:

```yaml=
apiversion: v1
kind: service
metadata:
  name: git
  namespace: soft-serve
  labels:
    app: soft-serve
spec:
  type: clusterip
  selector:
    app: soft-serve
  ports:
  - name: ssh
    port: 23231
    targetPort: 23231
    protocol: TCP
```

The `Deployment`:

```yaml=
apiVersion: apps/v1
kind: Deployment
metadata:
  name: soft-serve
  namespace: soft-serve
  labels:
    app: soft-serve
spec:
  replicas: 2
  revisionHistoryLimit: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: soft-serve
  template:
    metadata:
      labels:
        app: soft-serve
    spec:
      containers:
      - image: charmcli/soft-serve:latest
        name: soft-serve
        ports:
        - containerPort: 23231
          name: ssh
          protocol: TCP
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /soft-serve
          name: repos
          readOnly: true
      securityContext:
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
      volumes:
      - name: repos
        hostPath:
          path: /soft-serve
          type: Directory
```

In the `Deployment` we are setting some specific security context information to
make it so the container is running as the user who owns the directory that has been
mounted into the Kind node and which we are mounting into the `Pod`.

When you install a Linux distribution and create the main user for that install it
will likely have the id `1000` for the User and the Group, you can double check this
by running the following commands in your terminal:

```bash
❯ id # Prints the User ID, Group ID and IDs of other groups the user is part of
uid=1000(nalum) gid=1000(nalum) groups=1000(nalum),3(sys),90(network),98(power),962(realtime),964(docker),991(lp),995(audio),998(wheel)
❯ id -u # Only the User ID
1000
❯ id -g # Only the Group ID (this is the main group for this user)
1000
```

This was used earlier when we started Soft Serve with the `docker` command to set
the `user:group` for the container to run as.

Let's apply these resources to the Kind cluster:

```bash
❯ kubectl apply -f soft-serve.yaml
namespace/soft-serve created
service/git created
deployment.apps/soft-serve created
```

Now if you run the command to get the `Pods` in the created `Namespace` you should
see something like the following:

```bash
❯ kubectl -n soft-serve get pods
NAME                          READY   STATUS    RESTARTS   AGE
soft-serve-786fb9c9f8-2h4pl   1/1     Running   0          12s
soft-serve-786fb9c9f8-vzxgw   1/1     Running   0          12s
```

### Configure Soft Serve

Configuring Soft Serve is pretty simple, go into the config repo we cloned
earlier and edit the `config.yaml` file (cut down file for brevity):

```yaml=
...
host: git.soft-serve
...
repos:
...
- name: Flux
  repo: flux
  private: false
  note: "Flux Kind Cluster Demo running Soft Serve local git server"
...
```

We've changed the host name and added a new repository to the config.
To have this repository appear correctly and be used by Flux within the
`Kind` cluster we need to create a symlink from `${PATH_TO_CODE}/flux` in
`${PATH_TO_CODE}/repos`:

```bash
❯ cd ${PATH_TO_CODE}/repos
❯ ls -lah
total 12K
drwxr-xr-x  3 nalum nalum 4.0K Feb 24 09:18 .
drwxr-xr-x 13 nalum nalum 4.0K Feb  8 13:05 ..
drwxr-xr-x  5 nalum nalum 4.0K Feb  8 13:12 config
❯ ln -s ../flux ./flux
❯ ls -lah
total 12K
drwxr-xr-x  3 nalum nalum 4.0K Feb 24 09:18 .
drwxr-xr-x 13 nalum nalum 4.0K Feb  8 13:05 ..
drwxr-xr-x  5 nalum nalum 4.0K Feb  8 13:12 config
lrwxrwxrwx  1 nalum nalum    7 Feb 24 09:18 flux -> ../flux
```

If you have the repo setup already with a README that should display when
you view it in the SSH TUI, if you don't have a repo setup let's do a quick
setup of git and add a README file to the flux repo, something simple e.g.:

```bash
❯ cd flux
❯ git init
Initialized empty Git repository in /home/nalum/code/flux/.git/

❯ tee README.md <<EOT
heredoc> # Flux Kind Cluster
heredoc> 
heredoc> Flux GitRepository Source
heredoc> EOT
# Flux Kind Cluster

Flux GitRepository Source
❯ cat README.md
# Flux Kind Cluster

Flux GitRepository Source
❯ git add README.md
❯ git commit -m "Adding README"
[main (root-commit) b83b100] Adding README
 1 file changed, 3 insertions(+)
 create mode 100644 README.md
```

In order to see this change in the SSH TUI you will need to restart the docker
container e.g. `docker restart soft-serve`. Now when you ssh into Soft Serve
running on your local docker install you will see something like the following:

![SSH TUI View of the Flux Repo README we just added to the repo](/images/ss-flux.png "SSH TUI View of the Flux Repo README we just added to the repo")

The same can be done with the instance running on the `Kind` Cluster. Restart the
running `Pods` e.g. `kubectl --context kind-dev -n soft-serve delete pods -l app=soft-serve`.
Once the `Pods` have been recreated you can port forward to one of the running `Pods`
e.g. `kubectl --context kind-dev -n soft-serve port-forward soft-serve-786fb9c9f8-25wkc 23232:23231`
In the above command I've port forward the `Pods` port `23231` to the local port
`23232`. So now we can ssh into the TUI on the `Pod` and see the same view as above.

In the normal day to day use with Flux you do not need to stop and start the Soft
Serve instances unless you want to use the TUI and see the README file updates
as the TUI only appears to update when there is a push to the repo.

### Setting up Flux

In the `flux` repo setup a structure like the following:

```bash
❯ tree -a -I .git
.
├── clusters
│  └── dev
│      └── flux-system
└── README.md
```

We are going to export the Flux installation into the `flux-system` directory, you
can do this as follows: `flux install --export > clusters/dev/flux-system/gotk-components.yaml`

I won't include the content of that file as there is a lot in it, but feel free to
browse. Next we'll want to create a `gotk-sync.yaml` and a `kustomization.yaml`.
Which you can do as follows:

```bash
❯ flux create source git flux-system \
  --git-implementation=libgit2 \
  --url=ssh://git@git.soft-serve:23231/flux.git \
  --branch=main \
  --interval=1m \
  --export > clusters/dev/flux-system/gotk-sync.yaml
❯ flux create kustomization flux-system \
  --source=GitRepository/flux-system \
  --path=./clusters/dev \
  --prune=true \
  --interval=1m \
  --export >> clusters/dev/flux-system/gotk-sync.yaml
❯ cd clusters/dev/flux-system
❯ kustomize create --autodetect
❯ cd ../../..
❯ tree -a -I .git -I vendor
.
├── clusters
│   └── dev
│       └── flux-system
│           ├── gotk-components.yaml
│           ├── gotk-sync.yaml
│           └── kustomization.yaml
└── README.md
```

It is also worth adding the resource above that created the Soft Serve deployment
we are using and setting up the `kustomization.yaml` for that directory:

```bash
❯ tree -a -I .git
.
├── clusters
│   └── dev
│       ├── flux-system
│       │   ├── gotk-components.yaml
│       │   ├── gotk-sync.yaml
│       │   └── kustomization.yaml
│       └── soft-serve
│           ├── deployment.yaml
│           ├── kustomization.yaml
│           ├── namespace.yaml
│           └── service.yaml
└── README.md
```

Now let's apply the above `flux-system` resources to the Kind cluster:

```bash
❯ kubectl apply -f clusters/dev/flux-system/gotk-components.yaml
namespace/flux-system created
customresourcedefinition.apiextensions.k8s.io/alerts.notification.toolkit.fluxcd.io created
customresourcedefinition.apiextensions.k8s.io/buckets.source.toolkit.fluxcd.io created
customresourcedefinition.apiextensions.k8s.io/gitrepositories.source.toolkit.fluxcd.io created
customresourcedefinition.apiextensions.k8s.io/helmcharts.source.toolkit.fluxcd.io created
customresourcedefinition.apiextensions.k8s.io/helmreleases.helm.toolkit.fluxcd.io created
customresourcedefinition.apiextensions.k8s.io/helmrepositories.source.toolkit.fluxcd.io created
customresourcedefinition.apiextensions.k8s.io/kustomizations.kustomize.toolkit.fluxcd.io created
customresourcedefinition.apiextensions.k8s.io/providers.notification.toolkit.fluxcd.io created
customresourcedefinition.apiextensions.k8s.io/receivers.notification.toolkit.fluxcd.io created
serviceaccount/helm-controller created
serviceaccount/kustomize-controller created
serviceaccount/notification-controller created
serviceaccount/source-controller created
clusterrole.rbac.authorization.k8s.io/crd-controller-flux-system created
clusterrolebinding.rbac.authorization.k8s.io/cluster-reconciler-flux-system created
clusterrolebinding.rbac.authorization.k8s.io/crd-controller-flux-system created
service/notification-controller created
service/source-controller created
service/webhook-receiver created
deployment.apps/helm-controller created
deployment.apps/kustomize-controller created
deployment.apps/notification-controller created
deployment.apps/source-controller created
networkpolicy.networking.k8s.io/allow-egress created
networkpolicy.networking.k8s.io/allow-scraping created
networkpolicy.networking.k8s.io/allow-webhooks created
```

Before we apply the sync file we need to setup a `known_hosts` file and an SSH Key
that Flux can work with in order to pull from our Soft Serve repo. Let's start
with the SSH Key:

```bash
❯ ssh-keygen -t ed25519 -C "flux-system"
Generating public/private ed25519 key pair.
Enter file in which to save the key (/home/nalum/.ssh/id_ed25519): /home/nalum/code/ssh/identity
...
+----[SHA256]-----+
```

To create the `known_hosts` file we need to get the public key of the Soft Serve
server we're running, we can do that with:

```bash
❯ cat ${PATH_TO_CODE}/ssh/soft_serve_server_ed25519.pub
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIMDYacTjeLH4ha6BDgWN3rXeh70GDYmaSVBw5UQg1FQ3 root@2dcd99be9aef
```

Copy this public key into a new `${PATH_TO_CODE}/ssh/known_hosts` file and put
the domain name in front of it something like the following:

```=
git.soft-serve:23231 ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIMDYacTjeLH4ha6BDgWN3rXeh70GDYmaSVBw5UQg1FQ3 root@2dcd99be9aef
```

Now that we have the SSH Key and the `known_hosts` file we need to create a secret
that Flux will pick up, you can use `-o yaml` to see the output:

```bash
❯ kubectl -n flux-system create secret generic \
  --from-file=identity=${PATH_TO_CODE}/ssh/identity \
  --from-file=identity.pub=${PATH_TO_CODE}/ssh/identity.pub \
  --from-file=known_hosts=${PATH_TO_CODE}/ssh/known_hosts \
  flux-system
secret/flux-system created
```

With that put in place we can then apply the sync file:

```bash
❯ kubectl apply -f clusters/dev/flux-system/gotk-sync.yaml
gitrepository.source.toolkit.fluxcd.io/flux-system created
kustomization.kustomize.toolkit.fluxcd.io/flux-system created
```

If you now run `flux get all` you should see something like the following:

```bash
❯ flux get all
NAME                            READY   MESSAGE                         REVISION        SUSPENDED 
gitrepository/flux-system       True    Fetched revision: main/7e12f0c  main/7e12f0c    False    

NAME                            READY   MESSAGE                                                                                                 REVISION        SUSPENDED 
kustomization/flux-system       False   kustomization path not found: stat /tmp/flux-system2367946620/clusters/dev: no such file or directory                   False
```

The `Kustomization` is not working here because we have not committed any of the
files to the repo yet. If you run `watch flux get all --all-namespaces` and the add
and commit the `clusters` folder and it's contents, you will see the `Kustomization`
reconcile itself:

```bash
❯ flux get all --all-namespaces
NAMESPACE       NAME                            READY   MESSAGE                         REVISION        SUSPENDED 
flux-system     gitrepository/flux-system       True    Fetched revision: main/b6e77db  main/b6e77db    False    

NAMESPACE       NAME                            READY   MESSAGE                         REVISION        SUSPENDED 
flux-system     kustomization/flux-system       True    Applied revision: main/b6e77db  main/b6e77db    False
```

If you describe the kustomization object with kubectl you'll see the resources that
have been applied to the system through it:

```bash
❯ kubectl -n flux-system describe kustomizations.kustomize.toolkit.fluxcd.io flux-system  
Name:         flux-system
Namespace:    flux-system
Labels:       kustomize.toolkit.fluxcd.io/name=flux-system
              kustomize.toolkit.fluxcd.io/namespace=flux-system
Annotations:  <none>
API Version:  kustomize.toolkit.fluxcd.io/v1beta2
Kind:         Kustomization
Metadata:
  ...
Events:
  Type     Reason  Age    From                  Message
  ----     ------  ----   ----                  -------
  Warning  error   6m35s  kustomize-controller  kustomization path not found: stat /tmp/flux-system1699096295/clusters/dev: no such file or directory
  ...
  Normal   info    115s   kustomize-controller  CustomResourceDefinition/alerts.notification.toolkit.fluxcd.io configured
CustomResourceDefinition/buckets.source.toolkit.fluxcd.io configured
...
Namespace/flux-system configured
Namespace/soft-serve configured
ServiceAccount/flux-system/helm-controller configured
ServiceAccount/flux-system/kustomize-controller configured
...
Deployment/flux-system/source-controller configured
Deployment/soft-serve/soft-serve configured
...
  Normal  info  34s   kustomize-controller  Reconciliation finished in 680.646597ms, next run in 1m0s
```

## Final Thoughts

This is a lot of work in order to use Flux from a local repository. This was a fun
exercise to get going and document here. It could be automated and be a single command
to setup the whole system, which _maybe_ is worth it. I'll leave that for you to
decide.

Please leave a comment with your thoughts or if you have any issues following along
I'd be happy to try helping out.

