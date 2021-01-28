# Upgrade Kubernetes Cluster from version 1.18.2 to version 1.19.2 with zero downtime
Upgrade Kubernetes Cluster, This requirement will be expected in any production environment. Everyone will be happy if there is no downtime for their production.
Today let’s try to upgrade my home lab Kubernetes cluster running with kubeadm. I’m running with three virtual machines for my home lab. However, the same steps are applied for any number of nodes in a critical production environment. The Kubernetes cluster can be upgraded without a zero downtime by transferring the loads from one node to another.
The following document show how to upgrade Kubernetes from version 1.18.2 to version 1.19.2 on Kubernetes cluster running on RHEL7/Centos7.
Its a 3 node cluster 1 Master & 2 Worker nodes . We already have K8s cluster with v1.18.2 running on Centos7 

The upgrade workflow at high level is the following:
   - Upgrade the primary control plane node (kmaster).
   - Upgrade worker nodes (kworker1).
   - Upgrade additional woker nodes (kworker2).

Additional information

    All containers are restarted after upgrade, because the container spec hash value is changed.
    You only can upgrade from one MINOR version to the next MINOR version, or between PATCH versions of the same MINOR. That is, you cannot skip MINOR versions when you upgrade.  For example, you can upgrade from 1.y to 1.y+1, but not from 1.y to 1.y+2
```
user@lab-server:~/projects/kubernetes/vagrant-provisioning$ kubectl get node 
NAME                 STATUS   ROLES    AGE    VERSION
kmaster.mylab.com    Ready    master   12m   v1.18.2
kworker1.mylab.com   Ready    <none>   12m   v1.18.2
kworker2.mylab.com   Ready    <none>   11m   v1.18.2
```
We are running with few of nginx deployment and its pods on worker nodes kworker1 & kworker2. In any case, during our cluster version upgrade it should not be disturbed.
```
user@lab-server:~/projects/kubernetes$ kubectl get pods -o wide
NAME                    READY   STATUS    RESTARTS   AGE   IP               NODE                 NOMINATED NODE   READINESS GATES
nginx-f89759699-jlrxq   1/1     Running   0          18m   192.168.28.194   kworker2.mylab.com   <none>           <none>
nginx-f89759699-rqfhm   1/1     Running   0          17m   192.168.94.1     kworker1.mylab.com   <none>           <none>
```
Now we shall upgrade K8s cluster from version 1.18.2 to version 1.19.2 - we need to make sure our nginx pods do not get disturbed

## Upgrading control plane node, aka kmaster Node (login as root)
The Current version is v1.18.2 let’s plan to upgrade Kubernetes cluster version to 1.19.2. To know the version we can run the commands

```
$ kubectl get node 
NAME                 STATUS   ROLES    AGE    VERSION
kmaster.mylab.com    Ready    master   12m   v1.18.2
kworker1.mylab.com   Ready    <none>   12m   v1.18.2
kworker2.mylab.com   Ready    <none>   11m   v1.18.2
```
-	On the control plane node, run:
You should see output similar to this:
```
[root@kmaster ~]# kubeadm  version
kubeadm version: &version.Info{Major:"1", Minor:"18", GitVersion:"v1.18.2", GitCommit:"52c56ce7a8272c798dbc29846288d7cd9fbae032", GitTreeState:"clean", BuildDate:"2020-04-16T11:54:15Z", GoVersion:"go1.13.9", Compiler:"gc", Platform:"linux/amd64"}
```

```
[root@kmaster ~]# yum upgrade -y kubeadm-1.19.2
```
You should see output similar to this:
```
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
Now lets check the kubeadm version on kmaster node
```
[root@kmaster ~]# kubeadm  version
kubeadm version: &version.Info{Major:"1", Minor:"19", GitVersion:"v1.19.2", GitCommit:"f5743093fd1c663cb0cbc89748f730662345d44d", GitTreeState:"clean", BuildDate:"2020-09-16T13:38:53Z", GoVersion:"go1.15", Compiler:"gc", Platform:"linux/amd64"}
```
To start with the upgrade, the kubeadm will provide with a plan. Using the plan we are good to proceed safely and smoothly.

It will print the current cluster version, available latest stable version. Moreover, it will print the Kubernetes cluster components which are available for upgrade. Finally, it will print the command for an upgrade.

```
[root@kmaster ~]# kubeadm  upgrade plan
[upgrade/config] Making sure the configuration is correct:
[upgrade/config] Reading configuration from the cluster...
[upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[preflight] Running pre-flight checks.
[upgrade] Running cluster health checks
[upgrade] Fetching available versions to upgrade to
[upgrade/versions] Cluster version: v1.18.15
[upgrade/versions] kubeadm version: v1.19.2
I0127 09:27:49.529928    4455 version.go:252] remote version is much newer: v1.20.2; falling back to: stable-1.19
[upgrade/versions] Latest stable version: v1.19.7
[upgrade/versions] Latest stable version: v1.19.7
[upgrade/versions] Latest version in the v1.18 series: v1.18.15
[upgrade/versions] Latest version in the v1.18 series: v1.18.15

Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
COMPONENT   CURRENT       AVAILABLE
kubelet     3 x v1.18.2   v1.19.7

Upgrade to the latest stable version:

COMPONENT                 CURRENT    AVAILABLE
kube-apiserver            v1.18.15   v1.19.7
kube-controller-manager   v1.18.15   v1.19.7
kube-scheduler            v1.18.15   v1.19.7
kube-proxy                v1.18.15   v1.19.7
CoreDNS                   1.6.7      1.7.0
etcd                      3.4.3-0    3.4.13-0

You can now apply the upgrade by executing the following command:

	kubeadm upgrade apply v1.19.7

Note: Before you can perform this upgrade, you have to update kubeadm to v1.19.7.

_____________________________________________________________________


The table below shows the current state of component configs as understood by this version of kubeadm.
Configs that have a "yes" mark in the "MANUAL UPGRADE REQUIRED" column require manual config upgrade or
resetting to kubeadm defaults before a successful upgrade can be performed. The version to manually
upgrade to is denoted in the "PREFERRED VERSION" column.

API GROUP                 CURRENT VERSION   PREFERRED VERSION   MANUAL UPGRADE REQUIRED
kubeproxy.config.k8s.io   v1alpha1          v1alpha1            no
kubelet.config.k8s.io     v1beta1           v1beta1             no
_____________________________________________________________________
```
This command checks that your cluster can be upgraded, and fetches the versions you can upgrade to. It also shows a table with the component config version states.

Note: kubeadm upgrade also automatically renews the certificates that it manages on this node. To opt-out of certificate renewal the flag --certificate-renewal=false can be used. For more information see the certificate management guide.
Note: If kubeadm upgrade plan shows any component configs that require manual upgrade, users must provide a config file with replacement configs to kubeadm upgrade apply via the --config command line flag. Failing to do so will cause kubeadm upgrade apply to exit with an error and not perform an upgrade.

Choose a version to upgrade to, and run the appropriate command. For example:v1.19.2
### Let’s upgrade now.
Upgrade Kubernetes Cluster
Run the command which we got from the above output. This will pull the required images and start to upgrade the cluster.
```
[root@kmaster ~]# kubeadm upgrade apply v1.19.2
[upgrade/config] Making sure the configuration is correct:
[upgrade/config] Reading configuration from the cluster...
[upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[preflight] Running pre-flight checks.
[upgrade] Running cluster health checks
[upgrade/version] You have chosen to change the cluster version to "v1.19.2"
[upgrade/versions] Cluster version: v1.18.15
[upgrade/versions] kubeadm version: v1.19.2
[upgrade/confirm] Are you sure you want to proceed with the upgrade? [y/N]: y
[upgrade/prepull] Pulling images required for setting up a Kubernetes cluster
[upgrade/prepull] This might take a minute or two, depending on the speed of your internet connection
[upgrade/prepull] You can also perform this action in beforehand using 'kubeadm config images pull'
[upgrade/apply] Upgrading your Static Pod-hosted control plane to version "v1.19.2"...
Static pod: kube-apiserver-kmaster.mylab.com hash: 8ca6e0d6afa9a758a9371ba45baae192
Static pod: kube-controller-manager-kmaster.mylab.com hash: 9cb741915ac7dff3b2c42c8af5dff440
Static pod: kube-scheduler-kmaster.mylab.com hash: 76e104b1738c4d035ec548e977a4181c
[upgrade/etcd] Upgrading to TLS for etcd
Static pod: etcd-kmaster.mylab.com hash: ef02d47ad4dcbfbb75aba42051d13d4a
[upgrade/staticpods] Preparing for "etcd" upgrade
[upgrade/staticpods] Renewing etcd-server certificate
[upgrade/staticpods] Renewing etcd-peer certificate
[upgrade/staticpods] Renewing etcd-healthcheck-client certificate
[upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/etcd.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2021-01-27-09-33-07/etcd.yaml"
[upgrade/staticpods] Waiting for the kubelet to restart the component
[upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
Static pod: etcd-kmaster.mylab.com hash: ef02d47ad4dcbfbb75aba42051d13d4a
Static pod: etcd-kmaster.mylab.com hash: ef02d47ad4dcbfbb75aba42051d13d4a
Static pod: etcd-kmaster.mylab.com hash: a61b4f74daba6c457b2b2b43b31c46d9
[apiclient] Found 1 Pods for label selector component=etcd
[upgrade/staticpods] Component "etcd" upgraded successfully!
[upgrade/etcd] Waiting for etcd to become available
[upgrade/staticpods] Writing new Static Pod manifests to "/etc/kubernetes/tmp/kubeadm-upgraded-manifests763885084"
[upgrade/staticpods] Preparing for "kube-apiserver" upgrade
[upgrade/staticpods] Renewing apiserver certificate
[upgrade/staticpods] Renewing apiserver-kubelet-client certificate
[upgrade/staticpods] Renewing front-proxy-client certificate
[upgrade/staticpods] Renewing apiserver-etcd-client certificate
[upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-apiserver.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2021-01-27-09-33-07/kube-apiserver.yaml"
[upgrade/staticpods] Waiting for the kubelet to restart the component
[upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
Static pod: kube-apiserver-kmaster.mylab.com hash: 8ca6e0d6afa9a758a9371ba45baae192
Static pod: kube-apiserver-kmaster.mylab.com hash: 8ca6e0d6afa9a758a9371ba45baae192
Static pod: kube-apiserver-kmaster.mylab.com hash: 38c2abe2f426a0d26849ccbd8e67170e
[apiclient] Found 1 Pods for label selector component=kube-apiserver
[upgrade/staticpods] Component "kube-apiserver" upgraded successfully!
[upgrade/staticpods] Preparing for "kube-controller-manager" upgrade
[upgrade/staticpods] Renewing controller-manager.conf certificate
[upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-controller-manager.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2021-01-27-09-33-07/kube-controller-manager.yaml"
[upgrade/staticpods] Waiting for the kubelet to restart the component
[upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
Static pod: kube-controller-manager-kmaster.mylab.com hash: 9cb741915ac7dff3b2c42c8af5dff440
Static pod: kube-controller-manager-kmaster.mylab.com hash: a83e8bfcf7a49a3cb0bd90a416dc56ad
[apiclient] Found 1 Pods for label selector component=kube-controller-manager
[upgrade/staticpods] Component "kube-controller-manager" upgraded successfully!
[upgrade/staticpods] Preparing for "kube-scheduler" upgrade
[upgrade/staticpods] Renewing scheduler.conf certificate
[upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-scheduler.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2021-01-27-09-33-07/kube-scheduler.yaml"
[upgrade/staticpods] Waiting for the kubelet to restart the component
[upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
Static pod: kube-scheduler-kmaster.mylab.com hash: 76e104b1738c4d035ec548e977a4181c
Static pod: kube-scheduler-kmaster.mylab.com hash: 6c2614c45defe847469047ffeb7b41ea
[apiclient] Found 1 Pods for label selector component=kube-scheduler
[upgrade/staticpods] Component "kube-scheduler" upgraded successfully!
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.19" in namespace kube-system with the configuration for the kubelets in the cluster
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

[upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.19.2". Enjoy!

[upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.
[root@kmaster ~]#
```
That’s it, we have upgraded the cluster. Let’s try to get the nodes to verify the version.

ON OTHER TERMINAL WE CAN SEE OUR NGINX PODS DID NOT BREAK
```
user@lab-server:~/projects/kubernetes$ kubectl get pods -o wide
NAME                    READY   STATUS    RESTARTS   AGE   IP               NODE                 NOMINATED NODE   READINESS GATES
nginx-f89759699-jlrxq   1/1     Running   0          18m   192.168.28.194   kworker2.mylab.com   <none>           <none>
nginx-f89759699-rqfhm   1/1     Running   0          17m   192.168.94.1     kworker1.mylab.com   <none>           <none>
user@lab-server:~/projects/kubernetes$
```
Still our Kubelets are all in version 1.18.2
```
user@lab-server:~/projects/kubernetes$ kubectl get nodes
NAME                 STATUS   ROLES    AGE    VERSION
kmaster.mylab.com    Ready    master   105m   v1.18.2
kworker1.mylab.com   Ready    <none>   100m   v1.18.2
kworker2.mylab.com   Ready    <none>   96m    v1.18.2
```
BUT WE CAN SEE THE CLUSTER IS UPGRADED TO 1.19.2 
```
user@lab-server:~/projects/kubernetes$ kubectl version --short
Client Version: v1.18.4
Server Version: v1.19.2
```
On control machine , Let’s see how to drain the host 
```
user@lab-server:~/projects/kubernetes$ kubectl drain kmaster.mylab.com --ignore-daemonsets
node/kmaster.mylab.com cordoned
WARNING: ignoring DaemonSet-managed Pods: kube-system/calico-node-g576h, kube-system/kube-proxy-t4tq6
evicting pod kube-system/calico-kube-controllers-d85d4bdcd-jcvw7
pod/calico-kube-controllers-d85d4bdcd-jcvw7 evicted
node/kmaster.mylab.com evicted
user@lab-server:~/projects/kubernetes$
```
```
user@lab-server:~/projects/kubernetes$ kubectl get nodes -o wide
NAME                 STATUS                     ROLES    AGE    VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION           CONTAINER-RUNTIME
kmaster.mylab.com    Ready,SchedulingDisabled   master   115m   v1.18.2   172.42.42.100   <none>        CentOS Linux 7 (Core)   3.10.0-1127.el7.x86_64   docker://20.10.2
kworker1.mylab.com   Ready                      <none>   110m   v1.18.2   172.42.42.101   <none>        CentOS Linux 7 (Core)   3.10.0-1127.el7.x86_64   docker://20.10.2
kworker2.mylab.com   Ready                      <none>   106m   v1.18.2   172.42.42.102   <none>        CentOS Linux 7 (Core)   3.10.0-1127.el7.x86_64   docker://20.10.2
```
On Master Node , Upgrade kubelet and kubectl 
```
[root@kmaster kubernetes]# yum upgrade -y kubelet-1.19.2 
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: mirrors.piconets.webwerks.in
 * extras: mirrors.piconets.webwerks.in
 * updates: mirror.myfahim.com
Resolving Dependencies
--> Running transaction check
---> Package kubelet.x86_64 0:1.18.2-0 will be updated
---> Package kubelet.x86_64 0:1.19.2-0 will be an update
--> Finished Dependency Resolution

Dependencies Resolved

============================================================================================================================================
 Package                         Arch                           Version                            Repository                          Size
============================================================================================================================================
Updating:
 kubelet                         x86_64                         1.19.2-0                           kubernetes                          19 M

Transaction Summary
============================================================================================================================================
Upgrade  1 Package

Total download size: 19 M
Downloading packages:
No Presto metadata available for kubernetes
d9d997cdbfd6562824eb7786abbc7f4c6a6825662d0f451793aa5ab8c4a85c96-kubelet-1.19.2-0.x86_64.rpm                         |  19 MB  00:00:06     
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Updating   : kubelet-1.19.2-0.x86_64                                                                                                  1/2 
  Cleanup    : kubelet-1.18.2-0.x86_64                                                                                                  2/2 
  Verifying  : kubelet-1.19.2-0.x86_64                                                                                                  1/2 
  Verifying  : kubelet-1.18.2-0.x86_64                                                                                                  2/2 

Updated:
  kubelet.x86_64 0:1.19.2-0                                                                                                                 

Complete!
```
RESTART KUBELET SERVICE on MASTER NODE TO MAKE IT EFFECTIVE
```
[root@kmaster kubernetes]# systemctl daemon-reload
[root@kmaster kubernetes]# systemctl restart kubelet

[root@kmaster kubernetes]# systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
   Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; vendor preset: disabled)
  Drop-In: /usr/lib/systemd/system/kubelet.service.d
           └─10-kubeadm.conf
   Active: active (running) since Wed 2021-01-27 09:50:55 UTC; 46s ago
     Docs: https://kubernetes.io/docs/
 Main PID: 26540 (kubelet)
    Tasks: 15
   Memory: 36.7M
   CGroup: /system.slice/kubelet.service
           └─26540 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.co...

Jan 27 09:51:02 kmaster.mylab.com kubelet[26540]: I0127 09:51:02.333878   26540 reconciler.go:224] operationExecutor.VerifyControllerAtta...
Jan 27 09:51:02 kmaster.mylab.com kubelet[26540]: I0127 09:51:02.333980   26540 reconciler.go:224] operationExecutor.VerifyControll...d85c")
Jan 27 09:51:02 kmaster.mylab.com kubelet[26540]: I0127 09:51:02.334067   26540 reconciler.go:224] operationExecutor.VerifyControllerAtta...
Jan 27 09:51:02 kmaster.mylab.com kubelet[26540]: I0127 09:51:02.334178   26540 reconciler.go:224] operationExecutor.VerifyControll...cd19")
Jan 27 09:51:02 kmaster.mylab.com kubelet[26540]: I0127 09:51:02.334292   26540 reconciler.go:224] operationExecutor.VerifyControll...2945")
Jan 27 09:51:02 kmaster.mylab.com kubelet[26540]: I0127 09:51:02.334417   26540 reconciler.go:224] operationExecutor.VerifyControllerAtta...
Jan 27 09:51:02 kmaster.mylab.com kubelet[26540]: I0127 09:51:02.334514   26540 reconciler.go:224] operationExecutor.VerifyControll...d85c")
Jan 27 09:51:02 kmaster.mylab.com kubelet[26540]: I0127 09:51:02.334660   26540 reconciler.go:224] operationExecutor.VerifyControllerAtta...
Jan 27 09:51:02 kmaster.mylab.com kubelet[26540]: I0127 09:51:02.335201   26540 reconciler.go:157] Reconciler: start to sync state
Jan 27 09:51:03 kmaster.mylab.com kubelet[26540]: I0127 09:51:03.217485   26540 request.go:645] Throttling request took 1.039321868...sion=0
Hint: Some lines were ellipsized, use -l to show in full.
```

On Control node Pods are running fine
```
user@lab-server:~/projects/kubernetes$ kubectl get all -o wide
NAME                        READY   STATUS    RESTARTS   AGE   IP               NODE                 NOMINATED NODE   READINESS GATES
pod/nginx-f89759699-jlrxq   1/1     Running   0          36m   192.168.28.194   kworker2.mylab.com   <none>           <none>
pod/nginx-f89759699-rqfhm   1/1     Running   0          34m   192.168.94.1     kworker1.mylab.com   <none>           <none>

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE    SELECTOR
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   121m   <none>

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES   SELECTOR
deployment.apps/nginx   2/2     2            2           36m   nginx        nginx    app=nginx

NAME                              DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES   SELECTOR
replicaset.apps/nginx-f89759699   2         2         2       36m   nginx        nginx    app=nginx,pod-template-hash=f89759699
```
Once again run the get command to print the version. Now we can see the upgraded version of master node as v1.19.2.
```
user@lab-server:~/projects/kubernetes$ kubectl get nodes
NAME                 STATUS                     ROLES    AGE    VERSION
kmaster.mylab.com    Ready,SchedulingDisabled   master   122m   v1.19.2
kworker1.mylab.com   Ready                      <none>   116m   v1.18.2
kworker2.mylab.com   Ready                      <none>   113m   v1.18.2
```
That’s it for the master node part in our “Upgrade Kubernetes cluster” guide.

NOW UNCORDON THE MASTER NODE , SO MASTER WILL BE IN "Ready" STATUS
Bring the node back online by marking it schedulable:
```
user@lab-server:~/projects/kubernetes$ kubectl uncordon kmaster.mylab.com
node/kmaster.mylab.com uncordoned

user@lab-server:~/projects/kubernetes$ kubectl get nodes
NAME                 STATUS   ROLES    AGE    VERSION
kmaster.mylab.com    Ready    master   123m   v1.19.2
kworker1.mylab.com   Ready    <none>   118m   v1.18.2
kworker2.mylab.com   Ready    <none>   114m   v1.18.2
```
## Upgrade worker nodes
Now it’s time to start the upgrade with the worker nodes.

Now lets upgrade on KWORKER1 Node

Login as root to KWORKER1
Prepare the KWORKER1 node for maintenance by marking it unschedulable and evicting the workloads:

```
user@lab-server:~/projects/kubernetes$ kubectl drain kworker1.mylab.com --ignore-daemonsets
node/kworker1.mylab.com cordoned
WARNING: ignoring DaemonSet-managed Pods: kube-system/calico-node-frndx, kube-system/kube-proxy-r24pz
evicting pod kube-system/calico-kube-controllers-d85d4bdcd-cr86j
evicting pod default/nginx-f89759699-rqfhm
evicting pod kube-system/coredns-f9fd979d6-fb22q
pod/coredns-f9fd979d6-fb22q evicted
pod/calico-kube-controllers-d85d4bdcd-cr86j evicted
pod/nginx-f89759699-rqfhm evicted
node/kworker1.mylab.com evicted

```
 After drain the node, the scheduling with be disabled on kworker1.mylab.com
 
 FROM CONTROL NODE : NOW WE CAN SEE THE NGINXPOD GOT DELETED FROM KWORKER1 & GOT CREATED ON KWORKER2
 Once again verify where the pods are residing.
 ```
 user@lab-server:~/projects/kubernetes$ kubectl get all -o wide
NAME                        READY   STATUS    RESTARTS   AGE   IP               NODE                 NOMINATED NODE   READINESS GATES
pod/nginx-f89759699-4ww9z   1/1     Running   0          49s   192.168.28.196   kworker2.mylab.com   <none>           <none>
pod/nginx-f89759699-jlrxq   1/1     Running   0          47m   192.168.28.194   kworker2.mylab.com   <none>           <none>

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE    SELECTOR
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   132m   <none>

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES   SELECTOR
deployment.apps/nginx   2/2     2            2           47m   nginx        nginx    app=nginx

NAME                              DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES   SELECTOR
replicaset.apps/nginx-f89759699   2         2         2       47m   nginx        nginx    app=nginx,pod-template-hash=f89759699
user@lab-server:~/projects/kubernetes$
```
Now, we are able to see the pods are not running on kworker2.mylab.com

ON KWORKER1 as root
```
[root@kworker1 ~]# kubelet --version | cut -d '' -f 2
Kubernetes v1.18.2
```
Upgrade kubeadm on worker node1
```
[root@kworker1 ~]# kubeadm upgrade node
[upgrade] Reading configuration from the cluster...
[upgrade] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[upgrade] Skipping phase. Not a control plane node.
[kubelet-start] Downloading configuration for the kubelet from the "kubelet-config-1.19" ConfigMap in the kube-system namespace
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[upgrade] The configuration for this node was successfully updated!
[upgrade] Now you should go ahead and upgrade the kubelet package using your package manager.
```
Upgrade kubelet and kubectl 

```
[root@kworker1 ~]# yum upgrade -y kubelet-1.19.2 kubectl-1.19.2 --disableexcludes=kubernetes
Downloading packages:
No Presto metadata available for kubernetes
(1/2): b1b077555664655ba01b2c68d13239eaf9db1025287d0d9ccaeb4a8850c7a9b7-kubectl-1.19.2-0.x86_64.rpm                  | 9.0 MB  00:00:03     
(2/2): d9d997cdbfd6562824eb7786abbc7f4c6a6825662d0f451793aa5ab8c4a85c96-kubelet-1.19.2-0.x86_64.rpm                  |  19 MB  00:00:05     
--------------------------------------------------------------------------------------------------------------------------------------------
Total                                                                                                       4.8 MB/s |  29 MB  00:00:05     
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Updating   : kubelet-1.19.2-0.x86_64                                                                                                  1/4 
  Updating   : kubectl-1.19.2-0.x86_64                                                                                                  2/4 
  Cleanup    : kubectl-1.18.2-0.x86_64                                                                                                  3/4 
  Cleanup    : kubelet-1.18.2-0.x86_64                                                                                                  4/4 
  Verifying  : kubectl-1.19.2-0.x86_64                                                                                                  1/4 
  Verifying  : kubelet-1.19.2-0.x86_64                                                                                                  2/4 
  Verifying  : kubelet-1.18.2-0.x86_64                                                                                                  3/4 
  Verifying  : kubectl-1.18.2-0.x86_64                                                                                                  4/4 

Updated:
  kubectl.x86_64 0:1.19.2-0                                            kubelet.x86_64 0:1.19.2-0                                           

Complete

[root@kworker1 ~]# systemctl daemon-reload
[root@kworker1 ~]# systemctl restart kubelet
[root@kworker1 ~]# systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
   Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; vendor preset: disabled)
  Drop-In: /usr/lib/systemd/system/kubelet.service.d
           └─10-kubeadm.conf
   Active: active (running) since Wed 2021-01-27 10:21:38 UTC; 13s ago
     Docs: https://kubernetes.io/docs/
 Main PID: 5132 (kubelet)
    Tasks: 15
   Memory: 33.6M
   CGroup: /system.slice/kubelet.service
           └─5132 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.con...

Jan 27 10:21:45 kworker1.mylab.com kubelet[5132]: I0127 10:21:45.182483    5132 reconciler.go:224] operationExecutor.VerifyControll...b09a")
Jan 27 10:21:45 kworker1.mylab.com kubelet[5132]: I0127 10:21:45.182539    5132 reconciler.go:224] operationExecutor.VerifyControllerAtta...
Jan 27 10:21:45 kworker1.mylab.com kubelet[5132]: I0127 10:21:45.182644    5132 reconciler.go:224] operationExecutor.VerifyControllerAtta...
Jan 27 10:21:45 kworker1.mylab.com kubelet[5132]: I0127 10:21:45.182744    5132 reconciler.go:224] operationExecutor.VerifyControll...395c")
Jan 27 10:21:45 kworker1.mylab.com kubelet[5132]: I0127 10:21:45.182922    5132 reconciler.go:224] operationExecutor.VerifyControll...b09a")
Jan 27 10:21:45 kworker1.mylab.com kubelet[5132]: I0127 10:21:45.183068    5132 reconciler.go:224] operationExecutor.VerifyControll...395c")
Jan 27 10:21:45 kworker1.mylab.com kubelet[5132]: I0127 10:21:45.183280    5132 reconciler.go:224] operationExecutor.VerifyControll...b09a")
Jan 27 10:21:45 kworker1.mylab.com kubelet[5132]: I0127 10:21:45.183364    5132 reconciler.go:224] operationExecutor.VerifyControll...b09a")
Jan 27 10:21:45 kworker1.mylab.com kubelet[5132]: I0127 10:21:45.183425    5132 reconciler.go:224] operationExecutor.VerifyControllerAtta...
Jan 27 10:21:45 kworker1.mylab.com kubelet[5132]: I0127 10:21:45.183459    5132 reconciler.go:157] Reconciler: start to sync state
Hint: Some lines were ellipsized, use -l to show in full.

```
Now our kmaster & kworker1 are upgraded to v1.19.2
```
user@lab-server:~/projects/kubernetes$ kubectl get nodes
NAME                 STATUS                     ROLES    AGE    VERSION
kmaster.mylab.com    Ready                      master   153m   v1.19.2
kworker1.mylab.com   Ready,SchedulingDisabled   <none>   147m   v1.19.2
kworker2.mylab.com   Ready                      <none>   143m   v1.18.2
```

ON CONTROL NODE, STILL OUR PODS ARE HEALTHY
```
user@lab-server:~/projects/kubernetes$ kubectl get all -o wide
NAME                        READY   STATUS    RESTARTS   AGE   IP               NODE                 NOMINATED NODE   READINESS GATES
pod/nginx-f89759699-4ww9z   1/1     Running   0          16m   192.168.28.196   kworker2.mylab.com   <none>           <none>
pod/nginx-f89759699-jlrxq   1/1     Running   0          63m   192.168.28.194   kworker2.mylab.com   <none>           <none>

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE    SELECTOR
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   148m   <none>

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES   SELECTOR
deployment.apps/nginx   2/2     2            2           63m   nginx        nginx    app=nginx

NAME                              DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES   SELECTOR
replicaset.apps/nginx-f89759699   2         2         2       63m   nginx        nginx    app=nginx,pod-template-hash=f89759699

user@lab-server:~/projects/kubernetes$ kubectl get nodes
NAME                 STATUS                     ROLES    AGE    VERSION
kmaster.mylab.com    Ready                      master   153m   v1.19.2
kworker1.mylab.com   Ready,SchedulingDisabled   <none>   147m   v1.19.2
kworker2.mylab.com   Ready                      <none>   143m   v1.18.2
```

NOW WE HAVE SUCCESSFULLY UPGRADED KWORKER1 TO 1.19.2 , LETS UNCORDON KWORKER1
Bring the node back online by marking it schedulable

```
user@lab-server:~/projects/kubernetes$ kubectl uncordon kworker1.mylab.com
node/kworker1.mylab.com uncordoned
```
Verify the status of the cluster 
```
user@lab-server:~/projects/kubernetes$ kubectl get nodes
NAME                 STATUS   ROLES    AGE    VERSION
kmaster.mylab.com    Ready    master   154m   v1.19.2
kworker1.mylab.com   Ready    <none>   148m   v1.19.2
kworker2.mylab.com   Ready    <none>   145m   v1.18.2
```
The STATUS column should show Ready for all your nodes, and the version number should be updated

NOW LET US UPGRADE KWORKER2 , DRAIN THE PODS 
```
user@lab-server:~/projects/kubernetes$ kubectl drain kworker2.mylab.com --ignore-daemonsets
node/kworker2.mylab.com cordoned
WARNING: ignoring DaemonSet-managed Pods: kube-system/calico-node-bwfn5, kube-system/kube-proxy-fv8j5
evicting pod kube-system/calico-kube-controllers-d85d4bdcd-x4mp7
evicting pod default/nginx-f89759699-jlrxq
evicting pod kube-system/coredns-f9fd979d6-9w9xs
evicting pod default/nginx-f89759699-4ww9z
pod/coredns-f9fd979d6-9w9xs evicted
pod/nginx-f89759699-4ww9z evicted
pod/nginx-f89759699-jlrxq evicted
pod/calico-kube-controllers-d85d4bdcd-x4mp7 evicted
node/kworker2.mylab.com evicted
user@lab-server:~/projects/kubernetes$

user@lab-server:~/projects/kubernetes$ kubectl get all -o wide
NAME                        READY   STATUS              RESTARTS   AGE   IP             NODE                 NOMINATED NODE   READINESS GATES
pod/nginx-f89759699-2psmw   0/1     ContainerCreating   0          62s   <none>         kworker1.mylab.com   <none>           <none>
pod/nginx-f89759699-4c2cn   1/1     Running             0          62s   192.168.94.7   kworker1.mylab.com   <none>           <none>

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE    SELECTOR
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   157m   <none>

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES   SELECTOR
deployment.apps/nginx   1/2     2            1           72m   nginx        nginx    app=nginx

NAME                              DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES   SELECTOR
replicaset.apps/nginx-f89759699   2         2         1       72m   nginx        nginx    app=nginx,pod-template-hash=f89759699



user@lab-server:~/projects/kubernetes$ kubectl get all -o wide
NAME                        READY   STATUS    RESTARTS   AGE   IP             NODE                 NOMINATED NODE   READINESS GATES
pod/nginx-f89759699-2psmw   1/1     Running   0          81s   192.168.94.9   kworker1.mylab.com   <none>           <none>
pod/nginx-f89759699-4c2cn   1/1     Running   0          81s   192.168.94.7   kworker1.mylab.com   <none>           <none>

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE    SELECTOR
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   158m   <none>

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES   SELECTOR
deployment.apps/nginx   2/2     2            2           73m   nginx        nginx    app=nginx

NAME                              DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES   SELECTOR
replicaset.apps/nginx-f89759699   2         2         2       73m   nginx        nginx    app=nginx,pod-template-hash=f89759699

[root@kworker2 ~]# kubeadm upgrade node
[upgrade] Reading configuration from the cluster...
[upgrade] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[upgrade] Skipping phase. Not a control plane node.
[kubelet-start] Downloading configuration for the kubelet from the "kubelet-config-1.19" ConfigMap in the kube-system namespace
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[upgrade] The configuration for this node was successfully updated!
[upgrade] Now you should go ahead and upgrade the kubelet package using your package manager.
[root@kworker2 ~]# 
[root@kworker2 ~]# 
[root@kworker2 ~]# 
[root@kworker2 ~]# yum upgrade -y kubelet-1.19.2 kubectl-1.19.2 --disableexcludes=kubernetes
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: centos.excellmedia.net
 * extras: centos.excellmedia.net
 * updates: centos.excellmedia.net
Resolving Dependencies
--> Running transaction check
---> Package kubectl.x86_64 0:1.18.2-0 will be updated
---> Package kubectl.x86_64 0:1.19.2-0 will be an update
---> Package kubelet.x86_64 0:1.18.2-0 will be updated
---> Package kubelet.x86_64 0:1.19.2-0 will be an update
--> Finished Dependency Resolution

Dependencies Resolved

============================================================================================================================================
 Package                         Arch                           Version                            Repository                          Size
============================================================================================================================================
Updating:
 kubectl                         x86_64                         1.19.2-0                           kubernetes                         9.0 M
 kubelet                         x86_64                         1.19.2-0                           kubernetes                          19 M

Transaction Summary
============================================================================================================================================
Upgrade  2 Packages

Total download size: 29 M
Downloading packages:
No Presto metadata available for kubernetes
(1/2): b1b077555664655ba01b2c68d13239eaf9db1025287d0d9ccaeb4a8850c7a9b7-kubectl-1.19.2-0.x86_64.rpm                  | 9.0 MB  00:00:02     
(2/2): d9d997cdbfd6562824eb7786abbc7f4c6a6825662d0f451793aa5ab8c4a85c96-kubelet-1.19.2-0.x86_64.rpm                  |  19 MB  00:00:03     
--------------------------------------------------------------------------------------------------------------------------------------------
Total                                                                                                       7.3 MB/s |  29 MB  00:00:03     
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Updating   : kubelet-1.19.2-0.x86_64                                                                                                  1/4 
  Updating   : kubectl-1.19.2-0.x86_64                                                                                                  2/4 
  Cleanup    : kubectl-1.18.2-0.x86_64                                                                                                  3/4 
  Cleanup    : kubelet-1.18.2-0.x86_64                                                                                                  4/4 
  Verifying  : kubectl-1.19.2-0.x86_64                                                                                                  1/4 
  Verifying  : kubelet-1.19.2-0.x86_64                                                                                                  2/4 
  Verifying  : kubelet-1.18.2-0.x86_64                                                                                                  3/4 
  Verifying  : kubectl-1.18.2-0.x86_64                                                                                                  4/4 

Updated:
  kubectl.x86_64 0:1.19.2-0                                            kubelet.x86_64 0:1.19.2-0                                           

Complete!
[root@kworker2 ~]# 
[root@kworker2 ~]# 
[root@kworker2 ~]# systemctl daemon-reload
[root@kworker2 ~]# systemctl restart kubelet
[root@kworker2 ~]#  systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
   Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; vendor preset: disabled)
  Drop-In: /usr/lib/systemd/system/kubelet.service.d
           └─10-kubeadm.conf
   Active: active (running) since Wed 2021-01-27 10:31:40 UTC; 7s ago
     Docs: https://kubernetes.io/docs/
 Main PID: 11143 (kubelet)
    Tasks: 14
   Memory: 33.9M
   CGroup: /system.slice/kubelet.service
           └─11143 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.co...

Jan 27 10:31:47 kworker2.mylab.com kubelet[11143]: I0127 10:31:47.532156   11143 reconciler.go:224] operationExecutor.VerifyControl...ac53")
Jan 27 10:31:47 kworker2.mylab.com kubelet[11143]: I0127 10:31:47.532201   11143 reconciler.go:224] operationExecutor.VerifyControl...ac53")
Jan 27 10:31:47 kworker2.mylab.com kubelet[11143]: I0127 10:31:47.532277   11143 reconciler.go:224] operationExecutor.VerifyControllerAtt...
Jan 27 10:31:47 kworker2.mylab.com kubelet[11143]: I0127 10:31:47.532351   11143 reconciler.go:224] operationExecutor.VerifyControl...fdad")
Jan 27 10:31:47 kworker2.mylab.com kubelet[11143]: I0127 10:31:47.532462   11143 reconciler.go:224] operationExecutor.VerifyControl...fdad")
Jan 27 10:31:47 kworker2.mylab.com kubelet[11143]: I0127 10:31:47.532741   11143 reconciler.go:224] operationExecutor.VerifyControl...fdad")
Jan 27 10:31:47 kworker2.mylab.com kubelet[11143]: I0127 10:31:47.532828   11143 reconciler.go:224] operationExecutor.VerifyControl...fdad")
Jan 27 10:31:47 kworker2.mylab.com kubelet[11143]: I0127 10:31:47.532953   11143 reconciler.go:224] operationExecutor.VerifyControl...fdad")
Jan 27 10:31:47 kworker2.mylab.com kubelet[11143]: I0127 10:31:47.533053   11143 reconciler.go:224] operationExecutor.VerifyControl...fdad")
Jan 27 10:31:47 kworker2.mylab.com kubelet[11143]: I0127 10:31:47.533108   11143 reconciler.go:157] Reconciler: start to sync state
Hint: Some lines were ellipsized, use -l to show in full.
[root@kworker2 ~]# 

```

ON CONTROL NODE

```
user@lab-server:~/projects/kubernetes$ kubectl get node 
NAME                 STATUS                     ROLES    AGE    VERSION
kmaster.mylab.com    Ready                      master   161m   v1.19.2
kworker1.mylab.com   Ready                      <none>   156m   v1.19.2
kworker2.mylab.com   Ready,SchedulingDisabled   <none>   152m   v1.19.2
user@lab-server:~/projects/kubernetes$ 
```
LETS UNCORDON KWORKER2 NODE
```
user@lab-server:~/projects/kubernetes$ kubectl uncordon kworker2.mylab.com
node/kworker2.mylab.com uncordoned
user@lab-server:~/projects/kubernetes$ kubectl get node 
NAME                 STATUS   ROLES    AGE    VERSION
kmaster.mylab.com    Ready    master   163m   v1.19.2
kworker1.mylab.com   Ready    <none>   157m   v1.19.2
kworker2.mylab.com   Ready    <none>   154m   v1.19.2
user@lab-server:~/projects/kubernetes$ 
```
LETS DELETE 1 NGINXPOD TO SEE IT GETS DELETED on KWORKER1 AND SCHDEULE TO GET CREATE ON KWORKER2
```
user@lab-server:~/projects/kubernetes$ kubectl delete pod/nginx-f89759699-4c2cn
pod "nginx-f89759699-4c2cn" deleted
user@lab-server:~/projects/kubernetes$ kubectl get all -o wide
NAME                        READY   STATUS              RESTARTS   AGE     IP             NODE                 NOMINATED NODE   READINESS GATES
pod/nginx-f89759699-2psmw   1/1     Running             0          7m45s   192.168.94.9   kworker1.mylab.com   <none>           <none>
pod/nginx-f89759699-2xxdq   0/1     ContainerCreating   0          5s      <none>         kworker2.mylab.com   <none>           <none>

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE    SELECTOR
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   164m   <none>

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES   SELECTOR
deployment.apps/nginx   1/2     2            1           79m   nginx        nginx    app=nginx

NAME                              DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES   SELECTOR
replicaset.apps/nginx-f89759699   2         2         1       79m   nginx        nginx    app=nginx,pod-template-hash=f89759699

user@lab-server:~/projects/kubernetes$ kubectl get all -o wide
NAME                        READY   STATUS    RESTARTS   AGE     IP               NODE                 NOMINATED NODE   READINESS GATES
pod/nginx-f89759699-2psmw   1/1     Running   0          8m22s   192.168.94.9     kworker1.mylab.com   <none>           <none>
pod/nginx-f89759699-2xxdq   1/1     Running   0          42s     192.168.28.198   kworker2.mylab.com   <none>           <none>

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE    SELECTOR
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   165m   <none>

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES   SELECTOR
deployment.apps/nginx   2/2     2            2           80m   nginx        nginx    app=nginx

NAME                              DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES   SELECTOR
replicaset.apps/nginx-f89759699   2         2         2       80m   nginx        nginx    app=nginx,pod-template-hash=f89759699
```
NOW OUR KUBERNETES CLUSTER UPGRADED FROM V1.18.X TO v1.19.2
```
user@lab-server:~/projects/kubernetes$ kubectl get node 
NAME                 STATUS   ROLES    AGE    VERSION
kmaster.mylab.com    Ready    master   166m   v1.19.2
kworker1.mylab.com   Ready    <none>   160m   v1.19.2
kworker2.mylab.com   Ready    <none>   157m   v1.19.2
user@lab-server:~/projects/kubernetes$
```

## How it works
```kubeadm upgrade apply``` does the following:

    Checks that your cluster is in an upgradeable state:
        The API server is reachable
        All nodes are in the Ready state
        The control plane is healthy
    Enforces the version skew policies.
    Makes sure the control plane images are available or available to pull to the machine.
    Generates replacements and/or uses user supplied overwrites if component configs require version upgrades.
    Upgrades the control plane components or rollbacks if any of them fails to come up.
    Applies the new kube-dns and kube-proxy manifests and makes sure that all necessary RBAC rules are created.
    Creates new certificate and key files of the API server and backs up old files if they're about to expire in 180 days.

``` kubeadm upgrade node ``` does the following on additional control plane nodes:

    Fetches the kubeadm ClusterConfiguration from the cluster.
    Optionally backups the kube-apiserver certificate.
    Upgrades the static Pod manifests for the control plane components.
    Upgrades the kubelet configuration for this node.

``` kubeadm upgrade node``` does the following on worker nodes:

    Fetches the kubeadm ClusterConfiguration from the cluster.
    Upgrades the kubelet configuration for this node.

Conclusion

Upgrade Kubernetes cluster is a key requirement for a DevOPS in any production environment. To get the updated features of Kubernetes it is advised to up and run with an updated cluster version. By following 5 steps for each node we can complete the full upgrade process. 

* Documentation refrence : https://v1-19.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/
