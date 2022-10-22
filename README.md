# restore-etcd

```
echo "# restore-etcd" >> README.md
git init
git add README.md
git commit -m "first commit"
git branch -M main
git remote add origin https://github.com/alpha-wolf-jin/restore-etcd.git
git config --global credential.helper 'cache --timeout 7200'
git push -u origin main

git add . ; git commit -a -m "update README" ; git push -u origin main
```

**Select a control plane host to use as the recovery host - master0.ocp4.example.com

**Copy the etcd backup directory to the recovery control plane host

```
# ll /root/labs/nfs01/backup/
total 121316
-rw-------. 1 nobody nobody 124145696 Oct 22 16:14 snapshot_2022-10-22_081406.db
-rw-------. 1 nobody nobody     75109 Oct 22 16:14 static_kuberesources_2022-10-22_081406.tar.gz

[root@helper ~]# ssh core@master0
[core@master0 ~]$ mkdir backup
[core@master0 ~]$ exit

# scp /root/labs/nfs01/backup/* core@master0:~/backup/.

```

**Stop the static pods on any other control plane nodes - master1.ocp4.example.com

```
# ssh core@master1.ocp4.example.com

```

Move the existing etcd pod file out of the kubelet manifest directory:
```
[core@master1 ~]$ sudo mv /etc/kubernetes/manifests/etcd-pod.yaml /tmp
```

Verify that the etcd pods are stopped. Short while the output of this command should be empty.
```
[core@master1 ~]$ sudo crictl ps | grep etcd | grep -v operator
```

Move the existing Kubernetes API server pod file out of the kubelet manifest directory:
```
[core@master1 ~]$ sudo mv /etc/kubernetes/manifests/kube-apiserver-pod.yaml /tmp
```

Verify that the Kubernetes API server pods are stopped. It take time.  The output of this command should be empty. 
```
[core@master1 ~]$ sudo crictl ps | grep kube-apiserver | grep -v operator
```

Move the etcd data directory to a different location:
```
[core@master1 ~]$ sudo mv /var/lib/etcd/ /tmp
```

**Stop the static pods on any other control plane nodes - master2.ocp4.example.com

Same steps on master1

**Run the restore script on the recovery control plane host and pass in the path to the etcd backup directory
```
[core@master0 ~]$ sudo -E /usr/local/bin/cluster-restore.sh /home/core/backup
fe85795645d02ddfa993a7714a21a52a49e97fea687dfe7bf61f47dbf788cec6
etcdctl version: 3.5.3
API version: 3.5
Deprecated: Use `etcdutl snapshot status` instead.

{"hash":407820408,"revision":143458,"totalKey":28110,"totalSize":124145664}
...stopping kube-apiserver-pod.yaml
...stopping kube-controller-manager-pod.yaml
...stopping kube-scheduler-pod.yaml
...stopping etcd-pod.yaml
Waiting for container etcd to stop
.complete
Waiting for container etcdctl to stop
.............................complete
Waiting for container etcd-metrics to stop
complete
Waiting for container kube-controller-manager to stop
complete
Waiting for container kube-apiserver to stop
complete
Waiting for container kube-scheduler to stop
complete
Moving etcd data-dir /var/lib/etcd/member to /var/lib/etcd-backup
starting restore-etcd static pod
starting kube-apiserver-pod.yaml
static-pod-resources/kube-apiserver-pod-12/kube-apiserver-pod.yaml
starting kube-controller-manager-pod.yaml
static-pod-resources/kube-controller-manager-pod-8/kube-controller-manager-pod.yaml
starting kube-scheduler-pod.yaml
static-pod-resources/kube-scheduler-pod-8/kube-scheduler-pod.yaml
[core@master0 ~]$ 

```

**Check the nodes to ensure they are in the Ready state.
```

```
`



