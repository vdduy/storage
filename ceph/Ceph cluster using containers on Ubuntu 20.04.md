My cluster
```
cat <<EOF >> /etc/hosts
172.16.14.111 ceph-mon01
172.16.14.112 ceph-mon02
172.16.14.113 ceph-mon03
172.16.14.114 ceph-osd01
172.16.14.115 ceph-osd02
172.16.14.116 ceph-osd03
EOF
```
```
sudo apt update && sudo apt -y upgrade && sudo systemctl reboot
```

**Install Podman on all servers**
```
. /etc/os-release
echo "deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/xUbuntu_${VERSION_ID}/ /" | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/xUbuntu_${VERSION_ID}/Release.key | sudo apt-key add -
sudo apt-get update
sudo apt-get -y upgrade
sudo apt-get -y install podman
```

**Install Python 3**
Add the deadsnakes PPA to your system's source list:
```
sudo add-apt-repository ppa:deadsnakes/ppa
```
Update the package list:
```
sudo apt-get update
```
Install python3.9 or other versions
```
sudo apt install -y python3.6
```



**Deploy Ceph mon01**
Only ceph-mon01 has cephadm to deploy other servers.
```
sudo apt install cephadm
```
Download image ceph before deploy if you want.
```
podman image pull docker.io/ceph/ceph:v15
```
Deploy ceph mon
```
sudo mkdir -p /etc/ceph
sudo cephadm bootstrap --mon-ip 172.16.14.111
```
Output
```
Ceph Dashboard is now available at:

	     URL: https://ceph-mon01:8443/
	    User: admin
	Password: plcs53ae0w

You can access the Ceph CLI with:

	sudo /usr/sbin/cephadm shell --fsid d945961c-8250-11eb-9395-dfaac60d0015 -c /etc/ceph/ceph.conf -k /etc/ceph/ceph.client.admin.keyring

Please consider enabling telemetry to help improve Ceph:

	ceph telemetry on

For more information see:

	https://docs.ceph.com/docs/master/mgr/telemetry/

Bootstrap complete.

```

Create alias for ceph command
```
alias ceph='cephadm shell -- ceph'
echo "alias ceph='cephadm shell -- ceph'" >> ~/.bashrc
```
```
ceph -v
Inferring fsid a6728d92-8239-11eb-904b-ddbec3686bf4
Inferring config /var/lib/ceph/a6728d92-8239-11eb-904b-ddbec3686bf4/mon.ceph-mon01/config
Using recent ceph image docker.io/ceph/ceph:v15
ceph version 15.2.9 (357616cbf726abb779ca75a551e8d02568e15b17) octopus (stable)
```

```
sudo podman ps
CONTAINER ID  IMAGE                                COMMAND               CREATED        STATUS            PORTS   NAMES
CONTAINER ID  IMAGE                                 COMMAND               CREATED             STATUS                 PORTS   NAMES
3ddb80e620af  docker.io/ceph/ceph:v15               -n mon.ceph-mon01...  5 minutes ago       Up 5 minutes ago               ceph-d945961c-8250-11eb-9395-dfaac60d0015-mon.ceph-mon01
7de65c501c4a  docker.io/ceph/ceph:v15               -n mgr.ceph-mon01...  5 minutes ago       Up 5 minutes ago               ceph-d945961c-8250-11eb-9395-dfaac60d0015-mgr.ceph-mon01.brczjo
9742aa894b4a  docker.io/ceph/ceph:v15               -n client.crash.c...  3 minutes ago       Up 3 minutes ago               ceph-d945961c-8250-11eb-9395-dfaac60d0015-crash.ceph-mon01
34d9fec4fc77  docker.io/prom/node-exporter:v0.18.1  --no-collector.ti...  About a minute ago  Up About a minute ago          ceph-d945961c-8250-11eb-9395-dfaac60d0015-node-exporter.ceph-mon01
d50c8bdd99e1  docker.io/prom/prometheus:v2.18.1     --config.file=/et...  About a minute ago  Up About a minute ago          ceph-d945961c-8250-11eb-9395-dfaac60d0015-prometheus.ceph-mon01
018c5b379973  docker.io/prom/alertmanager:v0.20.0   --web.listen-addr...  About a minute ago  Up About a minute ago          ceph-d945961c-8250-11eb-9395-dfaac60d0015-alertmanager.ceph-mon01
dc202381d597  docker.io/ceph/ceph-grafana:6.6.2     /bin/bash             About a minute ago  Up About a minute ago          ceph-d945961c-8250-11eb-9395-dfaac60d0015-grafana.ceph-mon01
```
```
root@ceph-mon01:~# ceph -s
Inferring fsid a6728d92-8239-11eb-904b-ddbec3686bf4
Inferring config /var/lib/ceph/a6728d92-8239-11eb-904b-ddbec3686bf4/mon.ceph-mon01/config
Using recent ceph image docker.io/ceph/ceph:v15
  cluster:
    id:     a6728d92-8239-11eb-904b-ddbec3686bf4
    health: HEALTH_WARN
            OSD count 0 < osd_pool_default_size 3
 
  services:
    mon: 1 daemons, quorum ceph-mon01 (age 8m)
    mgr: ceph-mon01.isxamv(active, since 6m)
    osd: 0 osds: 0 up, 0 in
 
  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:     
```
Copy ssh-key to cluster
```
ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph-mon02
ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph-mon03
ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph-osd01
ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph-osd02
ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph-osd03
```
Tell Ceph that the new nodes is part of the cluster:
```
ceph orch host add ceph-mon02
ceph orch host add ceph-mon03
ceph orch host add ceph-osd01
ceph orch host add ceph-osd02
ceph orch host add ceph-osd03
```
Check Ceph clust nodes
```
ceph orch host ls
```
Deploy other mon nodes
```
ceph orch apply mon --unmanaged
ceph orch daemon add mon ceph-mon02:172.16.14.112
ceph orch daemon add mon ceph-mon03:172.16.14.113
```
Check Ceph status
```
ceph -s
Inferring fsid d945961c-8250-11eb-9395-dfaac60d0015
Inferring config /var/lib/ceph/d945961c-8250-11eb-9395-dfaac60d0015/mon.ceph-mon01/config
Using recent ceph image docker.io/ceph/ceph:v15
  cluster:
    id:     d945961c-8250-11eb-9395-dfaac60d0015
    health: HEALTH_WARN
            OSD count 0 < osd_pool_default_size 3
 
  services:
    mon: 3 daemons, quorum ceph-mon01,ceph-mon02,ceph-mon03 (age 11h)
    mgr: ceph-mon01.brczjo(active, since 16h), standbys: ceph-mon02.yyvrlw
    osd: 0 osds: 0 up, 0 in
 
  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:     
```
Check available disks
```
ceph orch device ls
Inferring fsid d945961c-8250-11eb-9395-dfaac60d0015
Inferring config /var/lib/ceph/d945961c-8250-11eb-9395-dfaac60d0015/mon.ceph-mon01/config
Using recent ceph image docker.io/ceph/ceph:v15
Hostname    Path      Type  Serial  Size   Health   Ident  Fault  Available  
ceph-osd01  /dev/sdc  hdd           5368M  Unknown  N/A    N/A    Yes        
ceph-osd01  /dev/sdd  hdd           5368M  Unknown  N/A    N/A    Yes        
ceph-osd01  /dev/sdb  hdd           5368M  Unknown  N/A    N/A    Yes        
```
Add OSD
```
ceph orch daemon add osd ceph-osd01:/dev/sdb
Inferring fsid d945961c-8250-11eb-9395-dfaac60d0015
Inferring config /var/lib/ceph/d945961c-8250-11eb-9395-dfaac60d0015/mon.ceph-mon01/config
Using recent ceph image docker.io/ceph/ceph:v15
Created osd(s) 0 on host 'ceph-osd01'
```
Or we can add all osd that are available.
```
ceph orch apply osd --all-available-devices
```
Check containers on ceph-osd01
```
podman ps
CONTAINER ID  IMAGE                                 COMMAND               CREATED         STATUS             PORTS   NAMES
534a5b1d39b5  docker.io/prom/node-exporter:v0.18.1  --no-collector.ti...  21 minutes ago  Up 21 minutes ago          ceph-d945961c-8250-11eb-9395-dfaac60d0015-node-exporter.ceph-osd01
102b08d47b29  docker.io/ceph/ceph:v15               -n client.crash.c...  21 minutes ago  Up 21 minutes ago          ceph-d945961c-8250-11eb-9395-dfaac60d0015-crash.ceph-osd01
c2faf1fc406a  docker.io/ceph/ceph:v15               -n osd.0 -f --set...  18 minutes ago  Up 18 minutes ago          ceph-d945961c-8250-11eb-9395-dfaac60d0015-osd.0
```
Check osd tree
```
ceph osd tree
Inferring fsid d945961c-8250-11eb-9395-dfaac60d0015
Inferring config /var/lib/ceph/d945961c-8250-11eb-9395-dfaac60d0015/mon.ceph-mon01/config
Using recent ceph image docker.io/ceph/ceph:v15
ID  CLASS  WEIGHT   TYPE NAME            STATUS  REWEIGHT  PRI-AFF
-1         0.04408  root default                                  
-3         0.01469      host ceph-osd01                           
 0    hdd  0.00490          osd.0            up   1.00000  1.00000
 1    hdd  0.00490          osd.1            up   1.00000  1.00000
 2    hdd  0.00490          osd.2            up   1.00000  1.00000
-5         0.01469      host ceph-osd02                           
 3    hdd  0.00490          osd.3            up   1.00000  1.00000
 4    hdd  0.00490          osd.4            up   1.00000  1.00000
 5    hdd  0.00490          osd.5            up   1.00000  1.00000
-7         0.01469      host ceph-osd03                           
 6    hdd  0.00490          osd.6            up   1.00000  1.00000
 7    hdd  0.00490          osd.7            up   1.00000  1.00000
 8    hdd  0.00490          osd.8            up   1.00000  1.00000
 ```
 remove OSD ID [8]
 ```
 ceph orch osd rm 8
Inferring fsid d945961c-8250-11eb-9395-dfaac60d0015
Inferring config /var/lib/ceph/d945961c-8250-11eb-9395-dfaac60d0015/mon.ceph-mon01/config
Using recent ceph image docker.io/ceph/ceph:v15
Scheduled OSD(s) for removal
```
Check removal status. completes if no entries are shown
```
ceph orch osd rm status
Inferring fsid d945961c-8250-11eb-9395-dfaac60d0015
Inferring config /var/lib/ceph/d945961c-8250-11eb-9395-dfaac60d0015/mon.ceph-mon01/config
Using recent ceph image docker.io/ceph/ceph:v15
No OSD remove/replace operations reported
```

Create Rados Gateway daemons.
```
cephadm install ceph-common
radosgw-admin realm create --rgw-realm=default --default
radosgw-admin zonegroup create --rgw-zonegroup=default --master --default
radosgw-admin zone create --rgw-zonegroup=default --rgw-zone=us-east-1 --master --default
ceph orch apply rgw default us-east-1 --placement="3 ceph-mon01 ceph-mon02 ceph-mon03"
```

  
Create user admin for dashboard.
```
root@ceph-mon01:~# radosgw-admin user create --uid="admin" --display-name="admin" --system
{
    "user_id": "admin",
    "display_name": "admin",
    "email": "",
    "suspended": 0,
    "max_buckets": 1000,
    "subusers": [],
    "keys": [
        {
            "user": "admin",
            "access_key": "50LM0WBLEHDZTE8QXO4S",
            "secret_key": "DBUdECL5eMoYcS2m2dk60ZOBsRHwta3g1o1HDcBC"
        }
    ],
    "swift_keys": [],
    "caps": [],
    "op_mask": "read, write, delete",
    "system": "true",
    "default_placement": "",
    "default_storage_class": "",
    "placement_tags": [],
    "bucket_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "user_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "temp_url_keys": [],
    "type": "rgw",
    "mfa_ids": []
}
```
Set access-key and secret-key for dashboard.
```
ceph dashboard set-rgw-api-access-key 50LM0WBLEHDZTE8QXO4S
ceph dashboard set-rgw-api-secret-key DBUdECL5eMoYcS2m2dk60ZOBsRHwta3g1o1HDcBC
ceph dashboard set-rgw-api-ssl-verify False
ceph dashboard set-rgw-api-user-id admin
```

```
root@ceph-mon01:~# curl http://ceph-mon01
<?xml version="1.0" encoding="UTF-8"?><ListAllMyBucketsResult xmlns="http://s3.amazonaws.com/doc/2006-03-01/"><Owner><ID>anonymous</ID><DisplayName></DisplayName></Owner><Buckets></Buckets></ListAllMyBucketsResult>
```

**Config Keepalived and HAProxy for Kubernetes API**
We need to deploy keepalived and HAProxy on k8s-master01, k8s-master02 and k8s-master03
```
sudo tee /etc/sysctl.d/ceph-rgw.conf<<EOF
net.ipv4.ip_forward = 1
net.ipv4.ip_nonlocal_bind = 1
EOF
sudo sysctl --system
```
```
sudo apt install -y keepalived haproxy
```
Keepalived Config
```
sudo vi /etc/keepalived/keepalived.conf
global_defs {
    router_id LVS_DEVEL
}
vrrp_script check_apiserver {
  script "/etc/keepalived/check_apiserver.sh"
  interval 3
  weight -2
  fall 10
  rise 2
}

vrrp_instance VI_1 {
    state [MASTER/BACKUP]
    interface ens160
    virtual_router_id 51
    priority [101/100] #101 for master and 100 for slave
    authentication {
        auth_type PASS
        auth_pass 1234567890 #your pass
    }
    virtual_ipaddress {
        172.16.14.110
    }
    track_script {
        check_apiserver
    }
}
```
```
sudo vi /etc/keepalived/check_apiserver.sh
#!/bin/sh

errorExit() {
    echo "*** $*" 1>&2
    exit 1
}

curl --silent --max-time 2 --insecure http://localhost:80/ -o /dev/null || errorExit "Error GET http://localhost:80/"
if ip addr | grep -q 172.16.14.100; then
    curl --silent --max-time 2 --insecure http://172.16.14.100:80/ -o /dev/null || errorExit "Error GET http://172.16.14.100:80/"
fi
```
Add the following config in haproxy.cfg
```
sudo vi /etc/haproxy/haproxy.cfg 
...

#---------------------------------------------------------------------
# Configure HAProxy for Kubernetes API Server
#---------------------------------------------------------------------
listen stats
  bind    172.16.14.110:9000
  mode    http
  stats   enable
  stats   hide-version
  stats   uri       /stats
  stats   refresh   30s
  stats   realm     Haproxy\ Statistics
  stats   auth      Admin:Password


############## Configure HAProxy Frontend #############
frontend rgw-http-proxy
    bind 172.16.14.110:8080
    mode tcp
    tcp-request inspect-delay 5s
    default_backend rgw-http

############## Configure HAProxy Backend #############
backend rgw-http
    balance roundrobin
    mode tcp
    option tcp-check
    default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
    server ceph-mon01 172.16.14.111:80 check
    server ceph-mon02 172.16.14.112:80 check
    server ceph-mon03 172.16.14.113:80 check
```                                                               

```
sudo systemctl enable haproxy --now
sudo systemctl enable keepalived --now
```
