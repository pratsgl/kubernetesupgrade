# Swithcing Container runtime from Docker to Containerd on Kubernetes 1.20

The following document runs you through how to change the container runtime (From Docker to Containerd) on your existing kubernetes cluster with workloads running.

We have a following Kubernetes cluster 1.20 running Docker as CONTAINER-RUNTIME
```
user@lab-system:~/kubernetes$ kubectl get nodes -o wide
NAME                 STATUS   ROLES                  AGE   VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION          CONTAINER-RUNTIME
kmaster.mylab.com    Ready    control-plane,master   43m   v1.20.2   172.42.42.100   <none>        CentOS Linux 7 (Core)   3.10.0-1127.el7.x86_64   docker://20.10.2
kworker1.mylab.com   Ready    <none>                 35m   v1.20.2   172.42.42.101   <none>        CentOS Linux 7 (Core)   3.10.0-1127.el7.x86_64   docker://20.10.2
kworker2.mylab.com   Ready    <none>                 28m   v1.20.2   172.42.42.102   <none>        CentOS Linux 7 (Core)   3.10.0-1127.el7.x86_64   docker://20.10.2
``` 
Lets create a deployment with nginx & scale it to 4 replicas

```
user@lab-system:~/kubernetes$ kubectl create deploy nginx --image nginx
deployment.apps/nginx created

user@lab-system:~/kubernetes$ kubectl get pods -o wide
NAME                     READY   STATUS    RESTARTS   AGE     IP               NODE                 NOMINATED NODE   READINESS GATES
nginx-6799fc88d8-wvhbc   1/1     Running   0          5m39s   192.168.28.193   kworker2.mylab.com   <none>           <none>

user@lab-system:~/kubernetes$ kubectl scale deploy nginx --replicas 4
deployment.apps/nginx scaled

user@lab-system:~/kubernetes$ kubectl get pods -o wide
NAME                     READY   STATUS    RESTARTS   AGE     IP               NODE                 NOMINATED NODE   READINESS GATES
nginx-6799fc88d8-4pc2d   1/1     Running   0          2m13s   192.168.94.1     kworker1.mylab.com   <none>           <none>
nginx-6799fc88d8-fckcq   1/1     Running   0          2m13s   192.168.28.194   kworker2.mylab.com   <none>           <none>
nginx-6799fc88d8-kqmrw   1/1     Running   0          2m13s   192.168.94.2     kworker1.mylab.com   <none>           <none>
nginx-6799fc88d8-wvhbc   1/1     Running   0          5m39s   192.168.28.193   kworker2.mylab.com   <none>           <none>
```
## On kworker2 node , lets change Container runtime from Docker to Containerd 
Let's cordon kworker2 node , so no pods get scheduled on that host 

```
user@lab-system:~/kubernetes$ kubectl cordon kworker2.mylab.com
node/kworker2.mylab.com cordoned

user@lab-system:~/kubernetes$ kubectl get nodes 
NAME                 STATUS                     ROLES                  AGE   VERSION
kmaster.mylab.com    Ready                      control-plane,master   84m   v1.20.2
kworker1.mylab.com   Ready                      <none>                 76m   v1.20.2
kworker2.mylab.com   Ready,SchedulingDisabled   <none>                 69m   v1.20.2
```
Now let's drain kworker2 node , so pods get evicted on kworker2 node and gets scheduled on kworker1

```
user@lab-system:~/kubernetes$ kubectl drain kworker2.mylab.com --ignore-daemonsets
node/kworker2.mylab.com already cordoned
WARNING: ignoring DaemonSet-managed Pods: kube-system/calico-node-28wpk, kube-system/kube-proxy-scfvt
evicting pod default/nginx-6799fc88d8-fckcq
evicting pod default/nginx-6799fc88d8-wvhbc
pod/nginx-6799fc88d8-fckcq evicted
pod/nginx-6799fc88d8-wvhbc evicted
node/kworker2.mylab.com evicted
```
We can see all the pods are now running on kworker1 node
```
[vagrant@kmaster ~]$ kubectl get pods -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP             NODE                 NOMINATED NODE   READINESS GATES
nginx-6799fc88d8-4pc2d   1/1     Running   0          14m   192.168.94.1   kworker1.mylab.com   <none>           <none>
nginx-6799fc88d8-7gcbh   1/1     Running   0          81s   192.168.94.4   kworker1.mylab.com   <none>           <none>
nginx-6799fc88d8-kqmrw   1/1     Running   0          14m   192.168.94.2   kworker1.mylab.com   <none>           <none>
nginx-6799fc88d8-wtwrz   1/1     Running   0          81s   192.168.94.3   kworker1.mylab.com   <none>           <none>
```


## On kworker1 node , lets change Container runtime from Docker to Containerd 
## On kmaster node , lets change Container runtime from Docker to Containerd 
