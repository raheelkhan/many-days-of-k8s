# Scheluding

## Using nodeName

We can supply a `nodeName` parameter in the parameter spec. In this case for example I am creating a deployment using file `node-name-deployment.yaml`. Iwill put a nodeName selector in it and observe the behaviour.

As expected I can see my pods are scheduled on the desired node
```bash
$ k get pods -o wide
NAME                                   READY   STATUS    RESTARTS   AGE   IP             NODE         NOMINATED NODE   READINESS GATES
node-name-deployment-8c9bb6cff-992rq   1/1     Running   0          27s   10.244.192.1   kubenode01   <none>           <none>
node-name-deployment-8c9bb6cff-fwb28   1/1     Running   0          27s   10.244.192.2   kubenode01   <none>           <none>
```

## Using nodeSelector

The benefit of using `nodeSelector` is that now I am not tied to one node. It works by putting a label on nodes and let the scheduler place pods based on the value that we specify on the `nodeSelector` attribute. For this purpose I will modify the file `node-name-deployment` and make a copy of it in a file `node-selector-deployment.yaml` and use nodeSelector. 

But before using `nodeSelector` I must verify that the nodes have some labels that I can use.

Right now the default labels looks like 
```bash
$ k get nodes --show-labels
NAME         STATUS   ROLES           AGE   VERSION   LABELS
kubemaster   Ready    control-plane   45h   v1.27.3   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=kubemaster,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node.kubernetes.io/exclude-from-external-load-balancers=
kubenode01   Ready    <none>          45h   v1.27.3   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=kubenode01,kubernetes.io/os=linux
kubenode02   Ready    <none>          45h   v1.27.3   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=kubenode02,kubernetes.io/os=linux
```

But I want to put explicit label on the worker nodes so that I can deploy the pods on worker nodes only and not on `kubemaster`. This is right now not possible because of missing labels. I will add label `nodetype=worker` to `kubenode01` and `kubenode02`

```bash
raheel at raheel-ThinkPad-T440s in ~/Playground/many-days-of-k8s/scheduling (master)
$ k label nodes kubenode01 nodetype=worker
node/kubenode01 labeled

raheel at raheel-ThinkPad-T440s in ~/Playground/many-days-of-k8s/scheduling (master)
$ k label nodes kubenode02 nodetype=worker
node/kubenode02 labeled
```

As you can see the labels are now attached to the worker nodes
```bash
$ k get nodes --show-labels
NAME         STATUS   ROLES           AGE   VERSION   LABELS
kubemaster   Ready    control-plane   46h   v1.27.3   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=kubemaster,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node.kubernetes.io/exclude-from-external-load-balancers=
kubenode01   Ready    <none>          45h   v1.27.3   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=kubenode01,kubernetes.io/os=linux,nodetype=worker
kubenode02   Ready    <none>          45h   v1.27.3   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=kubenode02,kubernetes.io/os=linux,nodetype=worker
```

I have applied the `nodeSelector` spec now and now lets see where the pods get scheduled.

```bash
raheel at raheel-ThinkPad-T440s in ~/Playground/many-days-of-k8s/scheduling (master)
$ k apply -f node-selector-deployment.yaml
deployment.apps/node-name-deployment created

raheel at raheel-ThinkPad-T440s in ~/Playground/many-days-of-k8s/scheduling (master)
$ k get pods -o wide
NAME                                    READY   STATUS    RESTARTS   AGE   IP             NODE         NOMINATED NODE   READINESS GATES
node-name-deployment-55494b74cb-6qkps   1/1     Running   0          7s    10.244.192.1   kubenode01   <none>           <none>
node-name-deployment-55494b74cb-bztlz   1/1     Running   0          7s    10.244.64.1    kubenode02   <none>           <none>
```
It shows that the nodeSelector has successfully selected nodes with the correct labels.

One thing to remember in nodeSelector is that they are effective only at the time of scheduling. Let say If I used a nodeSelector `nodetype=worker` and the pod gets scheduled. If now I remove the label from the node, the pod will still run on the same node.
