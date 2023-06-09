We can limit how much resources a Pod can consume on a node on which it is scheduled. In kubernetes we can define this constraints in terms of `request` and `limit`.

> CPU units
CPU resource is measured in `cpu units`. 1 cpu unit is equal to 1 cpu core. we can also supply unit in fractions for example if we want to assign a request of half cpu we can say `0.5`. This means a Pod can use upto half of the total CPU time in the node. 

Also fractions can be represented with the expression of millicpu or millicore. This means that `0.1` can be written as `100m`, `0.5` can be written as `500m` and so on.

To better understand, 1 CPU = 1000 millicores or millicpu
This means if I have a Pod with 2 containers and I want both of them to have an equal share of CPU time then I will assign `request` of `500m` to both

Similarly If I have a node with 4 CPU. and I want 3 containers to share this I could do 4000/3 = 1333m

Lets do a try here. I will create a one node minikube cluster and first I will check the Node cpu count.

I have a minikube cluster with 3 nodes including the controlplane. I check the node specs and all of them a 4 cpu 

```bash
$ k describe node minikube
Capacity:
  cpu:                4
  ephemeral-storage:  244504892Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             7823600Ki
  pods:               110
Allocatable:
  cpu:                4
  ephemeral-storage:  244504892Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             7823600Ki
  pods:               110
```

I will now try to create a container and requests `5000m`. Lets see what happen. I have created a file called `requests-excess-cpu.yaml`

```bash
$ k get pods --watch
NAME         READY   STATUS    RESTARTS   AGE
excess-cpu   0/1     Pending   0          8s
```

I observed that the Pod is in pending state. Which is good because there is no node in my cluster which as 5 cpu. Its interesting to see that kubernetes actually informed about it

```bash
$ k describe pod excess-cpu
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  75s   default-scheduler  0/3 nodes are available: 3 Insufficient cpu. preemption: 0/3 nodes are available: 3 No preemption victims found for incoming pod..
```
I changed it to `4000m` but still it complained about insufficient cpu. But it worked when I used `3000m`. 

So one of the takeway from this note is that if the pod are in Pending state, we better check its resource constraints and compare it with the nodes present in the cluster.

Also it is important to understand that when kubernetes tries to assign pod to node, it checks requests for all the containers in the pod and sum them before deciding where it should place the pod. Lets try this again. I will add one more container to the same file and make the sum of both containers requests higher than the node's capacity.

As expected the pod itself was not scheduled. It is not possible in kubernetes that one container can be alive and other is not scheduled. Because the atomic unit in kubernetes is a pod. It can either be fully scheduler or pending.

Another this to note is the difference between `request` and `limit`.

Request specifies the minimum requirements. For example if I say in the pod manifest `3000m` cpu and I have a node with `5000m` cpu. Then the pod may use the remaining `2000m` if its idle. 

If we want to enforce the upper limit and I want to say that the container should never use more than `3000` cpu then I have to specify that in `limits`. Both have similar syntax

```yaml
spec:
  containers:
  - name: app
    image: images.my-company.example/app:v4
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

> Memory

Its a boring topic just see [Memory Requests](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#meaning-of-memory)