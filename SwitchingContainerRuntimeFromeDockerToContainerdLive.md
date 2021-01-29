# Swithcing Container runtime from Docker to Containerd on Kubernetes 1.20
The following document runs you through how to change the container runtime (From Docker to Containerd) on your existing kubernetes cluster with workloads running.

user@pradeep-lab-system:~/projects/kubernetes/vagrant-provisioning$ kubectl get nodes -o wide
NAME                 STATUS   ROLES                  AGE   VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION           CONTAINER-RUNTIME
kmaster.mylab.com    Ready    control-plane,master   43m   v1.20.2   172.42.42.100   <none>        CentOS Linux 7 (Core)   3.10.0-1127.el7.x86_64   docker://20.10.2
kworker1.mylab.com   Ready    <none>                 35m   v1.20.2   172.42.42.101   <none>        CentOS Linux 7 (Core)   3.10.0-1127.el7.x86_64   docker://20.10.2
kworker2.mylab.com   Ready    <none>                 28m   v1.20.2   172.42.42.102   <none>        CentOS Linux 7 (Core)   3.10.0-1127.el7.x86_64   docker://20.10.2
