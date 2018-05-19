# Deploy CephFS in Docker Swarm

Note: Due to lack of `privileged:true` containers in Swarm, there is no way to run Ceph Bluestore.

### Deployment
Consider we have 5 nodes swarm cluster:
```bash
$ docker node ls --format 'table {{.Hostname}}\t{{.ManagerStatus}}'
HOSTNAME            MANAGER STATUS
node1.domain.local  Leader
node2.domain.local  Reachable
node3.domain.local  Reachable
node4.domain.local
node5.domain.local
```
First 3 nodes are masters, and the rest are workers. Locations of corresponding Ceph roles:
- `mon` to master nodes
- `osd` to each node
- `mds` two (active/standby) anywhere
- `mgr` one anywhere

As `osd` would work in directory mode, preparing disks on each swarm node manually:
```bash
apt install xfsprogs
mkfs.xfs -f -i size=2048 /dev/sdX
echo '/dev/sdX /mnt/osd xfs rw,noatime,inode64 0 0' >> /etc/fstab
mkdir -p /mnt/osd && mount /mnt/osd
```

Generate secrets and configs for uploading to swarm. This should be done on any swarm master node via throw-away container:
```bash
docker run -d --rm --net=host \
    --name ceph_mon \
    -v `pwd`/etc:/etc/ceph \
    -v `pwd`/var:/var/lib/ceph \
    -e NETWORK_AUTO_DETECT=4 \
    -e DEBUG=verbose \
    ceph/daemon mon

docker exec -it ceph_mon ceph mon getmap -o /etc/ceph/ceph.monmap

docker stop ceph_mon
```
Need to fix main config and provide all `mon` hostnames (which are the same as swarm masters):
```ini
# cat etc/ceph.conf
[global]
fsid = 1e4d9f52-314e-49f4-a2d3-5283da875e33
mon initial members = node1, node2, node3
mon host = node1.domain.local, node2.domain.local, node3.domain.local
osd journal size = 100
log file = /dev/null
mon cluster log file = /var/lib/ceph/mon/$cluster-$id/$channel.log
```
Create secrets and configs in swarm:
```bash
docker config create ceph.conf etc/ceph.conf
docker config ls

docker secret create ceph.monmap etc/ceph.monmap
docker secret create ceph.client.admin.keyring etc/ceph.client.admin.keyring
docker secret create ceph.mon.keyring etc/ceph.mon.keyring
docker secret create ceph.bootstrap-osd.keyring var/bootstrap-osd/ceph.keyring
docker secret ls

# Cleanup
rm -r ./var ./etc
```
Deploy the stack:
```
docker stack deploy -c docker-compose.yml ceph
```
After everything is up, login to any `mon` container:
```bash
# docker exec -it `docker ps -qf name=ceph_mon` bash
# ceph -s
  cluster:
    id:     1e4d9f52-314e-49f4-a2d3-5283da875e33
    health: HEALTH_OK

  services:
    mon: 3 daemons, quorum node3,node2,node1
    mgr: node1(active)
    osd: 5 osds: 5 up, 5 in

  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 bytes
    usage:   209 MB used, 10020 MB / 10230 MB avail
    pgs:

# Configure CephFS
ceph osd pool create cephfs_data 64
ceph osd pool create cephfs_metadata 64
ceph fs new cephfs cephfs_metadata cephfs_data
# User for mounting, save this key
ceph fs authorize cephfs client.swarm / rw

# Tweak for VMs
ceph osd pool set cephfs_data nodeep-scrub 1
```


### Client Mounting
On each node specify at least 2 swarm master nodes, to mount from:
```bash
# Save the key from previous step:
echo 'AQDilPRa1BYKFxAanqbBx0JnutW4AdlYJmUehg==' > /root/.ceph
apt install ceph-fs-common
echo 'node1.domain.local,node2.domain.local:/ /mnt/ceph ceph _netdev,name=swarm,secretfile=/root/.ceph 0 0' >> /etc/fstab
mkdir /mnt/ceph && mount /mnt/ceph
```

### Basic comparison with GlusterFS
Installing GlusterFS on same 3 swarm master nodes with one replica=3 volume mounted to `/mnt/gluster` on default settings:
```
gluster volume info

Volume Name: data
Type: Replicate
Volume ID: 9a582ddc-b593-4694-921c-d5601787936d
Status: Started
Snapshot Count: 0
Number of Bricks: 1 x 3 = 3
Transport-type: tcp
Bricks:
Brick1: node1.domain.local:/var/lib/brick/data
Brick2: node2.domain.local:/var/lib/brick/data
Brick3: node3.domain.local:/var/lib/brick/data
Options Reconfigured:
transport.address-family: inet
nfs.disable: on
performance.client-io-threads: off
```

Write throughtput:
```
dd if=/dev/zero of=/mnt/gluster/test bs=1M count=100
100+0 records in
100+0 records out
104857600 bytes (105 MB, 100 MiB) copied, 2.28377 s, 45.9 MB/s

dd if=/dev/zero of=/mnt/ceph/test bs=1M count=100
100+0 records in
100+0 records out
104857600 bytes (105 MB, 100 MiB) copied, 0.0868178 s, 1.2 GB/s
```
Metadata ops (same content ~125 dirs):
```
time ls -R /mnt/gluster >/dev/null
real  0m0.101s
user  0m0.000s
sys   0m0.004s

time ls -R /mnt/ceph >/dev/null
real  0m0.004s
user  0m0.000s
sys   0m0.000s
```
