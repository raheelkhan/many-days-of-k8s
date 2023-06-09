# Kube Proxy
The kube proxy runs as a DaemonSet within the kube-system namespace. I cross checked it with running 
`k get all -A` and I found that it is indeed preset as 

```bash
NAMESPACE     NAME                        DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
kube-system   daemonset.apps/kube-proxy   3         3         3       3            3           kubernetes.io/os=linux   10d
```
It creates 3 pods because I have 3 nodes cluster. I verified that indeed their is one pod in each node as dictated by the DaemonSet

```bash
$ k get pods -n kube-system
...
kube-proxy-cfhn7                   1/1     Running   6 (59m ago)    10d   192.168.49.3   minikube-m02   <none>           <none>
kube-proxy-tlpxl                   1/1     Running   4 (59m ago)    10d   192.168.49.4   minikube-m03   <none>           <none>
kube-proxy-xgm7s                   1/1     Running   6 (61m ago)    10d   192.168.49.2   minikube       <none>           <none>
```

My initial understanding was that pod to pod communication is faciliated by kube-proxy. For this purpose I created two pods via `two-pods-connectivity.yaml` file. Luckily they got schedules on different nodes. 

I noticed that that the ip address of pods were of same subnet

```bash
$ k get pods -o wide
NAME    READY   STATUS    RESTARTS   AGE   IP           NODE           NOMINATED NODE   READINESS GATES
pod-1   1/1     Running   0          10m   10.244.1.2   minikube-m02   <none>           <none>
pod-2   1/1     Running   0          10m   10.244.2.2   minikube-m03   <none>           <none>
```

I then exec into pod-1 and ping pod-2. It was working. Then I disabled the kube-proxy daemon set temporarily, But the ping was still working. This means that we can do pod to pod communication even if the kube-proxy pod is not running in the pod.

I just learned that the real work of kube proxy component comes in when we work with Kuberenetees Services.

The reason that we need Services is because in Kubernetes, Pods are ephermal. This means that at any point in time a pod can get down and if a pod is part of a Deployment or ReplicaSet, a new Pod will spin up having a different IP address. This means if I have a client pod that needs to talk to a server pod, I must have to maintain the IP address of those pods when they come and down and make changes to the appliation in my client pod so that it gets a new IP address of the server pod. This is not practical in real world.

To solve this issue, Kubernetes comes with another resource called Services. I will learn deeper into services in another day but for the sake of understanding kube proxy, It makes sense to touch a bit into Services.

When we create a Service it acts as a frontend for the backing pods. The services selects pods based on selector that match labels on the pod. There are multipe types of service but for now lets only focus on ClusterIP which is the default servic type if we do not specify the type in the service manifest.

For better explaination, I created a client and server pods. The server pods are backing a kubernetes services. These are created from file `kube-proxy.yaml`

```bash
$ k get all -o wide
NAME               READY   STATUS    RESTARTS   AGE   IP           NODE           NOMINATED NODE   READINESS GATES
pod/pod-client     1/1     Running   0          25s   10.244.1.5   minikube-m02   <none>           <none>
pod/server-pod-1   1/1     Running   0          25s   10.244.1.4   minikube-m02   <none>           <none>
pod/server-pod-2   1/1     Running   0          25s   10.244.2.5   minikube-m03   <none>           <none>

NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE   SELECTOR
service/kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP    10d   <none>
service/service      ClusterIP   10.105.56.21   <none>        8080/TCP   25s   app.kubernetes.io/name=server
```

I am just confused about how the IP CIDR for pods are created. But may be I will skip this for now and focus on kube-proxy.

The ClusterIP service/service as a IP address which is quite different from pods. I know that Kubernetes assigns IP address to services from a pool of addresses it stores somewhere. For somereason this IP address is called Virtual IP. 

So how does the service know where to forward request. First I will try to do a curl request from my client pod to the service ip `10.105.56.21` on port `8080` and see If I get the nginx page back as both my server pods are based on nginx containers.

I logged in to my client pod and indeed I can get the nginx page through the service IP.
```bash
$ k exec --stdin --tty pod-client -- /bin/sh
/ # wget 10.105.56.21:8080
Connecting to 10.105.56.21:8080 (10.105.56.21:8080)
saving to 'index.html'
index.html           100% |******************************************************************************************************************************************************************|   615  0:00:00 ETA
'index.html' saved
```

I will try now to disable the kube-proxy component in both worker nodes and see if I can still perform the above operation.

```bash
$ minikube ssh -n minikube-m02
docker stop <container id where kube-proxy is running>
docker@minikube-m02:~$ ps aux | grep proxy
docker@minikube-m02:~$ sudo kill -9 25052
```

I noticed that the kube proxy process keep comming up again on the worker node. To resort this, I deleted the cluster.

This time I will delete the kube-proxy pods before applying the `kube-proxy.yaml` manifest so to check the behaviour of service and pod reachibility if kube-proxy is non existent.

```bash
$ k delete daemonset.apps/kube-proxy -n kube-system
$ k apply -f kube-proxy.yaml
$ k get all -o wide
NAME               READY   STATUS    RESTARTS   AGE   IP           NODE           NOMINATED NODE   READINESS GATES
pod/pod-client     1/1     Running   0          23s   10.244.2.3   minikube-m03   <none>           <none>
pod/server-pod-1   1/1     Running   0          23s   10.244.1.2   minikube-m02   <none>           <none>
pod/server-pod-2   1/1     Running   0          23s   10.244.2.2   minikube-m03   <none>           <none>

NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE     SELECTOR
service/kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP    4m30s   <none>
service/service      ClusterIP   10.103.241.242   <none>        8080/TCP   23s     app.kubernetes.io/name=server

$ k exec --stdin --tty pod-client -- /bin/sh
/ # wget 10.103.241.242:8080
Connecting to 10.103.241.242:8080 (10.103.241.242:8080)

/ ping 10.244.1.2
64 bytes from 10.244.1.2: seq=0 ttl=62 time=0.181 ms
```

The above expirement proved that kube proxy is the one whom the Service needs in order to direct the traffic to backing pod. 

Kube proxy component on each node listen to the control plane and whenever there is a service and endpoints gets created, the kube proxy gets notified. It then writes some rules on the node it is running on that intercept packets for that service and change the destination address to one of the pod.

This is very vague description but in order to fully understand the kube proxy I read this article https://medium.com/@danielepolencic/learn-why-you-cant-ping-a-kubernetes-service-dec88b55e1a3