# LIVE switching of Container runtime from Docker to Containerd on Kubernetes 1.20

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
On other terminal , lets login to kworker2 as root user & check docker installation 

```
[root@kworker2 ~]# rpm -qa | grep docker
docker-ce-20.10.2-3.el7.x86_64
docker-ce-cli-20.10.2-3.el7.x86_64
docker-ce-rootless-extras-20.10.2-3.el7.x86_64

root@kworker2 ~]# docker version
Client: Docker Engine - Community
 Version:           20.10.2
 API version:       1.41
 Go version:        go1.13.15
 Git commit:        2291f61
 Built:             Mon Dec 28 16:17:48 2020
 OS/Arch:           linux/amd64
 Context:           default
 Experimental:      true

Server: Docker Engine - Community
 Engine:
  Version:          20.10.2
  API version:      1.41 (minimum version 1.12)
  Go version:       go1.13.15
  Git commit:       8891c58
  Built:            Mon Dec 28 16:16:13 2020
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.4.3
  GitCommit:        269548fa27e0089a8b8278fc4fc781d7f65a939b
 runc:
  Version:          1.0.0-rc92
  GitCommit:        ff819c7e9184c13b7c2607fe6c30ae19403a7aff
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0


[root@kworker2 ~]# docker ps
CONTAINER ID   IMAGE                   COMMAND                  CREATED             STATUS             PORTS     NAMES
74dc28aa7f18   calico/node             "start_runit"            About an hour ago   Up About an hour             k8s_calico-node_calico-node-28wpk_kube-system_6c56c445-70cc-46f9-a00c-dfa42c9f3281_0
af56f3174b38   k8s.gcr.io/kube-proxy   "/usr/local/bin/kube…"   About an hour ago   Up About an hour             k8s_kube-proxy_kube-proxy-scfvt_kube-system_afe395f8-3684-4069-bbb0-b27c0b2314f2_0
5b555e6d825b   k8s.gcr.io/pause:3.2    "/pause"                 About an hour ago   Up About an hour             k8s_POD_calico-node-28wpk_kube-system_6c56c445-70cc-46f9-a00c-dfa42c9f3281_0
5a63d1680f69   k8s.gcr.io/pause:3.2    "/pause"                 About an hour ago   Up About an hour             k8s_POD_kube-proxy-scfvt_kube-system_afe395f8-3684-4069-bbb0-b27c0b2314f2_0


[root@kworker2 ~]# ctr --namespace moby container list
CONTAINER                                                           IMAGE    RUNTIME                  
5a63d1680f69e6d02f46b8d8a5268431848441dbe74fa595cc7c8747fa2eb06e    -        io.containerd.runc.v2    
5b555e6d825bffc13bd968b2ececfaba3d985b4650441c77fb3a7b318f32aac7    -        io.containerd.runc.v2    
74dc28aa7f1872ce3fa9f8f0b27912fb00f680f3a5dcc24cf2f3d37a3204fb79    -        io.containerd.runc.v2    
af56f3174b38927633831db86971b56287ee308d2dbefea30cdb5c09a06e773a    -        io.containerd.runc.v2 

```
We need to stop docker & kubelet service 
```
[root@kworker2 ~]# systemctl stop docker
[root@kworker2 ~]# systemctl stop kubelet
```
Uninstall docker 
```
[root@kworker2 ~]# rpm -e docker-ce docker-ce-cli docker-ce-rootless-extras

[root@kworker2 ~]# rpm -qa | grep docker

[root@kworker2 ~]# ctr namespace list
NAME LABELS 
moby   
     
[root@kworker2 ~]# ctr --namespace moby container list
CONTAINER    IMAGE    RUNTIME
```
Check containerd & its configuration file
```
[root@kworker2 ~]# rpm -qa containerd.io
containerd.io-1.4.3-3.1.el7.x86_64

[root@kworker2 ~]# rpm -ql containerd.io | grep toml
/etc/containerd/config.toml
/usr/share/man/man5/containerd-config.toml.5
```
By default usage of "cri" is disabled , so we need to enable by editing following containerd config file
```
[root@kworker2 ~]# vi /etc/containerd/config.toml
from 
disabled_plugins = ["cri"]
to
#disabled_plugins = ["cri"]
```
To make it effective restart containerd service
```
[root@kworker2 ~]# systemctl restart containerd

[root@kworker2 ~]# systemctl status containerd
● containerd.service - containerd container runtime
   Loaded: loaded (/usr/lib/systemd/system/containerd.service; disabled; vendor preset: disabled)
   Active: active (running) since Fri 2021-01-29 06:35:41 UTC; 2min 1s ago
     Docs: https://containerd.io
  Process: 26469 ExecStartPre=/sbin/modprobe overlay (code=exited, status=0/SUCCESS)
 Main PID: 26471 (containerd)
    Tasks: 15
   Memory: 26.8M
   CGroup: /system.slice/containerd.service
           └─26471 /usr/bin/containerd

Jan 29 06:35:41 kworker2.mylab.com containerd[26471]: time="2021-01-29T06:35:41.128949812Z" level=info msg="loading plugin \"io.con...rpc.v1
Jan 29 06:35:41 kworker2.mylab.com containerd[26471]: time="2021-01-29T06:35:41.129423591Z" level=info msg=serving... address=/run/....ttrpc
Jan 29 06:35:41 kworker2.mylab.com containerd[26471]: time="2021-01-29T06:35:41.129518949Z" level=info msg=serving... address=/run/...d.sock
Jan 29 06:35:41 kworker2.mylab.com containerd[26471]: time="2021-01-29T06:35:41.129608746Z" level=info msg="containerd successfully...5628s"
Jan 29 06:35:41 kworker2.mylab.com containerd[26471]: time="2021-01-29T06:35:41.135395049Z" level=info msg="Start subscribing conta...event"
Jan 29 06:35:41 kworker2.mylab.com containerd[26471]: time="2021-01-29T06:35:41.135497250Z" level=info msg="Start recovering state"
Jan 29 06:35:41 kworker2.mylab.com containerd[26471]: time="2021-01-29T06:35:41.135783602Z" level=info msg="Start event monitor"
Jan 29 06:35:41 kworker2.mylab.com containerd[26471]: time="2021-01-29T06:35:41.135809409Z" level=info msg="Start snapshots syncer"
Jan 29 06:35:41 kworker2.mylab.com containerd[26471]: time="2021-01-29T06:35:41.135835808Z" level=info msg="Start cni network conf syncer"
Jan 29 06:35:41 kworker2.mylab.com containerd[26471]: time="2021-01-29T06:35:41.135845968Z" level=info msg="Start streaming server"
Hint: Some lines were ellipsized, use -l to show in full.
```
We need to now change "kubelet" configuration , edit following file and add 2 parameters container-runtime & container-runtime-endpoint
```
root@kworker2 ~]# vi /var/lib/kubelet/kubeadm-flags.env 

[root@kworker2 ~]# cat /var/lib/kubelet/kubeadm-flags.env 
from 
KUBELET_KUBEADM_ARGS="--network-plugin=cni --pod-infra-container-image=k8s.gcr.io/pause:3.2"
to
KUBELET_KUBEADM_ARGS="--network-plugin=cni --pod-infra-container-image=k8s.gcr.io/pause:3.2 --container-runtime=remote --container-runtime-endpoint=unix:///run/containerd/containerd.sock"
```
Now restart kubelet service
```
[root@kworker2 ~]# systemctl start kubelet

[root@kworker2 ~]# systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
   Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; vendor preset: disabled)
  Drop-In: /usr/lib/systemd/system/kubelet.service.d
           └─10-kubeadm.conf
   Active: active (running) since Fri 2021-01-29 06:42:41 UTC; 2s ago
     Docs: https://kubernetes.io/docs/
 Main PID: 26549 (kubelet)
    Tasks: 9
   Memory: 21.7M
   CGroup: /system.slice/kubelet.service
           └─26549 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.co...

Jan 29 06:42:41 kworker2.mylab.com systemd[1]: Started kubelet: The Kubernetes Node Agent.
Jan 29 06:42:41 kworker2.mylab.com kubelet[26549]: W0129 06:42:41.363814   26549 server.go:191] Warning: For remote container runti...nstead
Jan 29 06:42:41 kworker2.mylab.com kubelet[26549]: I0129 06:42:41.387538   26549 server.go:416] Version: v1.20.2
Jan 29 06:42:41 kworker2.mylab.com kubelet[26549]: I0129 06:42:41.388474   26549 server.go:837] Client rotation is on, will bootstr...ground
Jan 29 06:42:41 kworker2.mylab.com kubelet[26549]: I0129 06:42:41.403603   26549 certificate_store.go:130] Loading cert/key pair fr....pem".
Jan 29 06:42:41 kworker2.mylab.com kubelet[26549]: I0129 06:42:41.404779   26549 dynamic_cafile_content.go:167] Starting client-ca-...ca.crt
Hint: Some lines were ellipsized, use -l to show in full.
```

```
[root@kworker2 ~]# ctr namespace list
NAME   LABELS 
k8s.io        
moby 

[root@kworker2 ~]# ctr --namespace k8s.io container list
CONTAINER                                                           IMAGE                                    RUNTIME                  
2a468ecdb0cabaf44a8fe46e209c0fb1f0470a313da397fcca26cffd81b6226e    docker.io/calico/cni:v3.16.6             io.containerd.runc.v2    
4f9de04aec31e59ef1c83194560f09eb8abdf87e309a1cb549bbeedab540f2fd    docker.io/calico/node:v3.16.6            io.containerd.runc.v2    
60903d1a7c02216b0cebebe4173ff248159c3ae28212471f4b6ed0e54748ce63    k8s.gcr.io/pause:3.2                     io.containerd.runc.v2    
66400c2313408443a280a205e06a91264c652fb475b56484c5a7a0b0a95349c2    k8s.gcr.io/pause:3.2                     io.containerd.runc.v2  
```
Now you can see STATUS from NotReady to Ready & CONTAINER-RUNTIME now for kworker2 is "containerd://1.4.3" (scroll to right >> )
```
user@lab-system:~/kubernetes$ kubectl get nodes  -o wide
NAME                 STATUS                     ROLES                  AGE    VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION           CONTAINER-RUNTIME
kmaster.mylab.com    Ready                      control-plane,master   128m   v1.20.2   172.42.42.100   <none>        CentOS Linux 7 (Core)   3.10.0-1127.el7.x86_64   docker://20.10.2
kworker1.mylab.com   Ready                      <none>                 119m   v1.20.2   172.42.42.101   <none>        CentOS Linux 7 (Core)   3.10.0-1127.el7.x86_64   docker://20.10.2
kworker2.mylab.com   Ready,SchedulingDisabled   <none>                 113m   v1.20.2   172.42.42.102   <none>        CentOS Linux 7 (Core)   3.10.0-1127.el7.x86_64   containerd://1.4.3
```

You can observer pods are running healthy
```
user@lab-system:~/kubernetes$ kubectl get pods -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP             NODE                 NOMINATED NODE   READINESS GATES
nginx-6799fc88d8-4pc2d   1/1     Running   0          56m   192.168.94.1   kworker1.mylab.com   <none>           <none>
nginx-6799fc88d8-7gcbh   1/1     Running   0          43m   192.168.94.4   kworker1.mylab.com   <none>           <none>
nginx-6799fc88d8-kqmrw   1/1     Running   0          56m   192.168.94.2   kworker1.mylab.com   <none>           <none>
nginx-6799fc88d8-wtwrz   1/1     Running   0          43m   192.168.94.3   kworker1.mylab.com   <none>           <none>
```
Now lets uncordon kworker2 node
```
user@lab-system:~/kubernetes$ kubectl uncordon kworker2.mylab.com
node/kworker2.mylab.com uncordoned

user@lab-system:~/kubernetes$ kubectl get nodes  -o wide
NAME                 STATUS   ROLES                  AGE    VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION           CONTAINER-RUNTIME
kmaster.mylab.com    Ready    control-plane,master   131m   v1.20.2   172.42.42.100   <none>        CentOS Linux 7 (Core)   3.10.0-1127.el7.x86_64   docker://20.10.2
kworker1.mylab.com   Ready    <none>                 123m   v1.20.2   172.42.42.101   <none>        CentOS Linux 7 (Core)   3.10.0-1127.el7.x86_64   docker://20.10.2
kworker2.mylab.com   Ready    <none>                 116m   v1.20.2   172.42.42.102   <none>        CentOS Linux 7 (Core)   3.10.0-1127.el7.x86_64   containerd://1.4.3
```
## On kworker1 node , lets change Container runtime from Docker to Containerd 
Let's cordon kworker1 node , so no pods get scheduled on that host
```
user@lab-system:~/kubernetes$ kubectl cordon kworker1.mylab.com
node/kworker1.mylab.com cordoned

user@lab-system:~/kubernetes$ kubectl get nodes
NAME                 STATUS                     ROLES                  AGE     VERSION
kmaster.mylab.com    Ready                      control-plane,master   3h41m   v1.20.2
kworker1.mylab.com   Ready,SchedulingDisabled   <none>                 3h33m   v1.20.2
kworker2.mylab.com   Ready                      <none>                 3h26m   v1.20.2
```
Now let's drain kworker1 node , so pods get evicted on kworker1 node and gets scheduled on kworker2
```
user@lab-system:~/kubernetes$ kubectl drain kworker1.mylab.com --ignore-daemonsets
node/kworker1.mylab.com already cordoned
WARNING: ignoring DaemonSet-managed Pods: kube-system/calico-node-db85j, kube-system/kube-proxy-xmgwn
evicting pod default/nginx-6799fc88d8-7gcbh
evicting pod default/nginx-6799fc88d8-kqmrw
evicting pod default/nginx-6799fc88d8-wtwrz
evicting pod default/nginx-6799fc88d8-4pc2d
pod/nginx-6799fc88d8-kqmrw evicted
pod/nginx-6799fc88d8-4pc2d evicted
pod/nginx-6799fc88d8-wtwrz evicted
pod/nginx-6799fc88d8-7gcbh evicted
node/kworker1.mylab.com evicted
```
We can see all the pods are now running on kworker2 node
```
[vagrant@kmaster ~]$ kubectl get pods -o wide
NAME                     READY   STATUS    RESTARTS   AGE     IP               NODE                 NOMINATED NODE   READINESS GATES
nginx-6799fc88d8-54h75   1/1     Running   0          3m30s   192.168.28.196   kworker2.mylab.com   <none>           <none>
nginx-6799fc88d8-jbq7x   1/1     Running   0          3m30s   192.168.28.197   kworker2.mylab.com   <none>           <none>
nginx-6799fc88d8-rdfh4   1/1     Running   0          3m30s   192.168.28.195   kworker2.mylab.com   <none>           <none>
nginx-6799fc88d8-tlxkx   1/1     Running   0          3m30s   192.168.28.198   kworker2.mylab.com   <none>           <none>
```
```
user@lab-system:~/kubernetes$ kubectl get nodes
NAME                 STATUS                     ROLES                  AGE     VERSION
kmaster.mylab.com    Ready                      control-plane,master   3h45m   v1.20.2
kworker1.mylab.com   Ready,SchedulingDisabled   <none>                 3h37m   v1.20.2
kworker2.mylab.com   Ready                      <none>                 3h30m   v1.20.2
```
On other terminal , lets login to kworker1 as "root" user & repeat same steps as we did on kworker2
```
[root@kworker1 ~]# systemctl stop kubelet
[root@kworker1 ~]# systemctl stop docker
Warning: Stopping docker.service, but it can still be activated by:
  docker.socket
[root@kworker1 ~]# rpm -e docker-ce docker-ce-cli docker-ce-rootless-extras
```
By default usage of "cri" is disabled , so we need to enable by editing following containerd config file
```
[root@kworker1 ~]# vi /etc/containerd/config.toml
from 
disabled_plugins = ["cri"]
to
#disabled_plugins = ["cri"]
```
To make it effective restart containerd service
```
[root@kworker1 ~]# systemctl restart containerd
```
We need to now make "kubelet" configuration , edit following file and add 2 parameters container-runtime & container-runtime-endpoint
```
[root@kworker1 ~]# vi /var/lib/kubelet/kubeadm-flags.env 

[root@kworker1 ~]# cat /var/lib/kubelet/kubeadm-flags.env 
from 
KUBELET_KUBEADM_ARGS="--network-plugin=cni --pod-infra-container-image=k8s.gcr.io/pause:3.2"
to
KUBELET_KUBEADM_ARGS="--network-plugin=cni --pod-infra-container-image=k8s.gcr.io/pause:3.2 --container-runtime=remote --container-runtime-endpoint=unix:///run/containerd/containerd.sock"
```
Restart kubelet service
```
[root@kworker1 ~]# systemctl restart kubelet
[root@kworker1 ~]# ctr namespace list
NAME   LABELS 
k8s.io        
moby 

[root@kworker1 ~]# ctr --namespace k8s.io container list
CONTAINER                                                           IMAGE                 RUNTIME                                                                                                           
09e19608bf4f683eaad553f81b4b197bbdd21b48c32f1264adc1ff4ebfb4c575    k8s.gcr.io/pause:3.2  io.containerd.runc.v2                                                                                          
860f74ce7b2650f4e35f3ef044e971bb4c423c9e00ed41318c85e21a150943b2    k8s.gcr.io/pause:3.2  io.containerd.runc.v2
```
Now you can see STATUS from NotReady to Ready & CONTAINER-RUNTIME now for kworker2 & kworker are "containerd://1.4.3" (scroll to right >> )
```
user@lab-system:~/kubernetes$ kubectl get nodes
NAME                 STATUS                     ROLES                  AGE     VERSION
kmaster.mylab.com    Ready                      control-plane,master   3h58m   v1.20.2
kworker1.mylab.com   Ready,SchedulingDisabled   <none>                 3h50m   v1.20.2
kworker2.mylab.com   Ready                      <none>                 3h43m   v1.20.2
```
Now lets uncordon kworker1 node
```
user@lab-system:~/kubernetes$ kubectl uncordon kworker1.mylab.com
node/kworker1.mylab.com uncordoned
```
Now we can see both kworker1 & kworker2 are using containerd 
```
user@lab-system:~/kubernetes$ kubectl get nodes -o wide
NAME                 STATUS   ROLES                  AGE     VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION           CONTAINER-RUNTIME
kmaster.mylab.com    Ready    control-plane,master   3h59m   v1.20.2   172.42.42.100   <none>        CentOS Linux 7 (Core)   3.10.0-1127.el7.x86_64   docker://20.10.2
kworker1.mylab.com   Ready    <none>                 3h51m   v1.20.2   172.42.42.101   <none>        CentOS Linux 7 (Core)   3.10.0-1127.el7.x86_64   containerd://1.4.3
kworker2.mylab.com   Ready    <none>                 3h44m   v1.20.2   172.42.42.102   <none>        CentOS Linux 7 (Core)   3.10.0-1127.el7.x86_64   containerd://1.4.3
```
You can observer pods are running healthy
```
user@lab-system:~/kubernetes$ kubectl get pods -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP               NODE                 NOMINATED NODE   READINESS GATES
nginx-6799fc88d8-54h75   1/1     Running   0          17m   192.168.28.196   kworker2.mylab.com   <none>           <none>
nginx-6799fc88d8-jbq7x   1/1     Running   0          17m   192.168.28.197   kworker2.mylab.com   <none>           <none>
nginx-6799fc88d8-rdfh4   1/1     Running   0          17m   192.168.28.195   kworker2.mylab.com   <none>           <none>
nginx-6799fc88d8-tlxkx   1/1     Running   0          17m   192.168.28.198   kworker2.mylab.com   <none>           <none>
```
Just to rebalance pods run on both nodes, lets delete 2 nginx pods on kworker2

```
user@lab-system:~/kubernetes$ kubectl delete pod nginx-6799fc88d8-tlxkx
pod "nginx-6799fc88d8-tlxkx" deleted

user@lab-system:~/kubernetes$ kubectl delete pod nginx-6799fc88d8-rdfh4
pod "nginx-6799fc88d8-rdfh4" deleted


user@lab-system:~/kubernetes$ kubectl get pods -o wide
NAME                     READY   STATUS    RESTARTS   AGE     IP               NODE                 NOMINATED NODE   READINESS GATES
nginx-6799fc88d8-54h75   1/1     Running   0          21m     192.168.28.196   kworker2.mylab.com   <none>           <none>
nginx-6799fc88d8-jbq7x   1/1     Running   0          21m     192.168.28.197   kworker2.mylab.com   <none>           <none>
nginx-6799fc88d8-nzql4   1/1     Running   0          2m30s   192.168.94.5     kworker1.mylab.com   <none>           <none>
nginx-6799fc88d8-w89jj   1/1     Running   0          77s     192.168.94.6     kworker1.mylab.com   <none>           <none>
```
## On kmaster node , lets change Container runtime from Docker to Containerd 
Let's cordon kmaster node , so no pods get scheduled on that host
```
user@lab-system:~/kubernetes$ kubectl cordon kmaster.mylab.com
node/kmaster.mylab.com cordoned
```
Now let's drain kmaster node 
```
user@lab-system:~/kubernetes$ kubectl drain kmaster.mylab.com --ignore-daemonsets
node/kmaster.mylab.com already cordoned
WARNING: ignoring DaemonSet-managed Pods: kube-system/calico-node-blsr7, kube-system/kube-proxy-9lj6q
evicting pod kube-system/calico-kube-controllers-57fc9c76cc-s8jhd
evicting pod kube-system/coredns-74ff55c5b-9gnhz
evicting pod kube-system/coredns-74ff55c5b-mhp7b
pod/calico-kube-controllers-57fc9c76cc-s8jhd evicted
pod/coredns-74ff55c5b-mhp7b evicted
pod/coredns-74ff55c5b-9gnhz evicted
node/kmaster.mylab.com evicted
```
On other terminal , lets login to kmaster as "root" user & repeat same steps as we did on kworker1 & 2

```
[root@kmaster ~]# systemctl stop kubelet
```
Note : We will loose access to cluster at this stage , its OK it should be back once we starte kubelet service again in a while 
```
user@lab-system:~/kubernetes$ kubectl get nodes -o wide
The connection to the server 172.42.42.100:6443 was refused - did you specify the right host or port?
```
```
[root@kmaster ~]# rpm -e docker-ce docker-ce-cli docker-ce-rootless-extras

[root@kmaster ~]# vi /etc/containerd/config.toml
from 
disabled_plugins = ["cri"]
to
#disabled_plugins = ["cri"]
```
To make it effective restart containerd service
```
[root@kmaster ~]# systemctl restart containerd
```
By default usage of "cri" is disabled , so we need to enable by editing following containerd config file
```
[root@kmaster ~]# vi /var/lib/kubelet/kubeadm-flags.env 

[root@kmaster ~]# cat /var/lib/kubelet/kubeadm-flags.env 
from 
KUBELET_KUBEADM_ARGS="--network-plugin=cni --pod-infra-container-image=k8s.gcr.io/pause:3.2"
to
KUBELET_KUBEADM_ARGS="--network-plugin=cni --pod-infra-container-image=k8s.gcr.io/pause:3.2 --container-runtime=remote --container-runtime-endpoint=unix:///run/containerd/containerd.sock"
```
Restart kubelet service
```
[root@kmaster ~]# systemctl restart kubelet

[root@kmaster ~]# logout
```
Now we can see kmaster,kworker1 & kworker2 are using containerd 
```
user@lab-system:~/kubernetes$ kubectl get nodes,pods -o wide
NAME                      STATUS                     ROLES                  AGE     VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION           CONTAINER-RUNTIME
node/kmaster.mylab.com    Ready,SchedulingDisabled   control-plane,master   4h14m   v1.20.2   172.42.42.100   <none>        CentOS Linux 7 (Core)   3.10.0-1127.el7.x86_64   containerd://1.4.3
node/kworker1.mylab.com   Ready                      <none>                 4h6m    v1.20.2   172.42.42.101   <none>        CentOS Linux 7 (Core)   3.10.0-1127.el7.x86_64   containerd://1.4.3
node/kworker2.mylab.com   Ready                      <none>                 3h59m   v1.20.2   172.42.42.102   <none>        CentOS Linux 7 (Core)   3.10.0-1127.el7.x86_64   containerd://1.4.3

NAME                         READY   STATUS    RESTARTS   AGE   IP               NODE                 NOMINATED NODE   READINESS GATES
pod/nginx-6799fc88d8-54h75   1/1     Running   0          31m   192.168.28.196   kworker2.mylab.com   <none>           <none>
pod/nginx-6799fc88d8-jbq7x   1/1     Running   0          31m   192.168.28.197   kworker2.mylab.com   <none>           <none>
pod/nginx-6799fc88d8-nzql4   1/1     Running   0          12m   192.168.94.5     kworker1.mylab.com   <none>           <none>
pod/nginx-6799fc88d8-w89jj   1/1     Running   0          11m   192.168.94.6     kworker1.mylab.com   <none>           <none>

```
Lets uncordon kmaster
```
user@lab-system:~/kubernetes$ kubectl uncordon kmaster.mylab.com
node/kmaster.mylab.com uncordoned
```
We now switched all our nodes to use containerd as CONTAINER-RUNTIME
```
user@lab-system:~/kubernetes$ kubectl get nodes,pods -o wide
NAME                      STATUS   ROLES                  AGE     VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION           CONTAINER-RUNTIME
node/kmaster.mylab.com    Ready    control-plane,master   4h16m   v1.20.2   172.42.42.100   <none>        CentOS Linux 7 (Core)   3.10.0-1127.el7.x86_64   containerd://1.4.3
node/kworker1.mylab.com   Ready    <none>                 4h7m    v1.20.2   172.42.42.101   <none>        CentOS Linux 7 (Core)   3.10.0-1127.el7.x86_64   containerd://1.4.3
node/kworker2.mylab.com   Ready    <none>                 4h1m    v1.20.2   172.42.42.102   <none>        CentOS Linux 7 (Core)   3.10.0-1127.el7.x86_64   containerd://1.4.3

NAME                         READY   STATUS    RESTARTS   AGE   IP               NODE                 NOMINATED NODE   READINESS GATES
pod/nginx-6799fc88d8-54h75   1/1     Running   0          33m   192.168.28.196   kworker2.mylab.com   <none>           <none>
pod/nginx-6799fc88d8-jbq7x   1/1     Running   0          33m   192.168.28.197   kworker2.mylab.com   <none>           <none>
pod/nginx-6799fc88d8-nzql4   1/1     Running   0          14m   192.168.94.5     kworker1.mylab.com   <none>           <none>
pod/nginx-6799fc88d8-w89jj   1/1     Running   0          13m   192.168.94.6     kworker1.mylab.com   <none>           <none>
```
