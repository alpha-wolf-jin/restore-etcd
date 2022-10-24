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

**Create a project.
```
# oc new-project ocp-etcd-backup --description "Openshift Backup Automation Tool" --display-name "Backup ETCD Automation"
```

**Create Service Account
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

**Create ClusterRole
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

**Create ClusterRoleBinding
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

**Add service account to SCC "privileged"
```
# oc adm policy add-scc-to-user privileged -z openshift-backup
```

**Create Backup CronJob
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

**After creating CronJob, you can force the execution for validation with the command:
```
# oc create job backup-02 --from=cronjob/openshift-backup

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



