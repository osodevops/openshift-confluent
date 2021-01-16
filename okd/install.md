# Installing OpenShift/OKD

[[_TOC_]]

## 1. Preparation and Prerequisites

In order to install an OpenShift (hereinafter referred to as `OKD`) cluster on AWS, we first need a subdomain, which is also be the name of the cluster itself; OKD's architecture is such that DNS is bound tightly to core aspects of the setup, which is challenging in on-prem scenarios.

However, for AWS, it's plain-sailing.

### 1.1 Obtaining OKD Installer

Run, don't walk, to the [release status website](https://origin-release.apps.ci.l2s4.p1.openshiftapps.com/) for OKD. Once there, select the latest (should be top of list) of the `4-stable` branch.

As of Jan 2021, this happens to be: `4.6.0-0.okd-2020-12-12-135354`.

A new page will open with instructions on how to obtain the installer artefacts, inviting you to execute the followig:

```
oc adm release extract --tools quay.io/openshift/okd@sha256:01948f4c6bdd85cdd212eb40d96527a53d6382c4489d7da57522864178620a2c
```

This command appears to hang, but eventually dumps you back to the shell with the following files in your `$CWD`:

| **FILE** | **DESCRIPTION** |
|----------|-----------------|
| `openshift-client-linux-4.6.0-0.okd-2020-12-12-135354.tar.gz` | Contains `oc` and `kubectl`. The binary `oc` is OKD's `kubectl` with additional developer functions. `kubectl` is provided for die-hard kube fanboys - it will work with OKD as per normal Kubernetes. |
| `openshift-install-linux-4.6.0-0.okd-2020-12-12-135354.tar.gz` | The `openshift-install` command - a monster binary containing terraform for all the supported architectures for OKD |

Untar/gz both of these.

### 1.2 Preparing the `install-config.yaml`

This step is where we use `openshift-install` to generate the `install-config.yaml`, which is effectively the same as the YAML file you feed to `eksctl create cluster`.

The `openshift-install` command presents a Golang-based ncurses menu system to allow you to build the architecture.

Execute and follow - the `>` character is the cursor with which to select items. You need to give the `--dir DIRNAME` parameter. This `DIRNAME` will be where the TF state file for the cluster will be kept, as well as the initial `kubeconfig` and the `kubeadmin` user password is kept, among other things like logs. It is best practice to name this dir the same as the cluster name. **KEEP THIS DIR SAFE**.

**NOTE**

- The interface is quite rudimentary, and does not indicate when pagination is needed; i.e., if you think a list of, say, pubkeys should be longer/keys are missing from the list, try hitting down arrow until you "scroll" past the bottom entry of the list.

#### 1.2.1 `openshift-install` pubkey selection
```
(AWS: oso_okd-admin)_[dsw@orgonon aws_dev]$ ./openshift-install create install-config --dir ckc1
? SSH Public Key  [Use arrows to move, enter to select, type to filter, ? for more help]
> /home/dsw/.ssh/id_ed25519.pub
  /home/dsw/.ssh/id_ed25519_github.pub
 ```
Select the key and press enter.

#### 1.2.2 `openshift-install` platform selection

Select the relevant platform:

```
(AWS: oso_okd-admin)_[dsw@orgonon aws_dev]$ ./openshift-install create install-config --dir ckc1
? SSH Public Key /home/dsw/.ssh/id_ed25519.pub
? Platform  [Use arrows to move, enter to select, type to filter, ? for more help]
> aws
  azure
  gcp
  openstack
  ovirt
  vsphere
```
#### 1.2.3 `openshift-install` AWS region selection

Select region. Again here, try scrolling off the bottom of the list to reveal more choices.

```
(AWS: oso_okd-admin)_[dsw@orgonon aws_dev]$ ./openshift-install create install-config --dir ckc1
? SSH Public Key /home/dsw/.ssh/id_ed25519.pub
? Platform aws
INFO Credentials loaded from default AWS environment variables 
? Region  [Use arrows to move, enter to select, type to filter, ? for more help]
  eu-central-1 (Europe (Frankfurt))
  eu-north-1 (Europe (Stockholm))
  eu-south-1 (Europe (Milan))
> eu-west-1 (Europe (Ireland))
  eu-west-2 (Europe (London))
  eu-west-3 (Europe (Paris))
  me-south-1 (Middle East (Bahrain))
```

#### 1.2.3 `openshift-install` base domain selection

This part we select the base domain for the cluster (not the cluster name yet). This is optional, in that you can have:

`osodevops.io` (i.e, no subdomain)

or for our purposes herein:

`okd.osodevops.io` (tidy, in that we collect our OKD activities in an entire subdomain)

```
(AWS: oso_okd-admin)_[dsw@orgonon aws_dev]$ ./openshift-install create install-config --dir ckc1
? SSH Public Key /home/dsw/.ssh/id_ed25519.pub
? Platform aws
INFO Credentials loaded from default AWS environment variables 
? Region eu-west-1
? Base Domain  [Use arrows to move, enter to select, type to filter, ? for more help]
> okd.osodevops.io
```

#### 1.2.4 `openshift-install` cluster name input

Now we choose the cluster name. Note, that by default, the ingress controller will prepend its own subdomain named `apps` (and an additional wildcard before that!) to the cluster name, leaving us with a fairly tiring:

`*.apps.clustername.subdomain.osodevops.io`

We can get rid of the `apps` part - the pros and cons are discussed later in this document (`TODO: insert link`).

```
(AWS: oso_okd-admin)_[dsw@orgonon aws_dev]$ ./openshift-install create install-config --dir ckc1
? SSH Public Key /home/dsw/.ssh/id_ed25519.pub
? Platform aws
INFO Credentials loaded from default AWS environment variables 
? Region eu-west-1
? Base Domain okd.osodevops.io
? Cluster Name [? for help] ckc1
```

I've opted for `ckc1` - an arbitrary choice, call it what you like, but it's not changeable after installation, so bare this in mind for multi-tenant clusters.

#### 1.2.4 `openshift-install` pull secret input

The pull secret. Red Hat's (or should that now be Big Blue Hat?) "entitlement key" for accessing their paid-for registries (currently `registry.redhat.io`) which provide images which they actively scan/curate. This is currently a legal grey area, in that you are free to download an "official" pull secret via the Red Hat Developer Program (such as when you download the one-node Code Ready Container bundle, which allows you to bring up OpenShift on your laptop or something; much like minikube) and use it for OpenShift/OKD, but who knows when IBM will come calling (telemetry is actually pumped back to Red Hat on OpenShift/OKD - can be disabled). If this is deployed at a customer and they've been using a pull secret to which they are sort of legally not entitled, then it could spell trouble.

The best way forward is to use a `NULL` pull secret:

```
{"auths":{"fake":{"auth": "bar"}}}
```

This means we have to use the old/deprecated registry (`registry.access.redhat.com`), and don't get access to the official operators initially (only the community ones), but we can easily enable them afterwards.

Paste the NULL pull secret and press enter:

```
(AWS: oso_okd-admin)_[dsw@orgonon aws_dev]$ ./openshift-install create install-config --dir ckc1
? SSH Public Key /home/dsw/.ssh/id_ed25519.pub
? Platform aws
INFO Credentials loaded from default AWS environment variables 
? Region eu-west-1
? Base Domain okd.osodevops.io
? Cluster Name ckc1
? Pull Secret [? for help] **********************************
INFO Install-Config created in: ckc1              
(AWS: oso_okd-admin)_[dsw@orgonon aws_dev]$ 
```
Now we have the `install-config.yaml` file inside our artefact directory (`ckc1` - conveniently we name it the same as the cluster name).

### 1.3. Customising the installation (before installation itself)

The processing of the `install-config.yaml` and the general installation workflow is like this:

`install-config.yaml` --> `manifest yamls` --> `ignition configs` --> `cluster creation`

We can elect to go straight to the cluster creation, skipping dealing with the `install-config.yaml` if we wish (but the creation menu will still run us through the same menu as we do for generating the file).

It's best to always generate the `install-config.yaml` file, so that we can re-use it for future clusters and save time.

We can use the `install-config.yaml` to generate the cluster manifest yamls, which allow us to customise the installation before it begins. This, as mentioned previously, is where we can remove the annoying `apps` ingress subdomain, as well as update the AWS instance type of the nodes, etc.

#### 1.3.1 Altering node counts

Before we use the `install-config.yaml`, we can investigate it and change various aspects. We can change the VPC, subnets, the amount of node counts (replicas), among other things.

Here is the `install-config.yaml` which we just generated:

```
apiVersion: v1
baseDomain: okd.osodevops.io
compute:
- architecture: amd64
  hyperthreading: Enabled
  name: worker
  platform: {}
  replicas: 3
controlPlane:
  architecture: amd64
  hyperthreading: Enabled
  name: master
  platform: {}
  replicas: 3
metadata:
  creationTimestamp: null
  name: ckc1
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineNetwork:
  - cidr: 10.0.0.0/16
  networkType: OVNKubernetes
  serviceNetwork:
  - 172.30.0.0/16
platform:
  aws:
    region: eu-west-1
publish: External
pullSecret: '{"auths":{"fake":{"auth": "bar"}}}'
sshKey: |
  ssh-ed25519 XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX dsw@orgonon
```
If you wish to have more, or less workers, then you can adjust this as above. However, do not change the number of masters, unless you wish to have a single master, in which case select `1`. Further to this, you can additionally use `0` workers, and the installer will know to mark the single master as `Schedulable` so that you can actually use it as a master/worker hybrid. This is a cheap way to test things - but you won't be able to update the cluster to a new version, etc.

If in doubt, leave it as default 3 masters, 3 workers.

#### 1.3.2 A word on removing `apps` from the ingress/`Router`

As previously mentioned in step 1.2.4, we can do away with the annoying `apps` subdomain. If we keep it, then all pods/workloads will have the following nomenclature:

    <service>-<project>.apps.<clustername>.<basedomain>.osodevops.io

**NOTE**
- OKD uses the term `Project` for the Kubernetes `Namespace` object. The `Project` has additional security such as uid range, and `oc project <PROJECT>` will do the equivalent of `kubectl config set-context --current --namespace <NAMESPACE>`, which is a convenient life hack for us.

As you can see from the above FQDN placeholder for a typical deployment, it can get ugly. The default OKD console address for our impending install will be:

    console-openshift-console.apps.ckc1.okd.osodevops.io
    
Truly grim. However, the general rule of thumb on OpenShift in general is to not touch the defaults for the default subsystems deployed on our behalf by the system, unless you absolutely have to - madness that way lies. Exceptions to this rule are the Ingress (known as the `Router`) - replacing the self-signed cert is a matter of course.

By removing the `apps`, this then is reduced to:

    <service>-<project>.<clustername>.<basedomain>.osodevops.io
    
A word to the wise, however: the Kubernetes API is listening on the following address:

    api.<clustername>.<basedomain>.osodevops.io:6443

If we remove `apps`, then we "fall back" into the same subdomain level as the API. This means, on the entire cluster, we cannot have *any* app use the `api` name for ingress/`Route`. Of course, you can have this, though:

    api-myapp.<clustername>.<basedomain>.osodevops.io
    
But you see where this is going. Caveat Emptor.


#### 1.3.3 Generating the Manifests

**WARNING**
- The `install-config.yaml` lives, by default after creation, in the artefact directory. Any operations on it like manifest generation will actually _consume_ (delete) the file. This is a curious design choice, but I am told it's for the best. :shrug:.
- It's best practice to, post `install-config.yaml` generation, immediately copy the file out of the artefact directory so you have a backup copy. At worst it stops you needing to go through the rigmarole of the menu system again. At best it could be vital in understanding why a cluster isn't coming up (you could for example have selected the wrong subnet, or have a clashing CIDR, etc, and having access to the original yaml is good for troubleshooting that).

In order to generate the manifests so that we can customise, do the following:

```
(AWS: oso_okd-admin)_[dsw@orgonon aws_dev]$ ./openshift-install create manifests --dir ckc1
INFO Credentials loaded from default AWS environment variables 
INFO Consuming Install Config from target directory 
INFO Manifests created in: ckc1/manifests and ckc1/openshift 
(AWS: oso_okd-admin)_[dsw@orgonon aws_dev]$ 
```

Investigate the artefact directory:

```
(AWS: oso_okd-admin)_[dsw@orgonon aws_dev]$ cd ckc1/manifests/
(AWS: oso_okd-admin)_[dsw@orgonon manifests]$ ls
04-openshift-machine-config-operator.yaml  cluster-network-02-config.yml    etcd-metric-client-secret.yaml         etcd-signer-secret.yaml
cluster-config.yaml                        cluster-proxy-01-config.yaml     etcd-metric-serving-ca-configmap.yaml  kube-cloud-config.yaml
cluster-dns-02-config.yml                  cluster-scheduler-02-config.yml  etcd-metric-signer-secret.yaml         kube-system-configmap-root-ca.yaml
cluster-infrastructure-02-config.yml       cvo-overrides.yaml               etcd-namespace.yaml                    machine-config-server-tls-secret.yaml
cluster-ingress-02-config.yml              etcd-ca-bundle-configmap.yaml    etcd-service.yaml                      openshift-config-secret-pull-secret.yaml
cluster-network-01-crd.yml                 etcd-client-secret.yaml          etcd-serving-ca-configmap.yaml
(AWS: oso_okd-admin)_[dsw@orgonon manifests]$ 
```

For a typical install, we'll generally be interested in two items:

- `cluster-ingress-02-config.yml`
    It is here that we remove the `apps` part
- `cluster-infrastructure-02-config.yml`
    Change AWS instance type here

    
