# Day 2 Operations

## 1. Scaling the machineset / changing instance type

As mentioned in installation, we'll need to change the instance type of the worker nodes, to allow for bloated deployments.

We do this using a `MachineSet` - this is similar to an EKS `NodeGroup`.

We first need to scale the machinesets down:

```
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ oc get machineset -n openshift-machine-api
NAME                           DESIRED   CURRENT   READY   AVAILABLE   AGE
ckc1-twqf6-worker-eu-west-1a   1         1         1       1           167m
ckc1-twqf6-worker-eu-west-1b   1         1         1       1           167m
ckc1-twqf6-worker-eu-west-1c   1         1         1       1           167m
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ 
```

As you can see, we have one node in each AZ. Let's scale the `MachineSet` in 1c (delete the node in 1c), so that we can edit 
the `MachineSet` and change the instance type. Once the instance type is set, we can scale it back up to 1, and we'll have a new
node with that instance type. This works the same as changing the Launch Config/template in an EKS `NodeGroup`.

Scale 1c down to 0:

```
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ oc scale machineset ckc1-twqf6-worker-eu-west-1c --replicas=0 -n openshift-machine-api
machineset.machine.openshift.io/ckc1-twqf6-worker-eu-west-1c scaled
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ 
```

Check that the node is exiting the cluster:

```
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ oc get nodes
NAME                                         STATUS                     ROLES    AGE    VERSION
ip-10-0-133-38.eu-west-1.compute.internal    Ready                      worker   164m   v1.19.2+7070803-1008
ip-10-0-147-135.eu-west-1.compute.internal   Ready                      master   177m   v1.19.2+7070803-1008
ip-10-0-172-152.eu-west-1.compute.internal   Ready                      worker   164m   v1.19.2+7070803-1008
ip-10-0-181-119.eu-west-1.compute.internal   Ready                      master   177m   v1.19.2+7070803-1008
ip-10-0-204-1.eu-west-1.compute.internal     Ready,SchedulingDisabled   worker   164m   v1.19.2+7070803-1008
ip-10-0-210-130.eu-west-1.compute.internal   Ready                      master   177m   v1.19.2+7070803-1008
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ 
```

Yep; it's on the way out.

:fire: WARNING :fire:
- Be sure to allow the node to fully exit the cluster before changing the `MachineSet` and scaling back up to 1. I have personally witnessed a
race condition where by scaling back up too quickly has caused trouble between OKD and the AWS API, such that I had a zombie node and I was 
unable to bring in the new node without serious messing around.

Now it's gone:

```
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ oc get machineset -n openshift-machine-api
NAME                           DESIRED   CURRENT   READY   AVAILABLE   AGE
ckc1-twqf6-worker-eu-west-1a   1         1         1       1           3h19m
ckc1-twqf6-worker-eu-west-1b   1         1         1       1           3h19m
ckc1-twqf6-worker-eu-west-1c   0         0                             3h19m
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ 
```

So let's edit the instance type. Invoke the following, and a `vi` session opens up:

```
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ oc edit machineset ckc1-twqf6-worker-eu-west-1c -n openshift-machine-api
```

Search for `instanceType` and change from `m5.xlarge` to `m5a.2xlarge`. Remember, with AMD instances we save money.

``` 
    ...
    deviceIndex: 0
    iamInstanceProfile:
      id: ckc1-twqf6-worker-profile
    instanceType: m5a.2xlarge                       <-- m5.xlarge -> m5a.2xlarge
    kind: AWSMachineProviderConfig
    metadata:
      creationTimestamp: null
    placement:
      availabilityZone: eu-west-1c
    ...
```
Write/quit out of `vi`:

```
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ oc edit machineset ckc1-twqf6-worker-eu-west-1c -n openshift-machine-api
machineset.machine.openshift.io/ckc1-twqf6-worker-eu-west-1c edited
(AWS: oso_okd-admin)_[dsw@orgonon helm]$
```

We can now scale back up:

```
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ oc scale machineset ckc1-twqf6-worker-eu-west-1c --replicas=1 -n openshift-machine-api
machineset.machine.openshift.io/ckc1-twqf6-worker-eu-west-1c scaled
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ 
```

You can check in the EC2 console and you'll see the new instance spawning. It should take around 5 mins to come into the cluster
as a new worker node - it will initially be `NotReady`, of course:

```
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ oc get nodes
NAME                                         STATUS     ROLES    AGE     VERSION
ip-10-0-133-38.eu-west-1.compute.internal    Ready      worker   3h12m   v1.19.2+7070803-1008
ip-10-0-147-135.eu-west-1.compute.internal   Ready      master   3h25m   v1.19.2+7070803-1008
ip-10-0-181-119.eu-west-1.compute.internal   Ready      master   3h24m   v1.19.2+7070803-1008
ip-10-0-208-27.eu-west-1.compute.internal    NotReady   worker   12s     v1.19.2+7070803-1008
ip-10-0-210-130.eu-west-1.compute.internal   Ready      master   3h24m   v1.19.2+7070803-1008
(AWS: oso_okd-admin)_[dsw@orgonon helm]$ 
```

Repeat the above steps for 1b, and then 1a. There's no need to do them in reverse order - this is just my preference.
