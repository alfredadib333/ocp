# Installing Rook/Ceph 1.2 on Openshift 4.2
It is recommended to use dedicated compute nodes to host rook/ceph components, these nodes needs to be tainted so that they don't run any other work loads. Each storage node needs to have one or more dedicated raw disks. In the below example, we will be using the follow nodes/disks:
* storage1.ocp4.ibm.local - /dev/sdb
* storage2.ocp4.ibm.local - /dev/sdb
* storage3.ocp4.ibm.local - /dev/sdb

## Installing Rook/Ceph
1. Taint and label your compute nodes that will be dedicated for the storage:
```shell
{
oc adm taint nodes storage1.ocp4.ibm.local storage-node=yes:NoSchedule
oc adm taint nodes storage2.ocp4.ibm.local storage-node=yes:NoSchedule
oc adm taint nodes storage3.ocp4.ibm.local storage-node=yes:NoSchedule
oc label nodes storage1.ocp4.ibm.local storage-node=yes
oc label nodes storage2.ocp4.ibm.local storage-node=yes
oc label nodes storage3.ocp4.ibm.local storage-node=yes
}
```
2. Clone rook/ceph github repo:
```shell
git clone https://github.com/rook/rook
```
3. Copy the required configuration files in a working directory:
```shell
{
mkdir ~/rook-ceph
cd ~/rook-ceph
cp /root/git-projects/rook/cluster/examples/kubernetes/ceph/common.yaml ./
cp /root/git-projects/rook/cluster/examples/kubernetes/ceph/cluster.yaml ./
cp /root/git-projects/rook/cluster/examples/kubernetes/ceph/operator-openshift.yaml ./
cp /root/git-projects/rook/cluster/examples/kubernetes/ceph/csi/rbd/storageclass.yaml ./
cp /root/git-projects/rook/cluster/examples/kubernetes/ceph/csi/cephfs/storageclass.yaml ./storageclass-fs.yaml
cp /root/git-projects/rook/cluster/examples/kubernetes/ceph/filesystem.yaml ./
cp /root/git-projects/rook/cluster/examples/kubernetes/ceph/toolbox.yaml ./
}
```
4. Set the following configuration in cluster.yaml file:
* Tolerate the configured storage taint:
```yaml
  placement:
    all:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
          - matchExpressions:
            - key: storage-node
              operator: In
              values:
              - "yes"
      podAffinity:
      podAntiAffinity:
      tolerations:
      - key: storage-node
        operator: Exists
```
* Configure ceph cluster to use specific nodes/disks:
```yaml
  storage: # cluster level storage configuration and selection
    useAllNodes: false
    useAllDevices: false
```
* Set the nodes/disks that need to be used by the ceph cluster. We have assigned the storage nodes an extra 500GB raw disk to be used by ceph (/dev/sdb):
```yaml
    nodes:
    - name: "storage1.ocp4.ibm.local"
      devices: # specific devices to use for storage can be specified for each node
      - name: "sdb"
    - name: "storage2.ocp4.ibm.local"
      devices: # specific devices to use for storage can be specified for each node
      - name: "sdb"
    - name: "storage3.ocp4.ibm.local"
      devices: # specific devices to use for storage can be specified for each node
      - name: "sdb"
```
* Leave the rest as default.
5. Create the rook/ceph cluster:
```shell
{
oc create -f common.yaml
oc create -f operator-openshift.yaml
oc create -f cluster.yaml
}
```
6. Create cephfs filesystem:
```shell
oc create -f filesystem.yaml 
```
7. Create the 2 storage classess, one for the ceph RBD (block storage) and the other for cephfs:
```shell
{
oc create -f storageclass.yaml
oc create -f storageclass-fs.yaml
}
```
8. Create the toolbox deployment that can be used to ensure the ceph cluster health and perform troubleshooting (optional):
```shell
oc create -f toolbox.yaml 
``` 
## Verify the deployment of the rook/ceph cluster
* Ensure that the pods in rook-ceph project are up and running as shown below:
```shell
oc get pods -n rook-ceph 
NAME                                                              READY   STATUS      RESTARTS   AGE
csi-cephfsplugin-2lzj2                                            3/3     Running     0          70m
csi-cephfsplugin-dmfzq                                            3/3     Running     0          70m
csi-cephfsplugin-mf44r                                            3/3     Running     0          70m
csi-cephfsplugin-provisioner-d77bb49c6-bjdqs                      5/5     Running     0          70m
csi-cephfsplugin-provisioner-d77bb49c6-cmtsz                      5/5     Running     0          70m
csi-rbdplugin-4mrwj                                               3/3     Running     0          70m
csi-rbdplugin-9vc2l                                               3/3     Running     0          70m
csi-rbdplugin-hcfzf                                               3/3     Running     0          70m
csi-rbdplugin-provisioner-5b5cd64fd-hdwmv                         6/6     Running     0          70m
csi-rbdplugin-provisioner-5b5cd64fd-wgxdp                         6/6     Running     0          70m
rook-ceph-crashcollector-storage1.ocp4.ibm.local-77bc9d7f8hr5fv   1/1     Running     0          69m
rook-ceph-crashcollector-storage2.ocp4.ibm.local-58d8c8cd77kwb9   1/1     Running     0          69m
rook-ceph-crashcollector-storage3.ocp4.ibm.local-d799744f-n5gq6   1/1     Running     0          69m
rook-ceph-crashcollector-worker1.ocp4.ibm.local-f578bbf86-5d6nq   1/1     Running     0          3m8s
rook-ceph-crashcollector-worker3.ocp4.ibm.local-84b4ffc788rdjn4   1/1     Running     0          3m6s
rook-ceph-mds-myfs-a-5bff8fcdd6-bmzch                             1/1     Running     0          3m8s
rook-ceph-mds-myfs-b-759887f79-tm8sn                              1/1     Running     0          3m7s
rook-ceph-mgr-a-798fd7ccb-h8jr7                                   1/1     Running     0          69m
rook-ceph-mon-a-668456957f-9pj7b                                  1/1     Running     0          69m
rook-ceph-mon-b-74d88db866-m4f44                                  1/1     Running     0          69m
rook-ceph-mon-c-77c96dd478-s9cqb                                  1/1     Running     0          69m
rook-ceph-operator-656978cbd-fpnt2                                1/1     Running     0          73m
rook-ceph-osd-0-5875bb8fc4-8l7pg                                  1/1     Running     0          67m
rook-ceph-osd-1-6f68687959-4djg4                                  1/1     Running     0          67m
rook-ceph-osd-2-7ff7d87889-66jwd                                  1/1     Running     0          67m
rook-ceph-osd-prepare-storage1.ocp4.ibm.local-xvfd2               0/1     Completed   0          68m
rook-ceph-osd-prepare-storage2.ocp4.ibm.local-gpfhs               0/1     Completed   0          68m
rook-ceph-osd-prepare-storage3.ocp4.ibm.local-fsr2x               0/1     Completed   0          68m
rook-ceph-tools-7f96779fb9-wrt6w                                  1/1     Running     0          41m
rook-discover-5w8wp                                               1/1     Running     0          73m
rook-discover-6qwhh                                               1/1     Running     0          73m
rook-discover-drsrd                                               1/1     Running     0          73m
```
* Ensure storage class are created:
```shell
oc get sc
NAME              PROVISIONER                     AGE
rook-ceph-block   rook-ceph.rbd.csi.ceph.com      19m
rook-cephfs       rook-ceph.cephfs.csi.ceph.com   17m
sukrusc           kubernetes.io/vsphere-volume    4d16h
thin (default)    kubernetes.io/vsphere-volume    12d
```
## Verify the health of the ceph cluster
1. Check the ceph cluster health status using the toolbox pod:
```shell
oc exec -it rook-ceph-tools-7f96779fb9-wrt6w bash
bash-4.2$ ceph -s
  cluster:
    id:     c90f1d50-8ce5-453c-a4a5-48044031c2e7
    health: HEALTH_WARN
            too few PGs per OSD (8 < min 30)
 
  services:
    mon: 3 daemons, quorum a,b,c (age 28m)
    mgr: a(active, since 28m)
    osd: 3 osds: 3 up (since 27m), 3 in (since 27m)
 
  data:
    pools:   1 pools, 8 pgs
    objects: 0 objects, 0 B
    usage:   3.0 GiB used, 1.5 TiB / 1.5 TiB avail
    pgs:     8 active+clean
```
2. Check the OSD information
```shell
bash-4.2$ ceph osd status
+----+-------------------------+-------+-------+--------+---------+--------+---------+-----------+
| id |           host          |  used | avail | wr ops | wr data | rd ops | rd data |   state   |
+----+-------------------------+-------+-------+--------+---------+--------+---------+-----------+
| 0  | storage3.ocp4.ibm.local | 1025M |  497G |    0   |     0   |    0   |     0   | exists,up |
| 1  | storage1.ocp4.ibm.local | 1025M |  497G |    0   |     0   |    0   |     0   | exists,up |
| 2  | storage2.ocp4.ibm.local | 1025M |  497G |    0   |     0   |    0   |     0   | exists,up |
+----+-------------------------+-------+-------+--------+---------+--------+---------+-----------+
```
3. Check the cluster capacity utilization:
```shell
bash-4.2$ ceph df
RAW STORAGE:
    CLASS     SIZE        AVAIL       USED        RAW USED     %RAW USED 
    hdd       1.5 TiB     1.5 TiB     5.1 MiB      3.0 GiB          0.20 
    TOTAL     1.5 TiB     1.5 TiB     5.1 MiB      3.0 GiB          0.20 
 
POOLS:
    POOL            ID     STORED     OBJECTS     USED     %USED     MAX AVAIL 
    replicapool      1        0 B           0      0 B         0       473 GiB 
```
## Ensure successfull creation of PV using the created ceph storage classes
1. Provision RWO volume using Ceph RBD (rook-ceph-block)
```shell
cat << EOF > rwo-pvc.yaml 
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rwo-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: rook-ceph-block
EOF
```
```shell
oc create -f rwo-pvc.yaml 
persistentvolumeclaim/rwo-pvc created
```
Ensure the a pvc & a pv has been created and bounded:
```shell
oc get pvc
NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
rwo-pvc   Bound    pvc-75aec8fd-6230-419d-97a8-9c7fe31e4390   1Gi        RWO            rook-ceph-block   5s
``` 
Delete the created pvc:
```shell
oc delete -f rwo-pvc.yaml
```
2. Provision RWX volume using Cephfs (rook-cephfs)
```shell
cat << EOF > rwx-pvc.yaml 
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rwx-pvc
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: rook-cephfs
EOF
```
```shell
oc create -f rwx-pvc.yaml 
persistentvolumeclaim/rwx-pvc created
```
Ensure the a pvc & a pv has been created and bounded:
```shell
oc get pvc
NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
rwx-pvc   Bound    pvc-688d66b3-707c-4e22-a968-d773211ed6d6   1Gi        RWX            rook-cephfs    5s
```
Create a test app deployment that mount the created pv:
```shell
cat << EOF > test-app.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: fe-app
  name: fe-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: fe-app
  template:
    metadata:
      labels:
        app: fe-app
    spec:
      containers:
      - image: quay.io/bitnami/nginx
        name: fe-app
        ports:
          - containerPort: 80
            name: "http-server"
        volumeMounts:
          - mountPath: "/var/www/html"
            name: "fe-app-pv"
      volumes:
      - name: "fe-app-pv"
        persistentVolumeClaim:
          claimName: "rwx-pvc"
EOF
oc create -f test-app.yaml
```
Check the created pods:
```shell
oc get pods 
NAME                      READY   STATUS    RESTARTS   AGE
fe-app-58fc487956-f2vhd   1/1     Running   0          62s
fe-app-58fc487956-glmxh   1/1     Running   0          62s
fe-app-58fc487956-jfshf   1/1     Running   0          62s
```
Delete the created app deployment and pvc:
```shell
oc delete -f test-app.yaml
oc delete -f rwx-pvc.yaml
```
## Tear down
If at any point you faced an issue and want to start from scratch, you can tear down the cluster as the per the following:
```shell
{
oc delete storageclass rook-ceph-block
oc delete storageclass csi-cephfs
oc delete -f filesystem.yaml
oc delete -f toolbox.yaml
oc delete -n rook-ceph cephblockpool replicapool
oc delete -n rook-ceph cephcluster rook-ceph
oc delete -f cluster.yaml 
oc delete -f operator-openshift.yaml 
oc delete -f common.yaml 
oc adm taint nodes --all storage-node-
oc label nodes  --all storage-node-
oc get nodes|grep -v NAME|awk '{print $1}'|xargs -I arg ssh -i /workspace/ocp4-project/sshkey core@arg "sudo rm -rf /var/lib/rook"
}
```
Wipe the storage raw disks (warning: ensure you are using the correct device name maching your environment):
```shell
ssh -i /workspace/ocp4-project/sshkey core@storage1.ocp4.ibm.local "sudo wipefs -a /dev/sdb"
ssh -i /workspace/ocp4-project/sshkey core@storage2.ocp4.ibm.local "sudo wipefs -a /dev/sdb"
ssh -i /workspace/ocp4-project/sshkey core@storage3.ocp4.ibm.local "sudo wipefs -a /dev/sdb"
```
## References
* [Rook Documentation - Openshift prereqs](https://rook.io/docs/rook/v1.2/ceph-openshift.html)
* [Rook Documentation - RBD Block storage configuration](https://rook.io/docs/rook/v1.2/ceph-block.html)
* [Rook Documentation - cephfs configuration](https://rook.io/docs/rook/v1.2/ceph-filesystem.html)