Kuberenetes shipped with various controllers. They are bundled together in a controlplae component call `kube-controller-manager`.The job of controller is to make the actual state of cluster nearer to the desired state.

Because kubernetes cluster has many API objects, It was not possible to make a single controller that could operatoe on all the resources. Hence they are splitted into different codebase but overall managed by `kube-controller-manager`.

For example if we want to create a `Job` in kubernetes, We will define the `Job` in the manfiest file and apply it via `kubectl`. The job controller will see this and it will understand that it has to make sure that somewhere in the cluster this job has to be run and has to be completed successfully. The job controller will not run the pod itselt. It will tell the API Server to create or remove the Pod on which the job will run. To demonstrate this I am creating a Job as mentioned in `job-manifest.yaml` file.

When I apply this manfiest file, I did a watch on `k get pods` and I noticed that the pod becomes `Ready` and then `Completed`. Also running `k get jobs` gives me result 
```bash
$ k get jobs
NAME   COMPLETIONS   DURATION   AGE
pi     1/1           48s        56s
```

This means that a creation of Job results in a creation of Pod. I want to verify things to get better understanding of what happened behind the scene. So I check logs of the `kube-controller-manager` because this component is the one who handles `controller` as a wrapper.

The logs showd me that indeed the `Job Controller` came into action

```bash
k logs -n kube-system kube-controller-manager-minikube

...
I0605 12:39:12.804651       1 job_controller.go:514] enqueueing job default/pi
I0605 12:39:13.809220       1 job_controller.go:514] enqueueing job default/pi
I0605 12:39:14.858747       1 job_controller.go:514] enqueueing job default/pi
I0605 12:39:15.869140       1 job_controller.go:514] enqueueing job default/pi
I0605 12:39:15.876606       1 job_controller.go:514] enqueueing job default/pi
I0605 12:39:15.883507       1 job_controller.go:514] enqueueing job default/pi
I0605 12:39:15.883605       1 event.go:294] "Event occurred" object="default/pi" fieldPath="" kind="Job" apiVersion="batch/v1" type="Normal" reason="Completed" message="Job completed"
```

But I am still confused, How can I check who created the Pod on which the Job ran. I will probably come back to this with some evidence / logs but in theory the following happends

- Running kubectl apply or any other command triggers the Kubernetes API client, which communicates with the kube-apiserver.
- The kube-apiserver receives the API call and validates it, ensuring the request is properly authenticated and authorized.
- Once the validation is successful, the kube-apiserver persists the pod definition in etcd, which is a distributed key-value store used as Kubernetes' backing store.
- The kube-scheduler observes the state of the cluster and periodically checks etcd for newly created pods that are in the "Pending" state.
- When the kube-scheduler identifies a pending pod, it selects a suitable node for scheduling based on various factors such as resource availability, pod affinity/anti-affinity rules, and other policies.
- Once the kube-scheduler determines the appropriate node, it updates the pod's assigned node information in etcd.
- The kubelet, running on each worker node, continuously monitors the pod assignments from etcd.
- When the kubelet detects a new pod assignment, it communicates with the Container Runtime Interface (CRI) of the node (e.g., Docker, containerd) to start the pod.
- The CRI interacts with the container runtime to create the pod's containers, set up namespaces, networking, and other necessary resources.