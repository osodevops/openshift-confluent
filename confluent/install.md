# 1. Installing the Confluent Operator

## 1.1 Obtaining the Confluent Operator

Create an artefact/working directory for yourself, say, `confluent-operator-6.0.0`.

Then obtain/untar the following package:

```
[dsw@orgonon confluent-operator-6.0.0]$ curl -qs https://platform-ops-bin.s3-us-west-1.amazonaws.com/operator/confluent-operator-1.6.1-for-confluent-platform-6.0.0.tar.gz | tar xvzf -                                                                                                     helm/                                                                         
helm/confluent-operator/                                                      
helm/providers/                                                               
helm/scripts/                                                                 
helm/scripts/openshift/                                                       
. . . 
resources/crds/
resources/crds/physicalstatefulcluster.yaml
resources/crds/zookeepercluster.yaml
resources/crds/kafkacluster.yaml
resources/rbac/clusterrole.yaml
resources/rbac/clusterrolebinding.yaml
[dsw@orgonon confluent-operator]$ 
```

## 1.2 Create a new `Project`

Recall, a `Project` is the OKD ~equivalent of Kubernetes' `Namespace`. Let's create one to deploy Confluent to:

```
(AWS: oso_okd-admin)_[dsw@orgonon confluent-operator-6.0.0]$ oc new-project confluent
Now using project "confluent" on server "https://api.ckc1.okd.osodevops.io:6443".
	   
You can add applications to this project with the 'new-app' command. For example, try:
	   
    oc new-app ruby~https://github.com/sclorg/ruby-ex.git
	   
to build a new example application in Ruby. Or use kubectl to deploy a simple Kubernetes application:
	   
    kubectl create deployment hello-node --image=gcr.io/hello-minikube-zero-install/hello-node
	   
(AWS: oso_okd-admin)_[dsw@orgonon confluent-operator-6.0.0]$ 
```

## 1.3 Customise the values.yaml

We need to start from the AWS values template:

```
(AWS: oso_okd-admin)_[dsw@orgonon confluent-operator-6.0.0]$ ls
COPYRIGHT  grafana-dashboard  helm  IMAGES  resources  scripts
(AWS: oso_okd-admin)_[dsw@orgonon confluent-operator-6.0.0]$ cd helm
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ cp providers/aws.yaml values.yaml
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ 
```
The values.yaml file needs to be customised to fit our AWS deployment:

```
## Overriding values for Chart's values.yaml for AWS
##
global:
  provider:
    name: aws
    region: eu-west-1
    kubernetes:
      deployment:
        ## If kubernetes is deployed in multi zone mode then specify availability-zones as appropriate
        ## If kubernetes is deployed in single availability zone then specify appropriate values
        zones:
         - eu-west-1a
         - eu-west-1b
         - eu-west-1c
......
```
TODO: Add the rest.

## 1.4 Use helm3 to install Operator

Using Helm v3, install the operator:

```
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ helm3 install operator ./confluent-operator -f values.yaml --namespace confluent --set operator.enabled=true
NAME: operator
LAST DEPLOYED: Tue Jan  5 11:59:13 2021
NAMESPACE: confluent
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The Confluent Operator
	   
The Confluent Operator interacts with kubernetes API to create statefulsets resources. The Confluent Operator runs three
controllers, two component specific controllers for kubernetes by providing components specific Custom Resource
Definition (CRD) (for Kafka and Zookeeper) and one controller for creating other statefulsets resources.
	   
1. Validate if Confluent Operator is running.
	   
kubectl get pods -n confluent | grep cc-operator
	   
2. Validate if custom resource definition (CRD) is created.
	   
kubectl get crd | grep confluent
	   
Note: For Openshift Platform replace kubectl commands with 'oc' commands. If OpenShift Route enabled, port will be either 80/443
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ 
```
