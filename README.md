# restore-etcd

```
echo "# restore-etcd" >> README.md
git init
git add README.md
git commit -m "first commit"
git branch -M main
git remote add origin https://github.com/alpha-wolf-jin/restore-etcd.git
git config --global credential.helper 'cache --timeout 72000'
git push -u origin main

git add . ; git commit -a -m "update README" ; git push -u origin main
```

# ETCD BACKUP

https://access.redhat.com/solutions/5843611

**Create a project**
```
# oc new-project ocp-etcd-backup --description "Openshift Backup Automation Tool" --display-name "Backup ETCD Automation"
```

**Create Service Account**
```
# cat sa-etcd-bkp.yml 
---
kind: ServiceAccount
apiVersion: v1
metadata:
  name: openshift-backup
  namespace: ocp-etcd-backup
  labels:
    app: openshift-backup

# oc apply -f sa-etcd-bkp.yml

```

**Create ClusterRole**
```
# cat cluster-role-etcd-bkp.yml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-etcd-backup
rules:
- apiGroups: [""]
  resources:
     - "nodes"
  verbs: ["get", "list"]
- apiGroups: [""]
  resources:
     - "pods"
     - "pods/log"
  verbs: ["get", "list", "create", "delete", "watch"]

# oc apply -f cluster-role-etcd-bkp.yml
```

**Create ClusterRoleBinding**
```
# cat cluster-role-binding-etcd-bkp.yml
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: openshift-backup
  labels:
    app: openshift-backup
subjects:
  - kind: ServiceAccount
    name: openshift-backup
    namespace: ocp-etcd-backup
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-etcd-backup

# oc apply -f cluster-role-binding-etcd-bkp.yml
```

**Add service account to SCC "privileged"**
```
# oc adm policy add-scc-to-user privileged -z openshift-backup
```

**Create Backup CronJob**
```
# oc apply -f cluster-role-binding-etcd-bkp.yml
clusterrolebinding.rbac.authorization.k8s.io/openshift-backup created
[root@helper etcd-backup]# cat cronjob-etcd-bkp.yml 
kind: CronJob
apiVersion: batch/v1
metadata:
  name: openshift-backup
  namespace: ocp-etcd-backup
  labels:
    app: openshift-backup
spec:
  schedule: "55 23 * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 5
  failedJobsHistoryLimit: 5
  jobTemplate:
    metadata:
      labels:
        app: openshift-backup
    spec:
      backoffLimit: 0
      template:
        metadata:
          labels:
            app: openshift-backup
        spec:
          containers:
            - name: backup
              image: "registry.redhat.io/openshift4/ose-cli"
              command:
                - "/bin/bash"
                - "-c"
                - oc get no -l node-role.kubernetes.io/master --no-headers -o name | head -n 1 |xargs -I {} -- oc debug {} -- bash -c 'chroot /host rm -rf /home/core/backup && chroot /host  mkdir /home/core/backup && chroot /host  sudo -E  mount -t nfs 192.168.9.5:/root/labs/nfs01/backup    /home/core/backup && chroot /host sudo -E /usr/local/bin/cluster-backup.sh /home/core/backup && chroot /host sudo -E find /home/core/backup/ -type f -mmin +"1" -delete'
          restartPolicy: "Never"
          terminationGracePeriodSeconds: 30
          activeDeadlineSeconds: 500
          dnsPolicy: "ClusterFirst"
          serviceAccountName: "openshift-backup"
          serviceAccount: "openshift-backup"

# oc apply -f cronjob-etcd-bkp.yml
```

**After creating CronJob, you can force the execution for validation with the command:**
```
# oc create job backup-02 --from=cronjob/openshift-backup

# oc get pod
NAME              READY   STATUS              RESTARTS   AGE
backup-02-ss8cp   0/1     ContainerCreating   0          3s

# oc logs -f backup-02-ss8cp
Starting pod/master0ocp4examplecom-debug ...
To use host binaries, run `chroot /host`
Certificate /etc/kubernetes/static-pod-certs/configmaps/etcd-serving-ca/ca-bundle.crt is missing. Checking in different directory
Certificate /etc/kubernetes/static-pod-resources/etcd-certs/configmaps/etcd-serving-ca/ca-bundle.crt found!
found latest kube-apiserver: /etc/kubernetes/static-pod-resources/kube-apiserver-pod-9
found latest kube-controller-manager: /etc/kubernetes/static-pod-resources/kube-controller-manager-pod-7
found latest kube-scheduler: /etc/kubernetes/static-pod-resources/kube-scheduler-pod-6
found latest etcd: /etc/kubernetes/static-pod-resources/etcd-pod-10
ff7030d120d70c07ca67dd2b098aba0c28a0562df9493de6609a86fa75e4795a
etcdctl version: 3.5.3
API version: 3.5
{"level":"info","ts":"2022-10-22T12:18:13.271Z","caller":"snapshot/v3_snapshot.go:65","msg":"created temporary db file","path":"/home/core/backup/snapshot_2022-10-22_121810.db.part"}
{"level":"info","ts":"2022-10-22T12:18:13.277Z","logger":"client","caller":"v3/maintenance.go:211","msg":"opened snapshot stream; downloading"}
{"level":"info","ts":"2022-10-22T12:18:13.277Z","caller":"snapshot/v3_snapshot.go:73","msg":"fetching snapshot","endpoint":"https://192.168.9.97:2379"}
{"level":"info","ts":"2022-10-22T12:18:14.152Z","logger":"client","caller":"v3/maintenance.go:219","msg":"completed snapshot read; closing"}
{"level":"info","ts":"2022-10-22T12:18:15.639Z","caller":"snapshot/v3_snapshot.go:88","msg":"fetched snapshot","endpoint":"https://192.168.9.97:2379","size":"75 MB","took":"2 seconds ago"}
{"level":"info","ts":"2022-10-22T12:18:15.661Z","caller":"snapshot/v3_snapshot.go:97","msg":"saved","path":"/home/core/backup/snapshot_2022-10-22_121810.db"}
Snapshot saved at /home/core/backup/snapshot_2022-10-22_121810.db
Deprecated: Use `etcdutl snapshot status` instead.

{"hash":1813141540,"revision":42463,"totalKey":12631,"totalSize":74629120}
snapshot db and kube resources are successfully saved to /home/core/backup

Removing debug pod ...

[root@helper etcd-backup]# ll /root/labs/nfs01/backup/
total 72960
-rw-------. 1 nobody nobody 74629152 Oct 22 20:18 snapshot_2022-10-22_121810.db
-rw-------. 1 nobody nobody    75815 Oct 22 20:18 static_kuberesources_2022-10-22_121810.tar.gz
```

# Restoring to a previous cluster state

**Select a control plane host to use as the recovery host - master0.ocp4.example.com**

**Copy the etcd backup directory to the recovery control plane host**

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

**Stop the static pods on any other control plane nodes - master1.ocp4.example.com**

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

**Stop the static pods on any other control plane nodes - master2.ocp4.example.com**

On master2, repeat the same steps

Once finish the same steps on both master1 & master2, ocp is down
```
# oc get node
Unable to connect to the server: EOF
```
**Run the restore script on the recovery control plane host and pass in the path to the etcd backup directory**
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

**Check the nodes to ensure they are in the Ready state.**

Wait for ~5 minis in lab environment

```
# oc get node
NAME                       STATUS   ROLES    AGE   VERSION
master0.ocp4.example.com   Ready    master   38h   v1.23.5+8471591
master1.ocp4.example.com   Ready    master   38h   v1.23.5+8471591
master2.ocp4.example.com   Ready    master   38h   v1.23.5+8471591
worker0.ocp4.example.com   Ready    worker   38h   v1.23.5+8471591
worker1.ocp4.example.com   Ready    worker   37h   v1.23.5+8471591

```
If any nodes are in the NotReady state, log in to the nodes and remove all of the PEM files from the /var/lib/kubelet/pki directory on each node. You can SSH into the nodes or use the terminal window in the web console.

**Restart the kubelet service on all control plane hosts**

From the recovery host, run the following command:
```
# oc get project
Error from server (ServiceUnavailable): the server is currently unable to handle the request (get projects.project.openshift.io)

[core@master0 ~]$ sudo systemctl restart kubelet.service

● kubelet.service - Kubernetes Kubelet
   Loaded: loaded (/etc/systemd/system/kubelet.service; enabled; vendor preset: disabled)
  Drop-In: /etc/systemd/system/kubelet.service.d
           └─10-mco-default-env.conf, 10-mco-default-madv.conf, 20-logging.conf, 20-nodenet.conf
   Active: active (running) since Mon 2022-10-24 06:29:20 UTC; 52s ago
  Process: 92371 ExecStartPre=/bin/rm -f /var/lib/kubelet/memory_manager_state (code=exited, status=0/SUCCESS)
  Process: 92369 ExecStartPre=/bin/rm -f /var/lib/kubelet/cpu_manager_state (code=exited, status=0/SUCCESS)
  Process: 92367 ExecStartPre=/bin/mkdir --parents /etc/kubernetes/manifests (code=exited, status=0/SUCCESS)
 Main PID: 92373 (kubelet)
    Tasks: 18 (limit: 63249)
   Memory: 158.7M
      CPU: 11.720s
   CGroup: /system.slice/kubelet.service
           └─92373 kubelet --config=/etc/kubernetes/kubelet.conf --bootstrap-kubeconfig=/etc/kubernetes/kubeconfig --kubeconfig=/var/lib/kubelet/kubeconfig --container-runtime=remote --contai>

Oct 24 06:29:45 master0.ocp4.example.com hyperkube[92373]: I1024 06:29:45.138415   92373 operation_generator.go:756] "MountVolume.SetUp succeeded for volume \"env-overrides\" (UniqueName: \"k>
Oct 24 06:29:45 master0.ocp4.example.com hyperkube[92373]: I1024 06:29:45.138587   92373 operation_generator.go:756] "MountVolume.SetUp succeeded for volume \"env-overrides\" (UniqueName: \"k>
Oct 24 06:29:45 master0.ocp4.example.com hyperkube[92373]: I1024 06:29:45.138782   92373 operation_generator.go:756] "MountVolume.SetUp succeeded for volume \"node-tuning-operator-tls\" (Uniq>
Oct 24 06:29:45 master0.ocp4.example.com hyperkube[92373]: I1024 06:29:45.138934   92373 operation_generator.go:756] "MountVolume.SetUp succeeded for volume \"config\" (UniqueName: \"kubernet>
Oct 24 06:29:45 master0.ocp4.example.com hyperkube[92373]: I1024 06:29:45.139231   92373 operation_generator.go:756] "MountVolume.SetUp succeeded for volume \"auth-proxy-config\" (UniqueName:>
Oct 24 06:29:45 master0.ocp4.example.com hyperkube[92373]: I1024 06:29:45.362506   92373 scope.go:110] "RemoveContainer" containerID="728b2156c84d690ac19f1efae4fb1bd58bd1ad6f328dd203d4bf65ff4>
Oct 24 06:29:45 master0.ocp4.example.com hyperkube[92373]: I1024 06:29:45.760636   92373 logs.go:319] "Finished parsing log file" path="/var/log/pods/openshift-machine-api_cluster-autoscaler->
Oct 24 06:29:45 master0.ocp4.example.com hyperkube[92373]: I1024 06:29:45.761098   92373 kubelet.go:2163] "SyncLoop (PLEG): event for pod" pod="openshift-machine-api/cluster-autoscaler-operat>
Oct 24 06:29:47 master0.ocp4.example.com hyperkube[92373]: I1024 06:29:47.159814   92373 reconciler.go:258] "operationExecutor.MountVolume started for volume \"kube-api-access-hq9fv\" (Unique>
Oct 24 06:29:47 master0.ocp4.example.com hyperkube[92373]: I1024 06:29:47.160550   92373 operation_generator.go:756] "MountVolume.SetUp succeeded for volume \"kube-api-access-hq9fv\" (UniqueN>
~

[core@master0 ~]$ sudo systemctl status kubelet.service
● kubelet.service - Kubernetes Kubelet
   Loaded: loaded (/etc/systemd/system/kubelet.service; enabled; vendor preset: disabled)
  Drop-In: /etc/systemd/system/kubelet.service.d
           └─10-mco-default-env.conf, 10-mco-default-madv.conf, 20-logging.conf, 20-nodenet.conf
   Active: active (running) since Mon 2022-10-24 01:57:16 UTC; 48s ago
  Process: 69675 ExecStartPre=/bin/rm -f /var/lib/kubelet/memory_manager_state (code=exited, status=0/SUCCESS)
  Process: 69673 ExecStartPre=/bin/rm -f /var/lib/kubelet/cpu_manager_state (code=exited, status=0/SUCCESS)
  Process: 69671 ExecStartPre=/bin/mkdir --parents /etc/kubernetes/manifests (code=exited, status=0/SUCCESS)
 Main PID: 69677 (kubelet)
    Tasks: 16 (limit: 63249)
   Memory: 96.0M
      CPU: 3.856s
   CGroup: /system.slice/kubelet.service
           └─69677 kubelet --config=/etc/kubernetes/kubelet.conf --bootstrap-kubeconfig=/etc/kubernetes/kubeconfig --kubeconfig=/va>

Oct 24 01:58:03 master0.ocp4.example.com hyperkube[69677]: E1024 01:58:03.817675   69677 kubelet_node_status.go:94] "Unable to regi>
Oct 24 01:58:03 master0.ocp4.example.com hyperkube[69677]: E1024 01:58:03.901279   69677 kubelet.go:2484] "Error getting node" err=>
Oct 24 01:58:04 master0.ocp4.example.com hyperkube[69677]: E1024 01:58:04.001722   69677 kubelet.go:2484] "Error getting node" err=>
Oct 24 01:58:04 master0.ocp4.example.com hyperkube[69677]: I1024 01:58:04.043672   69677 csi_plugin.go:1063] Failed to contact API >
Oct 24 01:58:04 master0.ocp4.example.com hyperkube[69677]: E1024 01:58:04.102008   69677 kubelet.go:2484] "Error getting node" err=>
Oct 24 01:58:04 master0.ocp4.example.com hyperkube[69677]: E1024 01:58:04.202126   69677 kubelet.go:2484] "Error getting node" err=>
Oct 24 01:58:04 master0.ocp4.example.com hyperkube[69677]: E1024 01:58:04.302268   69677 kubelet.go:2484] "Error getting node" err=>
Oct 24 01:58:04 master0.ocp4.example.com hyperkube[69677]: E1024 01:58:04.403049   69677 kubelet.go:2484] "Error getting node" err=>
Oct 24 01:58:04 master0.ocp4.example.com hyperkube[69677]: E1024 01:58:04.503980   69677 kubelet.go:2484] "Error getting node" err=>
Oct 24 01:58:04 master0.ocp4.example.com hyperkube[69677]: E1024 01:58:04.604650   69677 kubelet.go:2484] "Error getting node" err=>
~
~

```

**Repeat this step on all other control plane hosts.**
On master1
```
[core@master1 ~]$ sudo systemctl restart kubelet.service

[core@master1 ~]$ sudo systemctl status kubelet.service
● kubelet.service - Kubernetes Kubelet
   Loaded: loaded (/etc/systemd/system/kubelet.service; enabled; vendor preset: disabled)
  Drop-In: /etc/systemd/system/kubelet.service.d
           └─10-mco-default-env.conf, 10-mco-default-madv.conf, 20-logging.conf, 20-nodenet.conf
   Active: active (running) since Mon 2022-10-24 06:33:36 UTC; 1min 27s ago
  Process: 55858 ExecStartPre=/bin/rm -f /var/lib/kubelet/memory_manager_state (code=exited, status=0/SUCCESS)
  Process: 55855 ExecStartPre=/bin/rm -f /var/lib/kubelet/cpu_manager_state (code=exited, status=0/SUCCESS)
  Process: 55853 ExecStartPre=/bin/mkdir --parents /etc/kubernetes/manifests (code=exited, status=0/SUCCESS)
 Main PID: 55860 (kubelet)
    Tasks: 16 (limit: 63249)
   Memory: 95.4M
      CPU: 8.895s
   CGroup: /system.slice/kubelet.service
           └─55860 kubelet --config=/etc/kubernetes/kubelet.conf --bootstrap-kubeconfig=/etc/kubernetes/kubeconfig --kubeconfig=/var/lib/kubelet/kubeconfig --container-runtime=remote --contai>

Oct 24 06:34:52 master1.ocp4.example.com hyperkube[55860]: I1024 06:34:52.694068   55860 patch_prober.go:29] interesting pod/kube-apiserver-guard-master1.ocp4.example.com container/guard name>
Oct 24 06:34:52 master1.ocp4.example.com hyperkube[55860]: I1024 06:34:52.694128   55860 prober.go:121] "Probe failed" probeType="Readiness" pod="openshift-kube-apiserver/kube-apiserver-guard>
Oct 24 06:34:53 master1.ocp4.example.com hyperkube[55860]: I1024 06:34:53.962845   55860 prober.go:121] "Probe failed" probeType="Readiness" pod="openshift-etcd/etcd-quorum-guard-78f4445675-2>
Oct 24 06:34:56 master1.ocp4.example.com hyperkube[55860]: I1024 06:34:56.816989   55860 prober.go:121] "Probe failed" probeType="Readiness" pod="openshift-etcd/etcd-quorum-guard-78f4445675-2>
Oct 24 06:34:57 master1.ocp4.example.com hyperkube[55860]: I1024 06:34:57.694495   55860 patch_prober.go:29] interesting pod/kube-apiserver-guard-master1.ocp4.example.com container/guard name>
Oct 24 06:34:57 master1.ocp4.example.com hyperkube[55860]: I1024 06:34:57.694556   55860 prober.go:121] "Probe failed" probeType="Readiness" pod="openshift-kube-apiserver/kube-apiserver-guard>
Oct 24 06:34:58 master1.ocp4.example.com hyperkube[55860]: I1024 06:34:58.956798   55860 prober.go:121] "Probe failed" probeType="Readiness" pod="openshift-etcd/etcd-quorum-guard-78f4445675-2>
Oct 24 06:35:02 master1.ocp4.example.com hyperkube[55860]: I1024 06:35:02.693788   55860 patch_prober.go:29] interesting pod/kube-apiserver-guard-master1.ocp4.example.com container/guard name>
Oct 24 06:35:02 master1.ocp4.example.com hyperkube[55860]: I1024 06:35:02.693845   55860 prober.go:121] "Probe failed" probeType="Readiness" pod="openshift-kube-apiserver/kube-apiserver-guard>
Oct 24 06:35:03 master1.ocp4.example.com hyperkube[55860]: I1024 06:35:03.946958   55860 prober.go:121] "Probe failed" probeType="Readiness" pod="openshift-etcd/etcd-quorum-guard-78f4445675-2>
[core@master1 ~]$ 

[core@master1 ~]$ sudo systemctl status kubelet.service
● kubelet.service - Kubernetes Kubelet
   Loaded: loaded (/etc/systemd/system/kubelet.service; enabled; vendor preset: disabled)
  Drop-In: /etc/systemd/system/kubelet.service.d
           └─10-mco-default-env.conf, 10-mco-default-madv.conf, 20-logging.conf, 20-nodenet.conf
   Active: active (running) since Mon 2022-10-24 01:59:14 UTC; 42s ago
  Process: 114004 ExecStartPre=/bin/rm -f /var/lib/kubelet/memory_manager_state (code=exited, status=0/SUCCESS)
  Process: 114002 ExecStartPre=/bin/rm -f /var/lib/kubelet/cpu_manager_state (code=exited, status=0/SUCCESS)
  Process: 114000 ExecStartPre=/bin/mkdir --parents /etc/kubernetes/manifests (code=exited, status=0/SUCCESS)
 Main PID: 114006 (kubelet)
    Tasks: 16 (limit: 63249)
   Memory: 72.3M
      CPU: 2.155s
   CGroup: /system.slice/kubelet.service
           └─114006 kubelet --config=/etc/kubernetes/kubelet.conf --bootstrap-kubeconfig=/etc/kubernetes/kubeconfig --kubeconfig=/v>

Oct 24 01:59:55 master1.ocp4.example.com hyperkube[114006]: E1024 01:59:55.571293  114006 kubelet.go:2484] "Error getting node" err>
Oct 24 01:59:55 master1.ocp4.example.com hyperkube[114006]: E1024 01:59:55.671791  114006 kubelet.go:2484] "Error getting node" err>
Oct 24 01:59:55 master1.ocp4.example.com hyperkube[114006]: E1024 01:59:55.772523  114006 kubelet.go:2484] "Error getting node" err>
Oct 24 01:59:55 master1.ocp4.example.com hyperkube[114006]: E1024 01:59:55.873274  114006 kubelet.go:2484] "Error getting node" err>
Oct 24 01:59:55 master1.ocp4.example.com hyperkube[114006]: E1024 01:59:55.974021  114006 kubelet.go:2484] "Error getting node" err>
Oct 24 01:59:56 master1.ocp4.example.com hyperkube[114006]: E1024 01:59:56.074747  114006 kubelet.go:2484] "Error getting node" err>
Oct 24 01:59:56 master1.ocp4.example.com hyperkube[114006]: E1024 01:59:56.175450  114006 kubelet.go:2484] "Error getting node" err>
Oct 24 01:59:56 master1.ocp4.example.com hyperkube[114006]: E1024 01:59:56.276580  114006 kubelet.go:2484] "Error getting node" err>
Oct 24 01:59:56 master1.ocp4.example.com hyperkube[114006]: I1024 01:59:56.288444  114006 csi_plugin.go:1063] Failed to contact API>
Oct 24 01:59:56 master1.ocp4.example.com hyperkube[114006]: E1024 01:59:56.377582  114006 kubelet.go:2484] "Error getting node" err>
[core@master1 ~]$ 

```
On master2
```
[core@master2 ~]$ sudo systemctl restart kubelet.service

[core@master2 ~]$ sudo systemctl status kubelet.service
● kubelet.service - Kubernetes Kubelet
   Loaded: loaded (/etc/systemd/system/kubelet.service; enabled; vendor preset: disabled)
  Drop-In: /etc/systemd/system/kubelet.service.d
           └─10-mco-default-env.conf, 10-mco-default-madv.conf, 20-logging.conf, 20-nodenet.conf
   Active: active (running) since Mon 2022-10-24 06:37:00 UTC; 1min 11s ago
  Process: 110223 ExecStartPre=/bin/rm -f /var/lib/kubelet/memory_manager_state (code=exited, status=0/SUCCESS)
  Process: 110221 ExecStartPre=/bin/rm -f /var/lib/kubelet/cpu_manager_state (code=exited, status=0/SUCCESS)
  Process: 110219 ExecStartPre=/bin/mkdir --parents /etc/kubernetes/manifests (code=exited, status=0/SUCCESS)
 Main PID: 110225 (kubelet)
    Tasks: 17 (limit: 63249)
   Memory: 107.3M
      CPU: 9.849s
   CGroup: /system.slice/kubelet.service
           └─110225 kubelet --config=/etc/kubernetes/kubelet.conf --bootstrap-kubeconfig=/etc/kubernetes/kubeconfig --kubeconfig=/var/lib/kubelet/kubeconfig --container-runtime=remote --conta>

Oct 24 06:38:11 master2.ocp4.example.com hyperkube[110225]: I1024 06:38:11.942381  110225 scope.go:110] "RemoveContainer" containerID="37a332f14a9bd8df009c143980b4c254bf78a67bf0d4beb4fe55bf8c>
Oct 24 06:38:11 master2.ocp4.example.com hyperkube[110225]: I1024 06:38:11.949063  110225 kuberuntime_container.go:729] "Killing container with a grace period" pod="openshift-marketplace/cert>
Oct 24 06:38:11 master2.ocp4.example.com hyperkube[110225]: I1024 06:38:11.983026  110225 kubelet.go:2141] "SyncLoop DELETE" source="api" pods=[openshift-marketplace/redhat-marketplace-t2g47]
Oct 24 06:38:11 master2.ocp4.example.com hyperkube[110225]: I1024 06:38:11.989410  110225 kubelet.go:2135] "SyncLoop REMOVE" source="api" pods=[openshift-marketplace/redhat-marketplace-t2g47]
Oct 24 06:38:12 master2.ocp4.example.com hyperkube[110225]: I1024 06:38:12.015269  110225 scope.go:110] "RemoveContainer" containerID="37a332f14a9bd8df009c143980b4c254bf78a67bf0d4beb4fe55bf8c>
Oct 24 06:38:12 master2.ocp4.example.com hyperkube[110225]: E1024 06:38:12.015991  110225 remote_runtime.go:597] "ContainerStatus from runtime service failed" err="rpc error: code = NotFound >
Oct 24 06:38:12 master2.ocp4.example.com hyperkube[110225]: I1024 06:38:12.016038  110225 pod_container_deletor.go:52] "DeleteContainer returned error" containerID={Type:cri-o ID:37a332f14a9b>
Oct 24 06:38:12 master2.ocp4.example.com hyperkube[110225]: I1024 06:38:12.535233  110225 reconciler.go:192] "operationExecutor.UnmountVolume started for volume \"kube-api-access-2hwzb\" (Uni>
Oct 24 06:38:12 master2.ocp4.example.com hyperkube[110225]: I1024 06:38:12.545161  110225 operation_generator.go:910] UnmountVolume.TearDown succeeded for volume "kubernetes.io/projected/39dd>
Oct 24 06:38:12 master2.ocp4.example.com hyperkube[110225]: I1024 06:38:12.636464  110225 reconciler.go:300] "Volume detached for volume \"kube-api-access-2hwzb\" (UniqueName: \"kubernetes.io>
[core@master2 ~]$ 

[core@master2 ~]$ sudo systemctl status kubelet.service
● kubelet.service - Kubernetes Kubelet
   Loaded: loaded (/etc/systemd/system/kubelet.service; enabled; vendor preset: disabled)
  Drop-In: /etc/systemd/system/kubelet.service.d
           └─10-mco-default-env.conf, 10-mco-default-madv.conf, 20-logging.conf, 20-nodenet.conf
   Active: active (running) since Mon 2022-10-24 02:02:02 UTC; 6s ago
  Process: 58672 ExecStartPre=/bin/rm -f /var/lib/kubelet/memory_manager_state (code=exited, status=0/SUCCESS)
  Process: 58670 ExecStartPre=/bin/rm -f /var/lib/kubelet/cpu_manager_state (code=exited, status=0/SUCCESS)
  Process: 58668 ExecStartPre=/bin/mkdir --parents /etc/kubernetes/manifests (code=exited, status=0/SUCCESS)
 Main PID: 58674 (kubelet)
    Tasks: 17 (limit: 63249)
   Memory: 53.9M
      CPU: 898ms
   CGroup: /system.slice/kubelet.service
           └─58674 kubelet --config=/etc/kubernetes/kubelet.conf --bootstrap-kubeconfig=/etc/kubernetes/kubeconfig --kubeconfig=/va>

Oct 24 02:02:08 master2.ocp4.example.com hyperkube[58674]: I1024 02:02:08.812398   58674 kubelet_node_status.go:590] "Recording eve>
Oct 24 02:02:08 master2.ocp4.example.com hyperkube[58674]: I1024 02:02:08.812406   58674 kubelet_node_status.go:590] "Recording eve>
Oct 24 02:02:08 master2.ocp4.example.com hyperkube[58674]: I1024 02:02:08.812421   58674 kubelet_node_status.go:72] "Attempting to >
Oct 24 02:02:08 master2.ocp4.example.com hyperkube[58674]: E1024 02:02:08.813800   58674 kubelet_node_status.go:94] "Unable to regi>
Oct 24 02:02:08 master2.ocp4.example.com hyperkube[58674]: E1024 02:02:08.832904   58674 kubelet.go:2484] "Error getting node" err=>
Oct 24 02:02:08 master2.ocp4.example.com hyperkube[58674]: E1024 02:02:08.933550   58674 kubelet.go:2484] "Error getting node" err=>
Oct 24 02:02:09 master2.ocp4.example.com hyperkube[58674]: E1024 02:02:09.034487   58674 kubelet.go:2484] "Error getting node" err=>
Oct 24 02:02:09 master2.ocp4.example.com hyperkube[58674]: E1024 02:02:09.134960   58674 kubelet.go:2484] "Error getting node" err=>
Oct 24 02:02:09 master2.ocp4.example.com hyperkube[58674]: W1024 02:02:09.177819   58674 reflector.go:324] k8s.io/client-go/informe>
Oct 24 02:02:09 master2.ocp4.example.com hyperkube[58674]: E1024 02:02:09.177848   58674 reflector.go:138] k8s.io/client-go/informe>
[core@master2 ~]$ 

```

**Approve the pending CSRs (if needed):**
```
# oc get node
NAME                       STATUS     ROLES    AGE   VERSION
master0.ocp4.example.com   NotReady   master   38h   v1.23.5+8471591
master1.ocp4.example.com   NotReady   master   38h   v1.23.5+8471591
master2.ocp4.example.com   NotReady   master   38h   v1.23.5+8471591
worker0.ocp4.example.com   NotReady   worker   38h   v1.23.5+8471591
worker1.ocp4.example.com   NotReady   worker   38h   v1.23.5+8471591

[root@helper ~]# oc get pod -n openshift-etcd -o wide
NAME                                          READY   STATUS      RESTARTS   AGE    IP             NODE                       NOMINATED NODE   READINESS GATES
etcd-master0.ocp4.example.com                 1/1     Running     0          14m    192.168.9.97   master0.ocp4.example.com   <none>           <none>
etcd-quorum-guard-78f4445675-2jnj2            0/1     Running     1          2d2h   192.168.9.98   master1.ocp4.example.com   <none>           <none>
etcd-quorum-guard-78f4445675-67qtx            1/1     Running     1          2d2h   192.168.9.97   master0.ocp4.example.com   <none>           <none>
etcd-quorum-guard-78f4445675-lvckp            0/1     Running     1          2d2h   192.168.9.99   master2.ocp4.example.com   <none>           <none>
installer-10-master0.ocp4.example.com         0/1     Completed   0          2d2h   10.128.0.81    master0.ocp4.example.com   <none>           <none>
installer-10-master1.ocp4.example.com         0/1     Completed   0          2d2h   10.129.0.41    master1.ocp4.example.com   <none>           <none>
installer-10-master2.ocp4.example.com         0/1     Completed   0          2d2h   10.130.0.29    master2.ocp4.example.com   <none>           <none>
installer-11-master0.ocp4.example.com         0/1     Completed   0          2d2h   10.128.0.84    master0.ocp4.example.com   <none>           <none>
installer-11-master1.ocp4.example.com         0/1     Completed   0          2d2h   10.129.0.48    master1.ocp4.example.com   <none>           <none>
installer-11-master2.ocp4.example.com         0/1     Completed   0          2d2h   10.130.0.34    master2.ocp4.example.com   <none>           <none>
installer-7-master0.ocp4.example.com          0/1     Completed   0          2d2h   10.128.0.68    master0.ocp4.example.com   <none>           <none>
installer-7-master1.ocp4.example.com          0/1     Completed   0          2d2h   10.129.0.27    master1.ocp4.example.com   <none>           <none>
installer-9-master0.ocp4.example.com          0/1     Completed   0          2d2h   10.128.0.72    master0.ocp4.example.com   <none>           <none>
installer-9-master2.ocp4.example.com          0/1     Completed   0          2d2h   10.130.0.20    master2.ocp4.example.com   <none>           <none>
revision-pruner-10-master0.ocp4.example.com   0/1     Completed   0          2d2h   10.128.0.76    master0.ocp4.example.com   <none>           <none>
revision-pruner-10-master1.ocp4.example.com   0/1     Completed   0          2d2h   10.129.0.40    master1.ocp4.example.com   <none>           <none>
revision-pruner-10-master2.ocp4.example.com   0/1     Completed   0          2d2h   10.130.0.23    master2.ocp4.example.com   <none>           <none>
revision-pruner-11-master0.ocp4.example.com   0/1     Completed   0          2d2h   10.128.0.83    master0.ocp4.example.com   <none>           <none>
revision-pruner-11-master1.ocp4.example.com   0/1     Completed   0          2d2h   10.129.0.47    master1.ocp4.example.com   <none>           <none>
revision-pruner-11-master2.ocp4.example.com   0/1     Completed   0          2d2h   10.130.0.33    master2.ocp4.example.com   <none>           <none>
revision-pruner-9-master0.ocp4.example.com    0/1     Completed   0          2d2h   10.128.0.74    master0.ocp4.example.com   <none>           <none>
revision-pruner-9-master1.ocp4.example.com    0/1     Completed   0          2d2h   10.129.0.37    master1.ocp4.example.com   <none>           <none>
revision-pruner-9-master2.ocp4.example.com    0/1     Completed   0          2d2h   10.130.0.19    master2.ocp4.example.com   <none>           <none>


```

**Verify that the single member control plane has started successfully.**

From the recovery host, verify that the etcd container is running.

```
[core@master0 ~]$ sudo crictl ps | grep etcd | grep -v operator
0c7adf1fe89dd       76f536e1fd2c0c776f71232f0318b4995c54beb708add1f958ad4924acd130b3                                                             23 minutes ago      Running             etcd                                          0                   4bcb94fbcac74
[core@master0 ~]$ 

[root@helper ~]# oc get pods -n openshift-etcd | grep -v etcd-quorum-guard | grep etcd
etcd-master0.ocp4.example.com                 1/1     Running     0          20m

```


```
[core@master1 ~]$ sudo crictl ps | grep etcd | grep -v operator
[core@master1 ~]$ 
[core@master2 ~]$ sudo crictl ps | grep etcd | grep -v operator
[core@master2 ~]$ 

```
