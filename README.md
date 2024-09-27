
# Configuring Rook with an external Ceph

Storage backend for Kubernetes or disaster recovery

#### UseCase 1: As a storage backend

Please be advised that the Rook-Ceph operator is employed to establish a Ceph cluster with a single click. However, this is presuming that you wish to use your current standalone Ceph cluster as the storage backend for your Kubernetes cluster.

Potential Drawback: Performance may be impacted by accessing storage from an external cluster when dealing with heavy loads, such as databases.


#### UseCase 2: For Disaster Recovery

Even if your rook-ceph cluster is destroyed (particularly if your mon data is on the k8s host -/var/lib/rook), you can restore the PV and PVC to another cluster and obtain the data.


## Configure Rook-Ceph to use External Ceph Cluster

First, let's build an all-in-one Ceph cluster using cephadm for testing purpose on Macbook Air M3 using UTM.

```bash
VM spec.
Hostname: ceph01
OS: Ubuntu 24.04
IP: 192.168.0.30
2 vCPU
2G RAM
20GB vda (OS disk)
10GB vdb
10GB vdc
10GB vdd

```
Note:- The cluster will be configured with no replica because cluster has single node only.

#### build steps
Download cephadmin

```bash
root@ceph01:~# curl --silent --remote-name --location https://github.com/ceph/ceph/raw/quincy/src/cephadm/cephadm

root@ceph01:~# chmod +x cephadm
root@ceph01:~# ls -l
total 372
-rwxr-xr-x 1 root root 378585 Sep 26 22:22 cephadm
root@ceph01:~#
```
 Install docker

```bash
root@ceph01:~# sudo apt-get install ca-certificates curl -y
root@ceph01:~# sudo install -m 0755 -d /etc/apt/keyrings
root@ceph01:~# sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
root@ceph01:~# sudo chmod a+r /etc/apt/keyrings/docker.asc
root@ceph01:~# echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
root@ceph01:~# sudo apt-get update -y
root@ceph01:~# sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
root@ceph01:~# docker --version
Docker version 27.3.1, build ce12230
root@ceph01:~# 
```
Add ceph repo
```bash
root@ceph01:~# ./cephadm add-repo --release squid
```
After this i start getting below error:
```bash
Installing repo GPG key from https://download.ceph.com/keys/release.gpg...
Installing repo file at /etc/apt/sources.list.d/ceph.list...
Updating package list...
Non-zero exit code 100 from apt-get update
apt-get: stdout Hit:1 http://ports.ubuntu.com/ubuntu-ports noble InRelease
apt-get: stdout Hit:2 https://download.docker.com/linux/ubuntu noble InRelease
apt-get: stdout Hit:3 http://ports.ubuntu.com/ubuntu-ports noble-updates InRelease
apt-get: stdout Hit:4 http://ports.ubuntu.com/ubuntu-ports noble-backports InRelease
apt-get: stdout Hit:5 http://ports.ubuntu.com/ubuntu-ports noble-security InRelease
apt-get: stdout Ign:6 https://download.ceph.com/debian-squid noble InRelease
apt-get: stdout Err:7 https://download.ceph.com/debian-squid noble Release
apt-get: stdout   404  Not Found [IP: 2607:5300:201:2000::3:58a1 443]
apt-get: stdout Reading package lists...
apt-get: stderr E: The repository 'https://download.ceph.com/debian-squid noble Release' does not have a Release file.
Traceback (most recent call last):
  File "/usr/local/bin/cephadm", line 9930, in <module>
    main()
  File "/usr/local/bin/cephadm", line 9918, in main
    r = ctx.func(ctx)
        ^^^^^^^^^^^^^
  File "/usr/local/bin/cephadm", line 8354, in command_add_repo
    pkg.add_repo()
  File "/usr/local/bin/cephadm", line 7976, in add_repo
    self.update()
  File "/usr/local/bin/cephadm", line 7997, in update
    call_throws(self.ctx, ['apt-get', 'update'])
  File "/usr/local/bin/cephadm", line 1886, in call_throws
    raise RuntimeError(f'Failed command: {" ".join(command)}: {s}')
RuntimeError: Failed command: apt-get update: E: The repository 'https://download.ceph.com/debian-squid noble Release' does not have a Release file.

root@ceph01:~#

```

To solve this please modifie the below file

```bash
From:-

root@ceph01:/usr/bin# cat /etc/apt/sources.list.d/ceph.list
deb https://download.ceph.com/debian-squid/ noble main
root@ceph01:/usr/bin# 

To:-

root@ceph01:/usr/bin# cat /etc/apt/sources.list.d/ceph.list
deb https://download.ceph.com/debian-squid/ jammy main
root@ceph01:/usr/bin# 
```
Install cephadm and ceph-common
```bash
root@ceph01:~# cephadm install
Installing packages ['cephadm']...
root@ceph01:~# 
root@ceph01:~# which cephadm
/usr/local/bin/cephadm
root@ceph01:~# cephadm install ceph-common
Installing packages ['ceph-common']...
root@ceph01:~# ceph -v
ceph version 19.2.0~git20240301.4c76c50 (4c76c50a73f63ba48ccdf0adccce03b00d1d80c7) squid (dev)
root@ceph01:~# 
```
#### Bootstrap ceph
```bash
root@ceph01:~# sudo mkdir -p /etc/ceph
root@ceph01:~# cephadm bootstrap --mon-ip 192.168.0.30
```

```bash
root@ceph01:~# ceph orch ps
NAME                  HOST    PORTS             STATUS         REFRESHED  AGE  MEM USE  MEM LIM  VERSION  IMAGE ID      CONTAINER ID  
alertmanager.ceph01   ceph01  *:9093,9094       running (30s)    23s ago   2m    13.5M        -  0.25.0   4d8d4d8334be  5d3392523601  
crash.ceph01          ceph01                    running (2m)     23s ago   2m    7216k        -  17.2.7   d1dd68cd2105  3cf0bc2085a6  
grafana.ceph01        ceph01  *:3000            running (29s)    23s ago  64s    58.8M        -  9.4.7    e7d6ac5c7ccd  85bf51d84422  
mgr.ceph01.xieofu     ceph01  *:9283,8765,8443  running (2m)     23s ago   2m     453M        -  17.2.7   d1dd68cd2105  110588110fd4  
mon.ceph01            ceph01                    running (2m)     23s ago   2m    28.2M    2048M  17.2.7   d1dd68cd2105  e8fd27d3af28  
node-exporter.ceph01  ceph01  *:9100            running (2m)     23s ago   2m    7631k        -  1.5.0    68cb0c05b3f2  6befc4ba6c6b  
prometheus.ceph01     ceph01  *:9095            running (40s)    23s ago  40s    24.4M        -  2.43.0   77ee200e57dc  ea5f7fb0f9f6  
root@ceph01:~# 

```

```bash
root@ceph01:~# ceph -s
  cluster:
    id:     0640e0d6-7c49-11ef-bfe8-26f3c4275220
    health: HEALTH_WARN
            mon ceph01 is low on available space
            OSD count 0 < osd_pool_default_size 3
 
  services:
    mon: 1 daemons, quorum ceph01 (age 3m)
    mgr: ceph01.xieofu(active, since 45s)
    osd: 0 osds: 0 up, 0 in
 
  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:     
 
root@ceph01:~# 
```
#### Add OSDs
```bash
root@ceph01:~# lsblk 
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sr0                        11:0    1 1024M  0 rom  
vda                       253:0    0   20G  0 disk 
├─vda1                    253:1    0  953M  0 part /boot/efi
├─vda2                    253:2    0  1.8G  0 part /boot
└─vda3                    253:3    0 17.3G  0 part 
  └─ubuntu--vg-ubuntu--lv 252:0    0   10G  0 lvm  /
vdb                       253:16   0   10G  0 disk 
vdc                       253:32   0   10G  0 disk 
vdd                       253:48   0   10G  0 disk 
root@ceph01:~# 

root@ceph01:~# ceph orch device ls
HOST    PATH      TYPE  DEVICE ID   SIZE  AVAILABLE  REFRESHED  REJECT REASONS  
ceph01  /dev/vdb  hdd              10.0G  Yes        2m ago                     
ceph01  /dev/vdc  hdd              10.0G  Yes        2m ago                     
ceph01  /dev/vdd  hdd              10.0G  Yes        2m ago                     
root@ceph01:~# 

root@ceph01:~# ceph orch apply osd --all-available-devices
Scheduled osd.all-available-devices update...
root@ceph01:~# 
```

```bash
root@ceph01:~# ceph -s
  cluster:
    id:     0640e0d6-7c49-11ef-bfe8-26f3c4275220
    health: HEALTH_WARN
            mon ceph01 is low on available space
 
  services:
    mon: 1 daemons, quorum ceph01 (age 4m)
    mgr: ceph01.xieofu(active, since 2m)
    osd: 3 osds: 3 up (since 35s), 3 in (since 50s)
 
  data:
    pools:   1 pools, 1 pgs
    objects: 0 objects, 0 B
    usage:   871 MiB used, 29 GiB / 30 GiB avail
    pgs:     100.000% pgs not active
             1 undersized+peered
 
root@ceph01:~#
```

```bash
root@ceph01:~# ceph osd tree
ID  CLASS  WEIGHT   TYPE NAME        STATUS  REWEIGHT  PRI-AFF
-1         0.02939  root default                              
-3         0.02939      host ceph01                           
 0    hdd  0.00980          osd.0        up   1.00000  1.00000
 1    hdd  0.00980          osd.1        up   1.00000  1.00000
 2    hdd  0.00980          osd.2        up   1.00000  1.00000
root@ceph01:~# 
root@ceph01:~# 
```

```bash
root@ceph01:~# ceph osd df
ID  CLASS  WEIGHT   REWEIGHT  SIZE    RAW USE  DATA     OMAP  META     AVAIL    %USE  VAR   PGS  STATUS
 0    hdd  0.00980   1.00000  10 GiB  290 MiB  288 KiB   0 B  290 MiB  9.7 GiB  2.84  1.00   22      up
 1    hdd  0.00980   1.00000  10 GiB  291 MiB  740 KiB   0 B  290 MiB  9.7 GiB  2.84  1.00   25      up
 2    hdd  0.00980   1.00000  10 GiB  290 MiB  292 KiB   0 B  290 MiB  9.7 GiB  2.84  1.00   18      up
                       TOTAL  30 GiB  872 MiB  1.3 MiB   0 B  870 MiB   29 GiB  2.84                   
MIN/MAX VAR: 1.00/1.00  STDDEV: 0.00
root@ceph01:~# 
```

```bash
root@ceph01:~# docker ps
CONTAINER ID   IMAGE                                     COMMAND                  CREATED        STATUS        PORTS     NAMES
672c764d663b   quay.io/ceph/ceph                         "/usr/bin/ceph-osd -…"   18 hours ago   Up 18 hours             ceph-0640e0d6-7c49-11ef-bfe8-26f3c4275220-osd-2
695a4ff5e828   quay.io/ceph/ceph                         "/usr/bin/ceph-osd -…"   18 hours ago   Up 18 hours             ceph-0640e0d6-7c49-11ef-bfe8-26f3c4275220-osd-1
4440207a078b   quay.io/ceph/ceph                         "/usr/bin/ceph-osd -…"   18 hours ago   Up 18 hours             ceph-0640e0d6-7c49-11ef-bfe8-26f3c4275220-osd-0
85bf51d84422   quay.io/ceph/ceph-grafana:9.4.7           "/bin/sh -c 'grafana…"   18 hours ago   Up 18 hours             ceph-0640e0d6-7c49-11ef-bfe8-26f3c4275220-grafana-ceph01
5d3392523601   quay.io/prometheus/alertmanager:v0.25.0   "/bin/alertmanager -…"   18 hours ago   Up 18 hours             ceph-0640e0d6-7c49-11ef-bfe8-26f3c4275220-alertmanager-ceph01
ea5f7fb0f9f6   quay.io/prometheus/prometheus:v2.43.0     "/bin/prometheus --c…"   18 hours ago   Up 18 hours             ceph-0640e0d6-7c49-11ef-bfe8-26f3c4275220-prometheus-ceph01
6befc4ba6c6b   quay.io/prometheus/node-exporter:v1.5.0   "/bin/node_exporter …"   18 hours ago   Up 18 hours             ceph-0640e0d6-7c49-11ef-bfe8-26f3c4275220-node-exporter-ceph01
3cf0bc2085a6   quay.io/ceph/ceph                         "/usr/bin/ceph-crash…"   18 hours ago   Up 18 hours             ceph-0640e0d6-7c49-11ef-bfe8-26f3c4275220-crash-ceph01
110588110fd4   quay.io/ceph/ceph:v17                     "/usr/bin/ceph-mgr -…"   18 hours ago   Up 18 hours             ceph-0640e0d6-7c49-11ef-bfe8-26f3c4275220-mgr-ceph01-xieofu
e8fd27d3af28   quay.io/ceph/ceph:v17                     "/usr/bin/ceph-mon -…"   18 hours ago   Up 18 hours             ceph-0640e0d6-7c49-11ef-bfe8-26f3c4275220-mon-ceph01
root@ceph01:~# 
```
a few minutes later, the ceph status become HEALTH_WARN
```bash
root@ceph01:~# ceph -s
  cluster:
    id:     0640e0d6-7c49-11ef-bfe8-26f3c4275220
    health: HEALTH_WARN
            mon ceph01 is low on available space
 
  services:
    mon: 1 daemons, quorum ceph01 (age 4m)
    mgr: ceph01.xieofu(active, since 2m)
    osd: 3 osds: 3 up (since 37s), 3 in (since 52s)
 
  data:
    pools:   1 pools, 1 pgs
    objects: 0 objects, 0 B
    usage:   871 MiB used, 29 GiB / 30 GiB avail
    pgs:     100.000% pgs not active
             1 undersized+peered
 
root@ceph01:~#

```
This is due to a pool has default size 3 and min_size 2. But the all-in-one cluster is only able to store single replica.

```bash
root@ceph01:~# ceph osd dump | grep 'replicated size'
pool 1 '.mgr' replicated size 3 min_size 2 crush_rule 0 object_hash rjenkins pg_num 1 pgp_num 1 autoscale_mode on last_change 14 flags hashpspool stripe_width 0 pg_num_max 32 pg_num_min 1 application mgr
root@ceph01:~#
```
#### Configure cluster to allow pool size one
```bash
root@ceph01:~# ceph config set global mon_allow_pool_size_one true
root@ceph01:~# 
```
set the '.mgr' pool size to 1 (set size 2 first and then 1)

```bash
root@ceph01:~# ceph osd dump | grep 'replicated size'
pool 1 '.mgr' replicated size 3 min_size 2 crush_rule 0 object_hash rjenkins pg_num 1 pgp_num 1 autoscale_mode on last_change 14 flags hashpspool stripe_width 0 pg_num_max 32 pg_num_min 1 application mgr

root@ceph01:~# ceph config set global mon_allow_pool_size_one true

root@ceph01:~# ceph osd dump | grep 'replicated size'
pool 1 '.mgr' replicated size 3 min_size 2 crush_rule 0 object_hash rjenkins pg_num 1 pgp_num 1 autoscale_mode on last_change 14 flags hashpspool stripe_width 0 pg_num_max 32 pg_num_min 1 application mgr
root@ceph01:~# 

root@ceph01:~# ceph osd pool set .mgr size 2 --yes-i-really-mean-it
set pool 1 size to 2
root@ceph01:~# 

root@ceph01:~# ceph osd dump | grep 'replicated size'
pool 1 '.mgr' replicated size 2 min_size 1 crush_rule 0 object_hash rjenkins pg_num 1 pgp_num 1 autoscale_mode on last_change 15 flags hashpspool stripe_width 0 pg_num_max 32 pg_num_min 1 application mgr
root@ceph01:~# 

root@ceph01:~# ceph osd pool set .mgr size 1 --yes-i-really-mean-it
set pool 1 size to 1
root@ceph01:~# 

root@ceph01:~# ceph osd dump | grep 'replicated size'
pool 1 '.mgr' replicated size 1 min_size 1 crush_rule 0 object_hash rjenkins pg_num 1 pgp_num 1 autoscale_mode on last_change 17 flags hashpspool stripe_width 0 pg_num_max 32 pg_num_min 1 application mgr
root@ceph01:~# 
 
root@ceph01:~# ceph osd pool set .mgr min_size 1
set pool 1 min_size to 1
root@ceph01:~# 
```
#### create "rbd" pool
```bash
root@ceph01:~# ceph osd pool create rbd 64 64 replicated
pool 'rbd' created
root@ceph01:~# 

```
set size to 1
```bash
root@ceph01:~# ceph osd pool set rbd min_size 1
set pool 2 min_size to 1
root@ceph01:~# ceph osd pool set rbd size 2 --yes-i-really-mean-it
set pool 2 size to 2
root@ceph01:~# ceph osd pool set rbd size 1 --yes-i-really-mean-it
set pool 2 size to 1
root@ceph01:~#
```
config pool for RBD use
```bash
root@ceph01:~# rbd pool init rbd
root@ceph01:~# ceph osd pool application enable rbd rbd
enabled application 'rbd' on pool 'rbd'
root@ceph01:~# 
root@ceph01:~#
```
Ignore the no replicas warning
```bash
root@ceph01:~# ceph -s
  cluster:
    id:     0640e0d6-7c49-11ef-bfe8-26f3c4275220
    health: HEALTH_WARN
            mon ceph01 is low on available space
            2 pool(s) have no replicas configured
 
  services:
    mon: 1 daemons, quorum ceph01 (age 8m)
    mgr: ceph01.xieofu(active, since 6m)
    osd: 3 osds: 3 up (since 4m), 3 in (since 4m)
 
  data:
    pools:   2 pools, 65 pgs
    objects: 3 objects, 449 KiB
    usage:   872 MiB used, 29 GiB / 30 GiB avail
    pgs:     65 active+clean
 
root@ceph01:~#
```

```bash
root@ceph01:~# ceph health mute POOL_NO_REDUNDANCY
root@ceph01:~# 
root@ceph01:~# ceph -s
  cluster:
    id:     0640e0d6-7c49-11ef-bfe8-26f3c4275220
    health: HEALTH_WARN
            mon ceph01 is low on available space
            (muted: POOL_NO_REDUNDANCY)
 
  services:
    mon: 1 daemons, quorum ceph01 (age 8m)
    mgr: ceph01.xieofu(active, since 6m)
    osd: 3 osds: 3 up (since 4m), 3 in (since 5m)
 
  data:
    pools:   2 pools, 65 pgs
    objects: 3 objects, 449 KiB
    usage:   872 MiB used, 29 GiB / 30 GiB avail
    pgs:     65 active+clean
 
root@ceph01:~#
```
#### test
```bash
root@ceph01:~# rbd create rbd0 --size 1024  --image-feature layering
root@ceph01:~# 
root@ceph01:~# rbd ls -l
NAME  SIZE   PARENT  FMT  PROT  LOCK
rbd0  1 GiB            2            
root@ceph01:~# 
```

#### other configs
to allow pool deletion
```bash
ceph config set global mon_allow_pool_delete true
```

# Install Rook operator


```bash
shashank@Mac Ceph % git clone --single-branch --branch v1.15.2 https://github.com/rook/rook.git
shashank@Mac rook % cd /rook/deploy/examples 
shashank@Mac examples % 
shashank@Mac examples % kubectl create -f crds.yaml -f common.yaml -f operator.yaml
```
Get the FSID and other details needed for the helper script from the ceph cluster
```bash
ceph health
ceph fsid
ceph auth get-key client.admin
ceph status (to get monitor ip/hostname)
```
Set the env variables
```bash
shashank@Mac examples % export NAMESPACE=rook-ceph-external
shashank@Mac examples % export ROOK_EXTERNAL_FSID=0640e0d6-7c49-11ef-bfe8-26f3c4275220
shashank@Mac examples % export ROOK_EXTERNAL_ADMIN_SECRET=AQC2yfVmgBBvLhAAcdTqbh9HOAXPxaNeFTfKMg==
shashank@Mac examples % export ROOK_EXTERNAL_CEPH_MON_DATA=a=192.168.0.30:6789
```
Please test if there is connectivity from your K8s cluster to Ceph cluster
```bash
root@master01:~# 
root@master01:~# cat < /dev/tcp/192.168.0.30/3300
ceph v2
^C
root@master01:~#
```
After setting the above environment variables apply the common-external.yaml to create the rook-ceph-external namespace and other RBAC bindings
```bash
shashank@Mac examples % kubectl create -f common-external.yaml                     
namespace/rook-ceph-external created
rolebinding.rbac.authorization.k8s.io/rook-ceph-cluster-mgmt created
rolebinding.rbac.authorization.k8s.io/rook-ceph-cmd-reporter created
serviceaccount/rook-ceph-cmd-reporter created
serviceaccount/rook-ceph-default created
role.rbac.authorization.k8s.io/rook-ceph-cmd-reporter created
shashank@Mac examples % 
```
Use the helper to create the configmaps and secrets from env data
```bash
shashank@Mac examples % bash import-external-cluster.sh 
cluster namespace rook-ceph-external already exists
secret/rook-ceph-mon created
configmap/rook-ceph-mon-endpoints created
configmap/external-cluster-user-command created
shashank@Mac examples % 
```
#### Rook will complain about empty ceph user name follow this workaround https://github.com/rook/rook/issues/5732#issuecomment-756042524 before doing the next step (if the bug is still open)

remove the empty fields ceph-secret and ceph-username: (do this before applying cluster-external.yaml)
```bash
shashank@Mac-398 .kube % kubectl edit secret -n rook-ceph-external             
secret/rook-ceph-mon edited
shashank@Mac-398 .kube % 

```
After this create the external cluster
```bash
shashank@Mac examples % kubectl create -f cluster-external.yaml                    
cephcluster.ceph.rook.io/rook-ceph-external created
shashank@Mac examples %
```
Check if the cluster is Connected

```bash
shashank@Mac examples % kubectl get cephcluster -A                                                
NAMESPACE            NAME                 DATADIRHOSTPATH   MONCOUNT   AGE   PHASE       MESSAGE                          HEALTH        EXTERNAL   FSID
rook-ceph-external   rook-ceph-external                                7s    Connected   Cluster connected successfully   HEALTH_WARN   true       0640e0d6-7c49-11ef-bfe8-26f3c4275220
shashank@Mac examples %

```
Before you create a Storage Class a pool has to be created in the external ceph cluster

```bash
root@ceph01:~# rados df
POOL_NAME     USED  OBJECTS  CLONES  COPIES  MISSING_ON_PRIMARY  UNFOUND  DEGRADED  RD_OPS      RD  WR_OPS       WR  USED COMPR  UNDER COMPR
.mgr       452 KiB        2       0       2                   0        0         0     106  91 KiB     127  1.6 MiB         0 B          0 B
rbd          8 KiB        4       0       4                   0        0         0      37  29 KiB       5    5 KiB         0 B          0 B

total_objects    6
total_used       872 MiB
total_avail      29 GiB
total_space      30 GiB
root@ceph01:~# 

```
create a test Ceph pool in external Ceph limited to 1 GB.

```bash
root@ceph01:~# ceph osd pool create test 128
pool 'test' created
root@ceph01:~# 
root@ceph01:~# ceph osd lspools
1 .mgr
2 rbd
3 test
root@ceph01:~# ceph osd pool set test size 1 --yes-i-really-mean-it
set pool 3 size to 1
root@ceph01:~# 
root@ceph01:~# 
root@ceph01:~# ceph osd pool stats
pool .mgr id 1
  nothing is going on

pool rbd id 2
  nothing is going on

pool test id 3
  nothing is going on
root@ceph01:~#
```
Create a StorageClass
```bash
shashank@Mac-398 Ceph % kubectl create -f storage-class.yaml 
storageclass.storage.k8s.io/rook-ceph-block-ext created
shashank@Mac-398 Ceph %
```
Create a PersistentVolumeClaim
```bash
shashank@Mac-398 Ceph % kubectl create -f pv.yaml
persistentvolumeclaim/ceph-ext created
shashank@Mac-398 Ceph % 
```
see the Used size increase in your external Ceph pool
```bash
root@ceph01:~# rados df
POOL_NAME     USED  OBJECTS  CLONES  COPIES  MISSING_ON_PRIMARY  UNFOUND  DEGRADED  RD_OPS      RD  WR_OPS       WR  USED COMPR  UNDER COMPR
.mgr       452 KiB        2       0       2                   0        0         0     106  91 KiB     127  1.6 MiB         0 B          0 B
rbd          8 KiB        4       0       4                   0        0         0      37  29 KiB       5    5 KiB         0 B          0 B
test         8 KiB        6       0       6                   0        0         0      33  26 KiB       8    8 KiB         0 B          0 B

total_objects    12
total_used       893 MiB
total_avail      29 GiB
total_space      30 GiB
root@ceph01:~# 
```
let’s use the PV in a Pod
```bash
shashank@Mac-398 Ceph % kubectl create -f test-app.yaml 
pod/nginx-test created
shashank@Mac-398 Ceph % 
```
Login to the Pod and add some data to the PersistentVolumeClaim
```bash
shashank@Mac-398 Ceph % kubectl -n test-apps exec -it nginx-test -- bash  
root@nginx-test:/# echo 'Hello Test Ceph External Storage' >> usr/share/nginx/html/index.html
root@nginx-test:/# curl http://localhost/
Hello Test Ceph External Storage
root@nginx-test:/# 
root@nginx-test:/# 
```
You can see that the external cluster has increased
```bash
root@ceph01:~# rados df
POOL_NAME     USED  OBJECTS  CLONES  COPIES  MISSING_ON_PRIMARY  UNFOUND  DEGRADED  RD_OPS       RD  WR_OPS       WR  USED COMPR  UNDER COMPR
.mgr       452 KiB        2       0       2                   0        0         0     106   91 KiB     127  1.6 MiB         0 B          0 B
rbd          8 KiB        4       0       4                   0        0         0      37   29 KiB       5    5 KiB         0 B          0 B
test       616 KiB       13       0      13                   0        0         0     455  4.0 MiB      34  664 KiB         0 B          0 B

total_objects    19
total_used       894 MiB
total_avail      29 GiB
total_space      30 GiB
root@ceph01:~# 
```


## Authors

[@shashank-linkedin](https://linkedin.com/in/shashank-sharma-137002124)

