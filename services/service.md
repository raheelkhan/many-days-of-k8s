# Servies

Kubernetes `Service` provides a single entry point for backing pods. In kubernetes, pods are ephemeral. Imagine I have set of backend api pods, managed by a `Deployment`. There are few challenges for me if there is no Service fronting those pods.

- How would my frontend pods / code know how many backend pods are there. It has to manage a set of IP addresses for those backend pods. It will be client responsibility to distribute traffic among those backend pods.

- When a pod dies, the deployment will spin up its subsitute, this time the pod will have a different IP address. We will have to somehow tell the client code to replace the old pod IP with the new IP.

In order to solve this problem, Kubernetes provides a Service object. Service provide a static IP address then can be distributed to the client code. It then manages the pods behind the scene. If a pod dies it knows about it and when a new pod comes in it knows about it.

Lets see how Service knows which pod(s) it has to send traffic to.

First I used imperative command to generate a manifiest file that contains deployment configuration
```bash
$ kubectl create deployment my-dep --image=nginx --replicas=3 --dry-run=client -o yaml > basic-service.yaml
```

I learned that service maintains a set of `Endpointslices`. When we create a Service, its controller keep scanning any pods that match the selectors and continously updates the Endpoint slices.

I will do the following expirement now

1 - Create a deployment with 3 repicas 
2 - Create a service that use the selectors that match label of the deployment pods
3 - View whats created in the Endpointslices
4 - scale up or down the deployment and see the change happening in endpoint slices.

Remember to name service that is a valid domain name. I first used `MyService` and it failed.

I applied the `basic-service.yaml` file and noticed the following resources created

```bash
$ k get all
NAME                         READY   STATUS    RESTARTS   AGE
pod/my-dep-8954bc845-9mkvc   1/1     Running   0          51s
pod/my-dep-8954bc845-9pcrm   1/1     Running   0          51s
pod/my-dep-8954bc845-xfq4c   1/1     Running   0          51s

NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/http-service   ClusterIP   10.109.218.68   <none>        80/TCP    3s
service/kubernetes     ClusterIP   10.96.0.1       <none>        443/TCP   6d22h

NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/my-dep   3/3     3            3           51s

NAME                               DESIRED   CURRENT   READY   AGE
replicaset.apps/my-dep-8954bc845   3         3         3       51s
```

I will discuss the relation between `Deployment` and `ReplicaSet` in a different document.

Lets see how to access the service.

I sshed into one of the node and tried to do `curl 10.109.218.68` and I was able to see the nginx page. This means that the service is properly able to forward the traffic to one of the three pods that are backing the service.

If I want to see how many pods the service is fronting 

```bash
$ k describe svc http-service
Name:              http-service
Namespace:         default
Labels:            <none>
Annotations:       <none>
Selector:          app.kubernetes.io/name=MyApp
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.109.218.68
IPs:               10.109.218.68
Port:              <unset>  80/TCP
TargetPort:        http-web-svc/TCP
Endpoints:         10.244.0.3:80,10.244.1.2:80,10.244.2.2:80
Session Affinity:  None
Events:            <none>
```

I scaled down the deployment by running the command
```bash
k scale --replicas=2 deployment/my-dep
```

Lets see if it is reflected in the service object.

```bash
$ k describe svc http-service
...
Endpoints:         10.244.0.3:80,10.244.2.2:80
```

Indeed it reflects that now the Endpoints have two IP address belonging to the two pods under deployment.

So far, my tests proved that if we create a service of type `ClusterIP` we can a virtual IP. This IP address is accessable from all the nodes. I have to check if it is resolvable from inside the pod also or not.

I spinned up a quick busybox pod and I was able to reach the service IP

```bash
$ k run busybox --image=busybox --command sleep infinity
$ k exec --stdin --tty busybox -- /bin/sh

# wget -q -O - 10.109.218.68
```

Kubernetes also put env variables inside pod related to the services found in the cluster. This way the client pods don't really need to remember the service IP address or port

```bash
/ # env
KUBERNETES_SERVICE_PORT=443
KUBERNETES_PORT=tcp://10.96.0.1:443
HOSTNAME=busybox
SHLVL=1
HOME=/root
HTTP_SERVICE_SERVICE_HOST=10.109.218.68
TERM=xterm
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
HTTP_SERVICE_SERVICE_PORT=80
HTTP_SERVICE_PORT=tcp://10.109.218.68:80
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_PROTO=tcp
HTTP_SERVICE_PORT_80_TCP_ADDR=10.109.218.68
HTTP_SERVICE_PORT_80_TCP_PORT=80
HTTP_SERVICE_PORT_80_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_SERVICE_HOST=10.96.0.1
PWD=/
HTTP_SERVICE_PORT_80_TCP=tcp://10.109.218.68:80
```

I will now try to see what kind of records are created inside CoreDNS for the service I just created.

There is no such cli provided by coredns that I can list the records in dns. But when I check one of the pods `/etc/resolve.conf` file it looks like that the DNS server IP is in the same subnet where the ClusterIP service is running

```bash
$ k exec my-dep-8954bc845-2rnxv -- cat /etc/resolv.conf
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5

$ k get svc
NAME           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
http-service   ClusterIP   10.109.218.68   <none>        80/TCP    2d
kubernetes     ClusterIP   10.96.0.1       <none>        443/TCP   8d
```

In summary, ClusterIP type of service is used in communication within the cluster. For example a client pod want to reach the server pod, It can actually hit ClusterIP without knowing actual pods that are backing up the service.

If the pods are in same namespace they can reach the service by its name.

For example if the service name is `http-service`, I can use `curl http-service` instead of writing its IP address. The `/etc/resolv.conf` file wil resolve the IP address of this service.

If the client pod is not in the same namespace where the server pods are located, then the DNS name would be `http-servce.<target-namespace>` or `http-service.<target-namespace>.svc.cluster.local`


