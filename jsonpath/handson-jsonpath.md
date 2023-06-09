I am trying to understand and practie jsonpath to be able to parse results from kubectl

I will do practice based on the `sample.json` file that I have obtained from command `kubectl get all -n kube-system -o json > sample.json`

Lets say I want to access keys on the root of the object 

```bash
k get all -n kube-system -o jsonpath='{$.apiVersion}'
v1%
```
I want to access `resourceVersion` that is nested inside `metadata`
```bash
k get all -n kube-system -o jsonpath='{$.metadata.resourceVersion}'
```

I want to get name of all pods 
```bash
k get pods -n kube-system -o jsonpath='{$.items[*].metadata.name}'
coredns-787d4945fb-4m275 etcd-minikube kindnet-cdcf4 kindnet-fhnfv kindnet-gpbhv kube-apiserver-minikube kube-controller-manager-minikube kube-proxy-7tptj kube-proxy-djdvf kube-proxy-jxkxz kube-scheduler-minikube storage-provisioner%
```

What if instead of plain string I want to get and array
```bash
kubectl get pods -n kube-system -o jsonpath='{range .items[*]}{.metadata.name}{end}'
```


I want to list all pods with their status and ip 
```bash
kubectl get pods -n kube-system -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\t"}{.status.podIP}{"\n"}{end}'

coredns-787d4945fb-4m275	Running	10.244.0.2
etcd-minikube	Running	192.168.49.2
kindnet-cdcf4	Running	192.168.49.4
kindnet-fhnfv	Running	192.168.49.3
kindnet-gpbhv	Running	192.168.49.2
kube-apiserver-minikube	Running	192.168.49.2
kube-controller-manager-minikube	Running	192.168.49.2
kube-proxy-7tptj	Running	192.168.49.3
kube-proxy-djdvf	Running	192.168.49.4
kube-proxy-jxkxz	Running	192.168.49.2
kube-scheduler-minikube	Running	192.168.49.2
storage-provisioner	Running	192.168.49.2
```

I want to see only pods with ip and status who belongs to a DaemonSet
```bash
k get pods -n kube-system -o jsonpath='{range .items[?(@.metadata.ownerReferences[*].kind=="DaemonSet")]}{.metadata.name}{"\t"}{.status.phase}{"\t"}{.status.podIP}{"\n"}{end}'

kindnet-cdcf4	Running	192.168.49.4
kindnet-fhnfv	Running	192.168.49.3
kindnet-gpbhv	Running	192.168.49.2
kube-proxy-7tptj	Running	192.168.49.3
kube-proxy-djdvf	Running	192.168.49.4
kube-proxy-jxkxz	Running	192.168.49.2
```

I think this is enought as I've read that it is not really a mandatory requirement for CKA. It is just a tool