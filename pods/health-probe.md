# Livness probe

Liveness probes are used to check containers health.

It is worth noting that the `livenessProbe` is configured on per container basis. If we have two containers in a single pod and we want to check if both our containers are working fine, we have to configure it on both containers.

Lets discuss some parameters in detail.

- `httpGet` This tells that the we are checking health based on HTTP GET method.
- `initialDelaySeconds` This means the kubernetes will hold this seconds after containers comes in `Created` state.
- `timeoutSeconds` This tells Kubnetes that it should expect the response of health check within this many seconds.
- `periodSeconds` This tells the time period on which kubernetes wil perform the health check. For example if it is set to `10`, the health check call will be made every `10` seconds.
- `failureThreshold` This tells how many times the health check should fail before Kubernetes restart the container.

I created a sample file `liveness-probe-pod.yaml` file to test the behaviour. There is a pod with 2 nginx containers. Both have `livenessProbe` configured but the 2nd container has a wrong path `/notfound` configured for health check. The watch command shows the following events

```bash
$ k get pods --watch
NAME                 READY   STATUS    RESTARTS   AGE
liveness-probe-pod   2/2     Running   0          7s
liveness-probe-pod   1/2     Error     0          9s
liveness-probe-pod   2/2     Running   1 (4s ago)   12s
liveness-probe-pod   1/2     Error     1 (6s ago)   14s
liveness-probe-pod   1/2     CrashLoopBackOff   1 (8s ago)   21s
liveness-probe-pod   2/2     Running            2 (24s ago)   37s
liveness-probe-pod   1/2     Error              2 (27s ago)   40s
liveness-probe-pod   1/2     CrashLoopBackOff   2 (2s ago)    41s
```

Then I check the pod describe command and it indicates that it is failing to start. But I could not figure out any logs related to health checks. Then I realized that I am using nginx with default configuration on which it listens to 80. Again it is not possible for two containers using same port within a Pod

So I learned that we can pass a different nginx configuration to the second pod and tell it to use port `81` instead. This way both container will atleast start properly so tha the `livenessProbe` can actually work.

So I have created a config file for nginx and we will try to mount it inside the container-2. For this demo I will use volume of `hostPath` as we don't want the volume to outlive the life of pod.

I will modify the `liveness-probe-pod.yaml` now to reflect the volume.

There are few learnings happend while I was trying to set up this demo. I will document it after coming from gym because I am tired. 

I have create a ConfigMap that contains nginx configuration. The manifest of config map is also present in the `liveness-probe-pod.yaml` file. Once I am able to run two nginx containers in the same pod. I noticed the expected behaviour.

Both containers got into Running state at first but then one container start to fail and restart. 

```bash
$ k get pods --watch
NAME                 READY   STATUS    RESTARTS      AGE
liveness-probe-pod   2/2     Running   3 (27s ago)   2m7s
liveness-probe-pod   2/2     Running   4 (4s ago)    2m14s
liveness-probe-pod   1/2     CrashLoopBackOff   4 (1s ago)    2m41s
liveness-probe-pod   2/2     Running            5 (53s ago)   3m33s
```

`CrashLoopBackOff` is the status when kubernetes attempted to restart the failing container.

I was able to see the Liveness probe log in the Events section of pod

```bash
Events:
  Type     Reason     Age                 From               Message
  ----     ------     ----                ----               -------
  Normal   Scheduled  111s                default-scheduler  Successfully assigned default/liveness-probe-pod to minikube-m03
  Normal   Pulling    110s                kubelet            Pulling image "nginx"
  Normal   Pulled     108s                kubelet            Successfully pulled image "nginx" in 2.378442401s (2.378453169s including waiting)
  Normal   Created    108s                kubelet            Created container container-1
  Normal   Started    108s                kubelet            Started container container-1
  Normal   Pulled     106s                kubelet            Successfully pulled image "nginx" in 2.393996145s (2.394004357s including waiting)
  Normal   Pulled     68s                 kubelet            Successfully pulled image "nginx" in 2.056759806s (2.056767718s including waiting)
  Normal   Killing    41s (x2 over 71s)   kubelet            Container container-2 failed liveness probe, will be restarted
  Normal   Pulling    40s (x3 over 108s)  kubelet            Pulling image "nginx"
  Normal   Created    38s (x3 over 106s)  kubelet            Created container container-2
  Normal   Started    38s (x3 over 105s)  kubelet            Started container container-2
  Normal   Pulled     38s                 kubelet            Successfully pulled image "nginx" in 2.520814946s (2.520829322s including waiting)
  Warning  Unhealthy  31s (x7 over 91s)   kubelet            Liveness probe failed: HTTP probe failed with statuscode: 404
```

Is there any way I can control what to do if the Liveness probe is failing. For this reason I am reading on `RestartPolicy`

We can provide a `restartPolicy` to pod that applies to all containers.

We cave have the following options
- `Always` This means if a container is a pod is exited / completed even with the exit code `0`, Kubernetes will restart it.
- `OnFailure` This means if a container in a pod is crashed, Kubernetes will restart it. But if the container is exited in successfull state, Kubernetes will not restart it.
- `Never` This means kubernetes will never restart the container.

I am now trying to set `restartPolicy` to `Never` and see what would happen to the container that fail `Liveness` probe.

```bash
$ k get pods --watch
NAME                 READY   STATUS              RESTARTS   AGE
liveness-probe-pod   0/2     ContainerCreating   0          5s
liveness-probe-pod   2/2     Running             0          6s
liveness-probe-podliveness-probe-pod   1/2     NotReady            0          31s
```

Noticed that the pod has `0` RESTARTS. Even though the liveness problem as failed.

```bash
 Warning  Unhealthy  5s (x2 over 15s)  kubelet            Liveness probe failed: HTTP probe failed with statuscode: 404
```

Some important points about `LivenessProbe`, `ReadinessProbe` and `StartupProbe`

If our application takes long time to initialized when the pod is launched. We have know idea about the time we can set for `initialDelaySeconds`. In this case we must use `startupProbe` and give enough time based on facts. For example if we know that the app initilization takes `300` seconds. then we must configure `startupProbe` with `initialDelaySeconds` of `300`. 

This will make sure that the `liveness` and `readiness` probel do not kill the pod if the pod is still in its initialization phase.

When it comes to `readinessProbe` it behaves differently than `livenessProbe`. When a container fails `readinessProbe` the pod, is removed from the Service but not killed.
