# Upgrade Kubernetes  from version 1.18.2 to version 1.19.2
The following document show how to upgrade Kubernetes  from version 1.18.2 to version 1.19.2 on Kubernetes cluster running on RHEL7/Centos7.
We already have K8s cluster with v1.18.2 running on Centos7 

```
user@lab-server:~/projects/kubernetes/vagrant-provisioning$ kubectl get node 
NAME                 STATUS   ROLES    AGE    VERSION
kmaster.mylab.com    Ready    master   12m   v1.18.2
kworker1.mylab.com   Ready    <none>   12m   v1.18.2
kworker2.mylab.com   Ready    <none>   11m   v1.18.2
```
And nginx pods (as deployment) running on each worker nodes kworker1 & kworker2
```
user@lab-server:~/projects/kubernetes$ kubectl get pods -o wide
NAME                    READY   STATUS    RESTARTS   AGE   IP               NODE                 NOMINATED NODE   READINESS GATES
nginx-f89759699-jlrxq   1/1     Running   0          18m   192.168.28.194   kworker2.mylab.com   <none>           <none>
nginx-f89759699-rqfhm   1/1     Running   0          17m   192.168.94.1     kworker1.mylab.com   <none>           <none>
```
Now we shall upgrade K8s cluster from version 1.18.2 to version 1.19.2 - we need to make sure our nginx pods do not get disturbed

## On Master Node (login as root)

```
[root@kmaster ~]# kubeadm  version
kubeadm version: &version.Info{Major:"1", Minor:"18", GitVersion:"v1.18.2", GitCommit:"52c56ce7a8272c798dbc29846288d7cd9fbae032", GitTreeState:"clean", BuildDate:"2020-04-16T11:54:15Z", GoVersion:"go1.13.9", Compiler:"gc", Platform:"linux/amd64"}

[root@kmaster ~]# yum upgrade -y kubeadm-1.19.2
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: mirrors.piconets.webwerks.in
 * extras: mirrors.piconets.webwerks.in
 * updates: mirror.myfahim.com
Resolving Dependencies
--> Running transaction check
---> Package kubeadm.x86_64 0:1.18.2-0 will be updated
---> Package kubeadm.x86_64 0:1.19.2-0 will be an update
--> Finished Dependency Resolution

Dependencies Resolved

============================================================================================================================================
 Package                         Arch                           Version                            Repository                          Size
============================================================================================================================================
Updating:
 kubeadm                         x86_64                         1.19.2-0                           kubernetes                         8.3 M

Transaction Summary
============================================================================================================================================
Upgrade  1 Package

Total download size: 8.3 M
Downloading packages:
No Presto metadata available for kubernetes
d0ba40edfc0fdf3aeec3dd8e56c01ff0d3a511cc0012aabce55d9a83d9bf2b69-kubeadm-1.19.2-0.x86_64.rpm                         | 8.3 MB  00:00:03     
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Updating   : kubeadm-1.19.2-0.x86_64                                                                                                  1/2 
  Cleanup    : kubeadm-1.18.2-0.x86_64                                                                                                  2/2 
  Verifying  : kubeadm-1.19.2-0.x86_64                                                                                                  1/2 
  Verifying  : kubeadm-1.18.2-0.x86_64                                                                                                  2/2 

Updated:
  kubeadm.x86_64 0:1.19.2-0                                                                                                                 

Complete!
```
