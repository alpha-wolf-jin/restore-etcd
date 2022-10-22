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
