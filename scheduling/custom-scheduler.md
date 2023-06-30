# Configure a custom scheduler

We can have multiple kube-scheduler in a kubernetes cluster. We can specify for certain pods which scheulder we cant to use.

Today I am following documentation for configuring a custom scheduler using kubernetes.io page https://kubernetes.io/docs/tasks/extend-kubernetes/configure-multiple-schedulers/

First I want to see how can I get official image for `kube-scheduler`. The docs suggested it to make a copy of original kube scheulder in a custom docker image. But here I want to avoid it and use the original image itself. I will simply name my custom scheduler something else to differentiate.

Because I am using kubeadm for my local cluster and I already knew that kubedm configure many cluster components in Pods. I checked in the directory if kube-scheduler is also configrued to run as a pod and luckily I found its manfiest file in 

```bash
vagrant ssh kuebmaster
ls /etc/kubernetes/manifest

...other files kube-scheulder.yaml

sudo cat /etc/kubernetes/manifest/kube-scheduler.yaml

...
image: registry.k8s.io/kube-scheduler:v1.27.3
```

Okay I've got the official kube-scheulder image. I will use this one for the rest of my learning.

As the docs suggested I will run the scheduler as a Pod. I have created a file `my-scheduler.yaml` and copied the contents from the docs

I just recalled, I can use :set paste command on Vim to paste the content properly form docs. I am then modifying few things to match my cluster in the above file specially the `image` and node labels

So far, I understood that the kube-schduler needs a configuration. This configuration is actually created as a kubernetes resource with name `KubeSchedulerConfiguration`. Because I am creating the my-schduler as a Pod, In this file it is done by creating a ConfigMap where the kye of file is named as `my-scheulder-config.yaml` and the data is actually the YAML manifest of resource `KubeSchedulerConfiguration`.

I also noticed that there is a service account created with the name my-scheduler and it is being used in the Pod definition of kube-scheulder. The Pod is actually deployed via Deployment and the reason is that we want to make the Pod always available. The Deployment will handle the Pod crash and restart a new container if it fails.

I also learned that this SerivceAccount needs some permission. The yaml file basically creates a `ClusterRoleBinding` where it bind this `ServiceAccount` to a role name `system:kube-schduler`. I believe this role is created by Kubeadm itself. But for my purpose this is enough to use the same role.

There are other RoleBinding also configured. For now I am not going into the details for this. I just need to pass CKA.

I ran the file 

```bash
kuebctl apply -f my-scheduler.yaml

kubectl get pods -A

$ k get pods -n kube-system -o wide | grep my-scheduler
my-scheduler-77d8c67b7d-d694t        1/1     Running   0             2m32s   10.244.192.1   kubenode01   <none>           <none>
```

This is a bit confusing for me. I was expecting that the pod will be scheduled on the kubemaster node. But nevermind, I will try to run the pod defined in `my-scheduler-pod.yaml` file to see if the scheduling is working or not. I expect the Pod to be in running state.If if is PENDING, this means the scheduler is not working. I will make sure to use `spec.schedulerName` in the pod manifest.

I can see the pod is in Running state. Lets see if I can verify who scheduled it.

```bash
k get pods 

NAME               READY   STATUS    RESTARTS   AGE
my-scheduler-pod   1/1     Running   0          17s

Events:
  Type    Reason     Age   From          Message
  ----    ------     ----  ----          -------
  Normal  Scheduled  91s   my-scheduler  Successfully assigned default/my-scheduler-pod to kubenode02
  Normal  Pulling    90s   kubelet       Pulling image "nginx"
  Normal  Pulled     79s   kubelet       Successfully pulled image "nginx" in 11.568495985s (11.568502331s including waiting)
  Normal  Created    79s   kubelet       Created container my-scheduler-pod
  Normal  Started    78s   kubelet       Started container my-scheduler-pod
```  

Nice, The Events tab actually showed that is actually scheduled from `my-scheduler`
