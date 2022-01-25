# **Ceph 3 Install on RHEL 7**

Co-located demo only!  Not supported outside of containers.  Container based install below.

[https://access.redhat.com/documentation/en-us/red_hat_ceph_storage/3/html-single/installation_guide_for_red_hat_enterprise_linux/index](https://access.redhat.com/documentation/en-us/red_hat_ceph_storage/3/html-single/installation_guide_for_red_hat_enterprise_linux/index)

Playbooks located at - github.com/johnsimcall/storage-install



1. Create admin server either as a VM or bare metal and install RHEL 7.x on it 
```
        subscription-manager register --username=<your user name> --password=<your password>
        subscription-manager attach --pool=<your pool id here>
        subscription-manager repos --disable="*" --enable=rhel-7-server-rpms --enable=rhel-7-server-rhceph-3-tools-rpms --enable=rhel-7-server-extras-rpms --enable=rhel-7-server-optional-rpms
        yum install -y ceph-ansible   # ceph-ansible pulls in ansible as a dependency
        ssh-keygen
```
2. Create ansible hosts file on admin server with hosts file as below.  This requires 9 servers (physical or virtual). 
```
**Ansible hosts file**
    $ cat /etc/ansible/hosts
    [mgrs]
    ceph[1:3]

    [mons]
    ceph[1:3]

    [osds]
    ceph[1:3]
    
    [rgws]
    ceph4
    
    [mdss]
    ceph5
    
    [cephgui]
    ceph-gui
    
    [clients]
    client
    
    [iscsigws]
    iscsigw
    
    [admin]
    admin
```
# PRE-REQS - for either rpm or container based installs

**From the admin server do the following:**
```
    # ansible-playbook ~/storage-install/make-vms-prompt.yml
    ANSIBLE_HOST_KEY_CHECKING=false ansible all -m authorized_key -a "user=root state=present key='<your public key here>'" -k
    ansible all -m ping
    ansible all -a "subscription-manager register --username=<your user name> --password=<your password>"
    ansible all -a "subscription-manager attach --pool=&lt;your pool id here>"
    ansible all -m shell -a "subscription-manager repos --disable='*' --enable=rhel-7-server-rpms --enable=rhel-7-server-extras-rpms"
    ansible mons -m shell -a "subscription-manager repos --enable=rhel-7-server-rhceph-3-mon-rpms"
    ansible osds -m shell -a "subscription-manager repos --enable=rhel-7-server-rhceph-3-osd-rpms"
    ansible rgws -m shell -a "subscription-manager repos --enable=rhel-7-server-rhceph-3-tools-rpms"
    ansible mdss -m shell -a "subscription-manager repos --enable=rhel-7-server-rhceph-3-tools-rpms"
    ansible clients -m shell -a "subscription-manager repos --enable=rhel-7-server-rhceph-3-tools-rpms" 
    ansible all -m yum -a "name=firewalld state=present"
    ansible all -m yum -a "name=ansible state=present"
    ansible all -m yum -a "name=ceph-ansible state=present"
    ansible clients -m yum -a "name=ceph-common state=present"
    ansible all -m systemd -a "name=firewalld state=started enabled=true"
    ansible mons -m firewalld -a "port=6789/tcp permanent=yes state=enabled"
    ansible osds,mgrs -m firewalld -a "port=6800-7300/tcp permanent=yes state=enabled"
    ansible mdss -m firewalld -a "port=6800/tcp permanent=yes state=enabled"
    ansible rgws -m firewalld -a "port=7480/tcp permanent=yes state=enabled"
    ansible rgws -m firewalld -a "port=80/tcp permanent=yes state=enabled"
    ansible all -m yum -a “name=ntp state=present”

    #check clocks and make sure they are the same.  Set up ntp / chrony if necessary.

    ansible all -m yum -a "name=* state=latest"

  **# IF CONTAINER BASED INSTALL**
    ansible all -m yum -a "name=docker state=latest"
    ansible all -m systemd -a "name=docker state=started enabled=true"
  **# END IF**

    ansible all -m reboot
```

**On the Admin (Ceph ansible) server (if this is the first run- skip this for subsequent runs):**
```
    mkdir ~/ceph-ansible-keys
    ln -s /usr/share/ceph-ansible/group_vars /etc/ansible/group_vars
    cd /usr/share/ceph-ansible
    cp site.yml.sample site.yml					                        # required - main installer
    cp group_vars/all.yml.sample group_vars/all.yml		          # required - mons settings
    cp group_vars/mgrs.yml.sample group_vars/mgrs.yml		        # required - ceph managers
    cp group_vars/osds.yml.sample group_vars/osds.yml		        # required - osds settings
    cp group_vars/mdss.yml.sample group_vars/mdss.yml		        # optional - if using cephfs
    cp group_vars/clients.yml.sample group_vars/clients.yml	    # optional - if using clients
    cp group_vars/rgws.yml.sample group_vars/rgws.yml		        # optional - if using object
    cp group_vars/nfss.yml.sample group_vars/nfss.yml		        # optional - if using NFS
    cp group_vars/iscsigws.yml.sample group_vars/iscsi-gws.yml	# optional - if using iscsi
```
**For RPM based install**
```
    mons variables (in all.yml - from grep "^[^#;]" group_vars/all.yml)
```
```
    ---
    dummy:
    fetch_directory: ~/ceph-ansible-keys
    ntp_service_enabled: false
    ceph_repository_type: cdn
    ceph_origin: repository
    ceph_repository: rhcs
    ceph_rhcs_version: 3
    monitor_interface: eth0
    public_network: 192.168.1.0/24
    radosgw_dns_name: ceph1.cluster.net 
    radosgw_civetweb_port: 80
    radosgw_interface: eth0
    ceph_conf_overrides:
      global:
    	osd_pool_default_pg_num: 128
    	osd_pool_default_pgp_num: 128
```

**For Container based install (that supports co-location of services like mons and osds)**

From: [https://access.redhat.com/documentation/en-us/red_hat_ceph_storage/3/html-single/container_guide/](https://access.redhat.com/documentation/en-us/red_hat_ceph_storage/3/html-single/container_guide/)

**Yml files to edit:**
```
    cp site-docker.yml.sample site-docker.yml
    cat /usr/share/ceph-ansible/group-vars/all.yml
    ---
    dummy:
    fetch_directory: ~/ceph-ansible-keys
    ntp_service_enabled: false
    containerized_deployment: true
    ceph_docker_image: "rhceph-3-rhel7"
    ceph_docker_image_tag: "latest"
    ceph_docker_registry: "registry.access.redhat.com/rhceph"
    monitor_interface: eth0
    public_network: 192.168.2.0/24
    radosgw_dns_name: ceph3.cluster.net 
    radosgw_civetweb_port: 80
    radosgw_interface: eth0
    ceph_conf_overrides:
      global:
            osd_pool_default_pg_num: 128
            osd_pool_default_pgp_num: 128
```

**All remaining yml files below are the same regardless of whether doing a container or rpm based install**

osds variables (in osds.yml - from grep "^[^#;]" group_vars/osds.yml))
```
    ---
    dummy:
    osd_auto_discovery: true
    osd_scenario: collocated
```
rgw servers (in rgws.yml - - from grep "^[^#;]" group_vars/rgws.yml)
```
    ---
    dummy:
    copy_admin_key: true
    ceph_rgw_civetweb_port: 80
```

Client servers (in clients.yml - from grep "^[^#;]" group_vars/clients.yml)

```
    ---
    dummy:
    copy_admin_key: true
```

iSCSI servers (in iscsi-gws.yml)
```
    ---
    dummy:
    gateway_ip_list: 192.168.2.223
    rbd_devices:
      - { pool: 'rbd', image: 'rbd1', size: '30G', host: 'ceph3', state: 'present' }
      - { pool: 'rbd', image: 'rbd2', size: '20G', host: 'ceph3', state: 'present' }
    client_connections:
      - { client: 'iqn.1994-05.com.redhat:rh7-iscsi-client', image_list: 'rbd.rbd1', chap: 'rh7-iscsi-client/redhat', status: 'present' }
      - { client: 'iqn.1991-05.com.microsoft:w2k12r2', image_list: 'rbd.rbd2', chap: 'w2k12r2/microsoft_w2k12', status: 'present' }
```

**Run the ansible playbooks:**
```
    cd into /usr/share/ceph-ansible: cd /usr/share/ceph-ansible
    ansible-playbook site.yml				# if doing rpm based install
```
**Or** 
```
    ansible-playbook site-docker.yml			# if doing container based install
```
**Or install each server type individually**
```
    ansible-playbook site.yml --limit mons	 	# install mons
    ansible-playbook site.yml --limit osds	 	# install osds - skip if co-located
    ansible-playbook site.yml --limit mgrs	 	# install managers - skip if co-located
    ansible-playbook site.yml --limit clients		# install client servers
    ansible-playbook site.yml --limit rgws		# install Object (S3/SWIFT) gateways
    ansible-playbook site.yml --limit mdss		# install CephFS gateways
    ansible-playbook site.yml --limit iscsigws		# install iSCSI gateways
```

**Post install**

**Get RBD running and map a client to an RBD**

**On the client(s):**

**Create rbd-pool and rbd:**
```
  ceph osd pool create rbd-pool 30 30 replicated
  ceph auth get-or-create client.rbd mon 'allow r' osd 'allow rwx pool=rbd-pool' -o /etc/ceph/rbd.keyring
  rbd create rbd1 --size 10240 --cluster ceph -p rbd-pool
```
**Disable default rbd features:**
```
  rbd feature disable rbd1 deep-flatten --cluster ceph -p rbd-pool
  rbd feature disable rbd1 fast-diff --cluster ceph -p rbd-pool
  rbd feature disable rbd1 object-map --cluster ceph -p rbd-pool
  rbd feature disable rbd1 exclusive-lock --cluster ceph -p rbd-pool
```
**Map the RBD:**
```
  rbd map rbd1 --cluster ceph --pool rbd-pool
  mkfs.xfs /dev/rbd0
  mkdir /mnt/rbd0
  mount /dev/rbd0 /mnt/rbd0
  df -h
```

**Get RGW running from the client**
    **On ceph1**
```
    mkdir /var/lib/ceph/radosgw/ceph-rgw.`hostname -s`
    touch /var/lib/ceph/radosgw/ceph-rgw.`hostname -s`/done
    radosgw-admin user create --uid mike --display-name="Mike" --email="mike@email.com" --access-key=craig --secret=craigsecret
```

**Test with the following.  You should not get a connection error.**

```
    curl http://ceph3.cluster.net
```

**Install s3cmd on the client**
```
    rpm -ivh https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
    yum update -y
    yum install s3cmd -y
    s3cmd --configure
        Access Key: craig
        Secret Key: craigsecret
        Default Region [US]:
        S3 Endpoint [s3.amazonaws.com]:ceph3.cluster.net
        DNS-style bucket+hostname:port template for accessing a bucket [%(bucket)s.s3.amazonaws.com]: %(bucket).ceph3.cluster.net
        Encryption password:
        Path to GPG program [/usr/bin/gpg]:
        Use HTTPS protocol [Yes]: No
        HTTP Proxy server name:
        Test access with supplied credentials? [Y/n] Y
        Success. Your access key and secret key worked fine :-)
        Save settings? [y/N] y
```

**Or use this minimal .s3cfg file**

```
    [default]
    access_key = craig
    host_base = sds.ddns.me:80
    host_bucket = %(bucket).s3.sds.ddns.me:80
    secret_key = craigsecret
    use_https = False
    signature_v2 = true
```
  
  **Using S3**
    **NOTE - from the Ceph docs :**
        Before creating pools, refer to the[ Pool, PG and CRUSH Config Reference](http://docs.ceph.com/docs/jewel/rados/configuration/pool-pg-config-ref). Ideally, you should override the default value for the number of placement groups in your Ceph configuration file, as the default is NOT ideal. For details on placement group numbers refer to[ setting the number of placement groups](http://docs.ceph.com/docs/jewel/rados/operations/placement-groups#set-the-number-of-placement-groups)

```
    s3cmd mb s3://bucket1
    Bucket 's3://bucket1/' created
    s3cmd ls
    2018-04-16 15:07  s3://bucket1
```

**Getting CephFS running**
**From client**

```
    ceph osd pool create cephfs-data 32
    ceph osd pool create cephfs-metadata 32
    ceph fs new cephfs cephfs-metadata cephfs-data
    ceph osd pool application enable cephfs-data cephfs
    ceph osd pool application enable cephfs-metadata cephfs
    ceph fs status cephfs
    ceph auth get-or-create client.cephfs mon 'allow r' mds 'allow rw' osd 'allow rwx pool=cephfs-metadata,allow rwx pool=cephfs-data'
    ceph auth get client.cephfs | grep -i key | cut -d ' ' -f 3 > /root/ceph.client.cephfs.secret
    mkdir /mnt/cephfs
    mount -t ceph ceph5:6789:/ /mnt/cephfs/ -o name=cephfs,secretfile=/root/ceph.client.cephfs.secret
    df -h
```

In the event something goes wrong, you can remove the mds this way:

```
    ceph tell mon.\* injectargs '--mon-allow-pool-delete=true'
    ceph mds cluster_down \
    ceph mds fail 0 \
    ceph fs rm &lt;cephfs name> --yes-i-really-mean-it \
    ceph osd pool delete cephfs_data cephfs_data --yes-i-really-really-mean-it \
    ceph osd pool delete cephfs_metadata pool cephfs_metadata pool --yes-i-really-really-mean-it \
```

**Getting iSCSI running**

**Just say no**



**Ceph demo**

Install? (takes a long time)

Create block storage

Create object storage

Create file storage

Create iSCSI storage

Connect to block storage

Connect to object storage

Connect to file storage

Connect to iSCSI storage

Put data in block storage

Put data in object storage

Put data in file storage

Put data in iSCSI storage

Take a snapshot of block storage

N/A - Take a snapshot of object storage 

N/A - Take a snapshot of file storage

Take a snapshot of iSCSI storage

Restore from a snapshot

Create a clone from a snapshot

Destroy storage

TODO - Expand Storage (node and/or drive/osd)

TODO - Fail a drive

TODO - Replace failed drive

TODO - Fail a node

TODO - Replace failed node

TODO - Add dedupe / compression

TODO - Bluestore

TODO - Multisite



**REMOVE AN OSD DRIVE **
```
ssh root@ceph5 “systemctl disable ceph-osd@1”
ssh root@ceph5 “systemctl stop ceph-osd@1”
ssh root@ceph1 “ceph --cluster ceph osd crush remove osd.1”
ssh root@ceph1 “ceph --cluster ceph auth del osd.1”
ssh root@ceph1 “ceph --cluster ceph osd rm 1”
```
**REMOVE AN OSD NODE **
```
ssh root@ceph1 “ceph --cluster ceph osd set noscrub”
ssh root@ceph1 “ceph --cluster ceph osd set nodeep-scrub”
ssh root@ceph1 ”Check Cluster Capacity”
ssh root@ceph1 “ceph --cluster ceph df”
ssh root@ceph1 “rados --cluster ceph df”
ssh root@ceph1 “ceph --cluster ceph osd df”
ssh root@ceph1 “ceph --cluster ceph osd crush rm osd5”
ssh root@ceph1 “ceph --cluster ceph osd unset noscrub”
ssh root@ceph1 “ceph --cluster ceph osd unset nodeep-scrub”
ssh root@ceph5 “poweroff”
```
**IN CASE OF EMERGENCY**
```
ansible-playbook infrastructure-playbooks/purge-docker-cluster.yml
```
