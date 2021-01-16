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

Execute and follow - the `>` character is the cursor with which to select items. You need to give the `--dir DIRNAME` parameter. This `DIRNAME` will be where the TF state file for the cluster will be kept, as well as the initial `kubeconfig` and the `kubeadmin` user password is kept, among other things like logs. **KEEP THIS DIR SAFE**.

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
Now we have the `install-config.yaml` file




    
