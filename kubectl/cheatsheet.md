- Count number of pods `kubectl get pods --no-headers | wc -l`
- Count number of containers in a pod `kubectl get pods webapp -o jsonpath='{.spec.containers[*].name}'`
- Scale replicaset `kubectl scale --replicas=4 rs/replica-set-name`
- Create Deployment `kubectl create deployment deployment-name --image=image-name`
- Update image on the deployment `kubectl set image deployment/deployment-name containername=image-name`
- Get pods with label `kubectl get pods -l key=value`
- Taint a node for NoSchdeule `kuebctl taint node nodename key=value:NoSchedule`
- To get all nodes with label `kubectl get nodes -l nodetype=worker`
- To get top on node / see cpu and memory usage  `kubectl top nodes`
- To see top on pods / see cpu and memory suage `kubectl top pods`
