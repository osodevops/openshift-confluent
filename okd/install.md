# Installing OpenShift/OKD

[[_TOC_]]

## 1. Preparation and Prerequisites

In order to install an OpenShift (hereinafter referred to as `OKD`) cluster on AWS, we first need a subdomain, which will also be the name of the cluster itself; OKD's architecture is such that DNS is bound tightly to core aspects of the setup, which is challenging in on-prem scenarios.

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


    
