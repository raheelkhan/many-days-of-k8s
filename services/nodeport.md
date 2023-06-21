# Nodeport

When we configure the Service of type Nodeport, Kubernetes will assign some port from range `30000-32767`. It will be the same port on every node and we can access the application by pointing to the IP address of any node with the port mentioned.

Lets try this example. I have created file `node-port-service.yaml` that is same as `basic-service.yaml` but we will specify the type of Service to `NodePort`.

After I applied the manifest I see the following

```bash
$ k get all -o wide
NAME                         READY   STATUS    RESTARTS   AGE   IP           NODE           NOMINATED NODE   READINESS GATES
pod/my-dep-8954bc845-thfbb   1/1     Running   0          6s    10.244.1.5   minikube-m02   <none>           <none>
pod/my-dep-8954bc845-vwlfl   1/1     Running   0          6s    10.244.2.3   minikube-m03   <none>           <none>

NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE   SELECTOR
service/http-service   NodePort    10.101.198.239   <none>        80:30001/TCP   6s    app.kubernetes.io/name=MyApp
service/kubernetes     ClusterIP   10.96.0.1        <none>        443/TCP        9d    <none>

NAME                     READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES   SELECTOR
deployment.apps/my-dep   2/2     2            2           6s    nginx        nginx    app.kubernetes.io/name=MyApp

NAME                               DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES   SELECTOR
replicaset.apps/my-dep-8954bc845   2         2         2       6s    nginx        nginx    app.kubernetes.io/name=MyApp,pod-template-hash=8954bc845
```

Now I will try to access the service from node IP and port. To proof further I used 2 replicas. So that I can test the NodePort from the node where the pod is not scheduled in this case `minikube` node.

```bash
curl 192.168.49.4:30001
```

All the nodes are properly accessing the service and I am able to see the nginx page coming from pod.

But the question is, Can I access service from the pod. Like we do in ClusterIP

```bash
$ k run busybox --image=busybox --command sleep infinity
$ k exec --stdin --tty busybox -- /bin/sh
#/ wget -q -O - 10.101.198.239
```

I can access the IP address of Service. It means even if we use NodePort type, We still get a IP address that can be used within cluster to communicate with service.