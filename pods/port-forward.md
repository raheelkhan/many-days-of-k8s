What if there is an application running on a pod but that pod does not belong to any service.

For example I created a pod for nginx 
```bash
$ k run nginx --image=nginx
pod/nginx created

$ k get pods -o wide
NAME    READY   STATUS    RESTARTS   AGE   IP           NODE           NOMINATED NODE   READINESS GATES
nginx   1/1     Running   0          33s   10.244.1.2   minikube-m02   <none>           <none>
```

Now How can I access this pod from my host machine.

```bash
$ curl http://10.244.1.2:80

```
I am not able to reach the pod. Here I chose 80 because nginx by default runs on port 80.

Lets try doing a port forward.
```bash
$ k port-forward nginx 8080:80
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80

curl http://localhost:8080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

After doing port forward I am able to access the container port 80 from my host port 8080.

I am curious to know what will happen if there are two containers running within a pod. First is what will happen if I have two nginx containers within a pod. Are they allowed to run in a single pod while we know that both listen to port 80 by default.

For this I created a pod manifest `two-nginx-pods.yaml`

```bash
$ k apply -f two-nginx-pods.yaml
pod/two-nginx created

$ k get pods --watch
NAME        READY   STATUS              RESTARTS   AGE
two-nginx   0/2     ContainerCreating   0          3s
two-nginx   2/2     Running             0          7s
two-nginx   1/2     Error               0          10s
two-nginx   2/2     Running             1 (4s ago)   13s
two-nginx   1/2     Error               1 (7s ago)   16s
two-nginx   1/2     CrashLoopBackOff    1 (16s ago)   31s
two-nginx   2/2     Running             2 (21s ago)   36s
two-nginx   1/2     Error               2 (23s ago)   38s
```

I noticed that its not working. It was kind of expected but lets find the explanation of why it is happening.

My first approach was to see `k logs two-nginx` but it did not gave any usefull information except that it mentioned about default container selection

```bash
$ k logs two-nginx
Defaulted container "container-1" out of: container-1, container-2
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: info: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
2023/06/07 11:00:09 [notice] 1#1: using the "epoll" event method
2023/06/07 11:00:09 [notice] 1#1: nginx/1.25.0
2023/06/07 11:00:09 [notice] 1#1: built by gcc 10.2.1 20210110 (Debian 10.2.1-6)
2023/06/07 11:00:09 [notice] 1#1: OS: Linux 5.19.0-43-generic
2023/06/07 11:00:09 [notice] 1#1: getrlimit(RLIMIT_NOFILE): 1048576:1048576
2023/06/07 11:00:09 [notice] 1#1: start worker processes
2023/06/07 11:00:09 [notice] 1#1: start worker process 29
2023/06/07 11:00:09 [notice] 1#1: start worker process 30
2023/06/07 11:00:09 [notice] 1#1: start worker process 31
2023/06/07 11:00:09 [notice] 1#1: start worker process 32
```

Then i looked into kubernetes events
```bash
$ k events
LAST SEEN               TYPE      REASON      OBJECT          MESSAGE
13m                     Normal    Scheduled   Pod/nginx       Successfully assigned default/nginx to minikube-m02
13m                     Normal    Pulling     Pod/nginx       Pulling image "nginx"
13m                     Normal    Pulled      Pod/nginx       Successfully pulled image "nginx" in 2.005787965s (2.005796917s including waiting)
13m                     Normal    Created     Pod/nginx       Created container nginx
13m                     Normal    Started     Pod/nginx       Started container nginx
5m46s                   Normal    Killing     Pod/nginx       Stopping container nginx
5m34s                   Normal    Pulling     Pod/two-nginx   Pulling image "nginx"
5m34s                   Normal    Scheduled   Pod/two-nginx   Successfully assigned default/two-nginx to minikube-m03
5m32s                   Normal    Pulled      Pod/two-nginx   Successfully pulled image "nginx" in 2.186029616s (2.186039756s including waiting)
5m32s                   Normal    Created     Pod/two-nginx   Created container container-1
5m32s                   Normal    Started     Pod/two-nginx   Started container container-1
5m27s                   Normal    Pulled      Pod/two-nginx   Successfully pulled image "nginx" in 4.33392025s (4.333936985s including waiting)
5m22s                   Normal    Pulled      Pod/two-nginx   Successfully pulled image "nginx" in 2.416767506s (2.416794613s including waiting)
4m59s                   Normal    Pulled      Pod/two-nginx   Successfully pulled image "nginx" in 4.797217723s (4.797228886s including waiting)
4m26s (x4 over 5m32s)   Normal    Pulling     Pod/two-nginx   Pulling image "nginx"
4m24s (x4 over 5m27s)   Normal    Created     Pod/two-nginx   Created container container-2
4m24s (x4 over 5m27s)   Normal    Started     Pod/two-nginx   Started container container-2
4m24s                   Normal    Pulled      Pod/two-nginx   Successfully pulled image "nginx" in 2.404481857s (2.404499683s including waiting)
24s (x21 over 5m18s)    Warning   BackOff     Pod/two-nginx   Back-off restarting failed container container-2 in pod two-nginx_default(3f24bc99-022a-437e-86a5-33b6581eea55)
```

Here I noticed that the container-2 bas BackOff reason.

I also noticed that it has been restarted 6 times.
```bash
$ k get pod two-nginx --watch
NAME        READY   STATUS             RESTARTS       AGE
two-nginx   1/2     CrashLoopBackOff   6 (4m4s ago)   10m
```

I also see the describe command for pod. It shows that the container is indeed in crashing state 
```bash
$ k describe pod two-nginx
    State:          Waiting
      Reason:       CrashLoopBackOff
    Last State:     Terminated
      Reason:       Error
      Exit Code:    1
      Started:      Wed, 07 Jun 2023 15:11:44 +0400
      Finished:     Wed, 07 Jun 2023 15:11:47 +0400
    Ready:          False
```

But no where it mentioned the reason. Lets try to find more options to see the error.

I figured out that I can actually see the logs of specific container wihtin pod as 
```bash
$ k logs two-nginx -c container-2
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: info: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
2023/06/07 11:11:44 [emerg] 1#1: bind() to 0.0.0.0:80 failed (98: Address already in use)
nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)
2023/06/07 11:11:44 [emerg] 1#1: bind() to [::]:80 failed (98: Address already in use)
...
```

This made sense now that the container-2 is also attempting to listen to port 80 whereas the container-1 is already up listening to port 80 within the same pod.

We know that the containers within a pod shares the same network namespace. Lets do a little experiment to prove this fact.

I've created another file `two-pods-different-ports.yaml` for this purpose.

```bash
$ k apply -f two-pods-different-ports.yaml
pod/nginx-busybox created

$ k get pods  -o wide
NAME            READY   STATUS    RESTARTS   AGE    IP           NODE           NOMINATED NODE   READINESS GATES
nginx-busybox   2/2     Running   0          100s   10.244.2.3   minikube-m03   <none>           <none>
```

I will now exec into the busybox container and try to access nginx

```bash
$k exec -it -c busybox nginx-busybox -- /bin/sh
#/ wget localhost -O - 2>/dev/null
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
```

I can see nginx page by accessing `localhost`. This made sure that the loopback interface of busybox container is same as nginx.

Lets now see the namespace this container is attached to

```bash
#/ netstat -tnlp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      -
tcp        0      0 :::80                   :::*                    LISTEN      -
```

I see that there is something running on port 80 even though I am in busybox container. But I don't see what is running. This is because the containers in pod shares network namesapce and not process namespace.

Lets dig more and find the network namespace name.

I take the conatinerIDs from the pod
```bash
$ k get pods -o jsonpath='{range $.items[*].status.containerStatuses[*]}{.containerID}{"\n"}{end}'
docker://361fa0f9229d9d62df14cccac1faadbb47fd244153081dae395418bc11143fbe
docker://759228dd81e13061f0e6b2b84cb94360f101a52d67641ffa74700d1689d93713
```

```bash
$ k get pods -o wide
NAME            READY   STATUS    RESTARTS   AGE   IP           NODE           NOMINATED NODE   READINESS GATES
nginx-busybox   2/2     Running   0          63m   10.244.2.3   minikube-m03   <none>           <none>
```

Now in order to inspect containers I will SSH into the host where this pod is running. I am then taking the Process ID for both containers in the minikube-m03 host. Once I get the process ID. I will run `lsns <Pid>` command to see if both containers (process) shares the same network namespace.

```bash
minikube ssh -n minikube-m03
docker inspect 361fa0f9229d9d62df14cccac1faadbb47fd244153081dae395418bc11143fbe | jq '.[].State.Pid'
104414

docker inspect 759228dd81e13061f0e6b2b84cb94360f101a52d67641ffa74700d1689d93713 | jq '.[].State.Pid'

sudo lsns -p 104414
4026532860 net         7 104297 65535 /pause

sudo lsns -p 104460
4026532860 net         7 104297 65535 /pause
```

As we can see the netns is same for both containers of this pod hence it make sense why when we put 2 nginx containers in a single pod with both listening on default port 8 were not allowed to run.