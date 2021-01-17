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

We also need to choose a name for our Kafka cluster, as well as select our domain:

```
......
## Kafka Cluster
##
kafka:
  name: kafka-oso
  replicas: 3
  disableHostPort: true
  resources:
    requests:
      cpu: 300m
      memory: 2Gi
  loadBalancer:
    enabled: false
    domain: "okd.osodevops.io"
  tls:
    enabled: false
    fullchain: |-
    privkey: |-
    cacerts: |-
  configOverrides:
    server:
    - auto.create.topics.enable=true
    - group.initial.rebalance.delay.ms=100
  metricReporter:
    enabled: true
......
```

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
Check that the operator pod is up:

```
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ oc get pods
NAME                          READY   STATUS    RESTARTS   AGE
cc-operator-95595745f-svn5m   1/1     Running   1          28s
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ 
```

## 1.5 Deploy ZooKeeper

### 1.5.1 Create an SCC-based role/rolebinding

In order to deploy ZK, we need to allow OKD to start privileged pods. This is where the is a deviation in the Confluent documentation for OpenShift/OKD.

Create a custom role/rolebinding which leverage the `anyuid` Security Context Constraint (`SCC`), allowing a pod to run as any UID. Do not try to install the bundled `randomUID` or `customUID` yamls, which purport to do this for you. Things changed between 4.3 and 4.6, such that you'll need to execute the following:

```
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ oc create role confluent-default-anyuid --verb=use --resource=scc --resource-name=anyuid -n confluent
role.rbac.authorization.k8s.io/confluent-default-anyuid created
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ oc create rolebinding confluent-default-anyuid --role=confluent-default-anyuid --serviceaccount=confluent:default -n confluent
rolebinding.rbac.authorization.k8s.io/confluent-default-anyuid created
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ 
```
This is allowing the default `ServiceAccount` to start pods with arbitrary UIDs.

### 1.5.2 Adjust the `StorageClass` to support Multi-AZ

We need to update the `StorageClass` in the `values.yaml` file to use a zonally-aware `volumeBindingMode`. This is not considered in Confluent's documentation as far as I can see, but it will result in volume conflicts when scheduling pods. By default, the mode is `Immediate`, which creates an EBS volume and binds it to the node which registered the PVC. This is unlikely to always line up with
the right Pod in the the right node in the right AZ, and so the Pod (or even the node) will be unable to use the PV. By setting the
mode to `WaitForFirstConsumer`, the PV is guaranteed to be created in the same AZ as the Pod, and therefore the node.

```
storage:
  provisioner: kubernetes.io/aws-ebs
  volumeBindingMode: WaitForFirstConsumer   <-- Add this
  reclaimPolicy: Delete                     <-- Optionaly make this Retain if you want to keep data around after PVC deletion
  parameters:
    encrypted: "false"
    kmsKeyId: ""
    type: gp2
```

### 1.5.3 Create the ZooKeeper ensemble

Now we can deploy ZK:

```
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ helm3 install zookeeper ./confluent-operator -f values.yaml --namespace confluent --set zookeeper.enabled=true
NAME: zookeeper
LAST DEPLOYED: Sun Jan 17 15:13:46 2021
NAMESPACE: confluent
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Zookeeper Cluster Deployment

Zookeeper cluster is deployed through CR.

  0. List of Deprecated Features (if any)
     

  1. Validate if Zookeeper Custom Resource (CR) is created

     kubectl get zookeeper -n confluent | grep zookeeper

  2. Check the status/events of CR: zookeeper

     kubectl describe zookeeper zookeeper -n confluent

  3. Check if Zookeeper cluster is Ready

     kubectl get zookeeper zookeeper -ojson -n confluent

     kubectl get zookeeper zookeeper -ojsonpath='{.status.phase}' -n confluent

  4. Update/Upgrade Zookeeper Cluster

     The upgrade can be done either through the helm upgrade or by editing the CR directly as below;

     kubectl edit zookeeper zookeeper  -n confluent

Note: For Openshift Platform replace kubectl commands with 'oc' commands. If OpenShift Route enabled, port will be either 80/443
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ 
```

**NOTE**
- We are given a heads-up by the deployment `NOTES` as to how to check that all is well:

```
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ echo $(oc get zookeeper zookeeper -ojsonpath='{.status.phase}' -n confluent)
RUNNING
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ 
```

**NOTE**
- If you see `PENDING`, you can check the output of `oc get events --sort-by=.metadata.creationTimestamp`. It could be, that your cluster does
not have enough resources, or more commonly, `Pending` deployments hanging indefinitely are usually due to PV-based problems (such as the
`StorageClass` issue mentioned above, so be sure you switched the `StorageClass` to `WaitForFirstConsumer`.

The pods also show we are up:

```
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ oc get pods
NAME                          READY   STATUS    RESTARTS   AGE
cc-operator-95595745f-svn5m   1/1     Running   1          17m
zookeeper-0                   1/1     Running   0          4m53s
zookeeper-1                   1/1     Running   0          4m53s
zookeeper-2                   1/1     Running   0          4m53s
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ 
```
## 1.6 Deploy Kafka Brokers

Now we can bring up the Kafka cluster itself. We should have already named the cluster from step 1.3. We elected to call our
particular one `kafka-oso`, and we specify that on command line:

```
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ helm3 install kafka-oso ./confluent-operator -f values.yaml --namespace confluent --set kafka.enabled=true
NAME: kafka-oso
LAST DEPLOYED: Sun Jan 17 15:21:59 2021 
NAMESPACE: confluent
STATUS: deployed                                                              
REVISION: 1
TEST SUITE: None                                                              
NOTES:
Kafka Cluster Deployment    
                                                                                                                                                             
Kafka Cluster is deployed to kubernetes through CR Object
                                                                              
  0. List of Deprecated Features (if any)
                                                                              

  1. Validate if Kafka Custom Resource (CR) is created                                                                                                       

     kubectl get kafka -n confluent | grep kafka-oso

  2. Check the status/events of CR: kafka-oso    
                                                                              
     kubectl describe kafka kafka-oso -n confluent
                                                                                                                                                             
  3. Check if Kafka cluster is Ready
                                                                              
     kubectl get kafka kafka-oso -ojson -n confluent
                                                                              
     kubectl get kafka kafka-oso -ojsonpath='{.status.phase}' -n confluent
                                                                              
  4.  Broker Listener (Protocol/Port)
                                                                                                                                                             
      External Listener: kubectl  -n confluent  get kafka kafka-oso -ojsonpath='{.status.brokerExternalListener}'
      Internal Listener: kubectl  -n confluent  get kafka kafka-oso -ojsonpath='{.status.brokerInternalListener}'

      Note: If Protocol is SSL, configure truststore (https://docs.confluent.io/current/kafka/encryption.html#clients) and keystore 
        (https://docs.confluent.io/current/kafka/authentication_ssl.html#clients) if client Authentication is enabled (
        kubectl  -n confluent  get kafka kafka-oso -ojsonpath='{.status.clientAuthentication}' )                                                

  5. Update/Upgrade Kafka Cluster                                                                                                                            
                                                                              
     The upgrade can be done either through the helm upgrade or by editing the CR directly as below;

     kubectl edit kafka kafka-oso  -n confluent

     Note: Switching Kafka security requires manual restart of all the dependent components as the JAAS configuration changes is required.

  6. All Kafka Information like zookeeper connect, replications factor, isr, Client Jaas Configuration
     and much more can be found on the Status section of CR.

     kubectl get kafka kafka-oso -n confluent -oyaml

  7. Client Access:

     Run below command to validate if Kafka cluster is working.

     1. Get the Client Jaas Information 
        Internal : kubectl  -n confluent  get kafka kafka-oso -ojsonpath='{.status.internalClient}' > kafka.properties
     2. To get the Bootstrap endpoint

        kubectl -n confluent get kafka kafka-oso -ojsonpath='{.status.bootstrapEndpoint}'

     3. To get the Replication Factor

        kubectl -n confluent get kafka kafka-oso -ojsonpath='{.status.replicationFactor}'

     4. To get the Zookeeper Connect endpoint

        kubectl -n confluent get kafka kafka-oso -ojsonpath='{.status.zookeeperConnect}'
    Internal:

        - Go to one of the Kafka Pods

          kubectl -n confluent exec -it kafka-oso-0 bash

        - Inside the pod, run below command

cat <<EOF > kafka.properties
## Copy information from kafka.properties available in step 7 (Client Access) step 1.
EOF
            + Check brokers API versions

              Get <bootstrapEndpoint> from step 2

              kafka-broker-api-versions --command-config kafka.properties --bootstrap-server <bootstrapEndpoint>

            + Run command to create topic

              Get <replicationFactor> from step 3
              Get <zookeeperConnect> from step 4

              kafka-topics --create --zookeeper <zookeeperConnect> --replication-factor <replicationFactor> --partitions 1 --topic example

              Note: Above command only works inside the kubernetes network

            + Run command to Publish events on topic example

              Get <bootstrapEndpoint> from step 2

              seq 10000 | kafka-console-producer --topic example --broker-list <bootstrapEndpoint> --producer.config kafka.properties

            + Run command to Consume events on topic example (run in different terminal)

              Get <bootstrapEndpoint> from step 2

              kafka-console-consumer --from-beginning --topic example --bootstrap-server  <bootstrapEndpoint> --consumer.config kafka.properties

Note: For Openshift Platform replace kubectl commands with 'oc' commands. If OpenShift Route enabled, port will be either 80/443
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ 
```

Again here we can check that all is running nicely:

```
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ echo $(oc get kafka kafka-oso -ojsonpath='{.status.phase}' -n confluent)
RUNNING
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ 
```

.. as well as interrogate the pod list to see it all `Running`:

```
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ oc get pods
NAME                          READY   STATUS    RESTARTS   AGE
cc-operator-95595745f-svn5m   1/1     Running   1          24m
kafka-oso-0                   1/1     Running   0          3m42s
kafka-oso-1                   1/1     Running   0          3m42s
kafka-oso-2                   1/1     Running   0          3m42s
zookeeper-0                   1/1     Running   0          11m
zookeeper-1                   1/1     Running   0          11m
zookeeper-2                   1/1     Running   0          11m
(AWS: oso_okd-admin)_[dsw@orgonon helm]$
```

## 1.7 Deploy Schema Registry

Deploy as follows:

```
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ helm3 install schemaregistry ./confluent-operator -f values.yaml --namespace confluent --set schemaregistry.enabled=true
NAME: schemaregistry
LAST DEPLOYED: Sun Jan 17 15:27:19 2021
NAMESPACE: confluent
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Schema Registry is deployed through PSC. Configure Schema Registry through REST Endpoint

  0. List of Deprecated Features (if any)
     

  1. Validate if schema registry cluster is running

     kubectl get pods -n confluent | grep schemaregistry

  2. Access
    Internal REST Endpoint : http://schemaregistry:8081  (Inside kubernetes)

    OR

    http://localhost:8081 (Inside Pod)

    More information about schema registry REST API can be found here,

    https://docs.confluent.io/current/schema-registry/docs/api.html

Note: For Openshift Platform replace kubectl commands with 'oc' commands. If OpenShift Route enabled, port will be either 80/443
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ 
```

Check that we are up and running:

```
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ oc get pods
NAME                          READY   STATUS    RESTARTS   AGE
cc-operator-95595745f-svn5m   1/1     Running   1          26m
kafka-oso-0                   1/1     Running   0          5m49s
kafka-oso-1                   1/1     Running   0          5m49s
kafka-oso-2                   1/1     Running   0          5m49s
schemaregistry-0              0/1     Running   0          32s           <-- 
schemaregistry-1              0/1     Running   0          32s           <--
zookeeper-0                   1/1     Running   0          14m
zookeeper-1                   1/1     Running   0          14m
zookeeper-2                   1/1     Running   0          14m
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ 
```

## 1.8 Deploy Control Center

```
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ helm3 install controlcenter ./confluent-operator -f values.yaml --namespace confluent --set controlcenter.enabled=true
NAME: controlcenter
LAST DEPLOYED: Sun Jan 17 15:29:26 2021
NAMESPACE: confluent
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
ControlCenter Deployment

ControlCenter is deployed through PSC

   0. List of Deprecated Features (if any)
      

   1. Validate if controlcenter is running

     kubectl get pods -n confluent | grep controlcenter

   2. Access
      External: http://controlcenter.rd-sky.net:80
      Internal: http://controlcenter:9021 (Inside Kubernetes)
      
      Local Test:

        kubectl -n confluent port-forward controlcenter-0 12345:9021
        Open on browser: http://localhost:12345

   3. Authentication: Basic

Note: For Openshift Platform replace kubectl commands with 'oc' commands. If OpenShift Route enabled, port will be either 80/443
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ 
```
And check all is well:

```
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ oc get pods
NAME                          READY   STATUS    RESTARTS   AGE
cc-operator-95595745f-svn5m   1/1     Running   1          29m
controlcenter-0               0/1     Running   0          80s
kafka-oso-0                   1/1     Running   0          8m45s
kafka-oso-1                   1/1     Running   0          8m45s
kafka-oso-2                   1/1     Running   0          8m45s
schemaregistry-0              1/1     Running   0          3m28s
schemaregistry-1              1/1     Running   0          3m28s
zookeeper-0                   1/1     Running   0          16m
zookeeper-1                   1/1     Running   0          16m
zookeeper-2                   1/1     Running   0          16m
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ 
```
## 1.9 Deploy Kafka Connect Framework

```
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ helm3 install connectors ./confluent-operator -f values.yaml --set connect.enabled=true
NAME: connectors
LAST DEPLOYED: Sun Jan 17 15:31:39 2021
NAMESPACE: confluent
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Connect Cluster Deployment

Connect Cluster is deployed through PSC. To configure connectors use ControlCenter or Use REST API to configure.
The Connect Cluster endpoint does not support authorization and must not be open for Internet Access.

  0. List of Deprecated Features (if any)
     

  1. Validate if connect cluster is running

     kubectl get pods -n confluent | grep connectors

  2. Access
     Internal REST Endpoint : http://connectors:8083 (Inside Kubernetes)

     OR

     http://localhost:8083 (Inside Pod)

     More information can be found here: https://docs.confluent.io/current/connect/references/restapi.html

Note: For Openshift Platform replace kubectl commands with 'oc' commands. If OpenShift Route enabled, port will be either 80/443
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ 
```

Check that everything is up:

```
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ oc get pods
NAME                          READY   STATUS    RESTARTS   AGE
cc-operator-95595745f-svn5m   1/1     Running   1          31m
connectors-0                  0/1     Running   0          28s
controlcenter-0               1/1     Running   0          2m40s
kafka-oso-0                   1/1     Running   0          10m
kafka-oso-1                   1/1     Running   0          10m
kafka-oso-2                   1/1     Running   0          10m
schemaregistry-0              1/1     Running   0          4m48s
schemaregistry-1              1/1     Running   0          4m48s
zookeeper-0                   1/1     Running   0          18m
zookeeper-1                   1/1     Running   0          18m
zookeeper-2                   1/1     Running   0          18m
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ 
```

## 2.0 Expose Control Center to the World

Now that Kafka has been deployed, we can open the `Ingress/Route` up to the world (or wherever your VPC is scoped).

However, first we must get rid of the **PORT 80** (:cry:) AWS LB which the operator created on our behalf, and also create a more secure route:

```
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ oc delete svc controlcenter-bootstrap-lb
service "controlcenter-bootstrap-lb" deleted
(AWS: oso_okd-admin)_[dsw@orgonon helm]$          
```

And now the route:

```
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ oc create route edge controlcenter --service controlcenter --hostname franz.ckc1.okd.osodevops.io
route.route.openshift.io/controlcenter created
(AWS: oso_okd-admin)_[dsw@orgonon helm]$
```

All done - Control Center is now available on https://franz.ckc1.okd.osodevops.io.
