### Cinder Storage Service
The OpenStack Block Storage service provides persistent block storage resources that OpenStack Compute instances can consume. This includes secondary attached storage similar to the Amazon Elastic Block Storage (EBS) offering. In addition, you can write images to a Block Storage device for Compute to use as a bootable persistent instance. With the Block Storage service, you can attach a device to only one instance.

The Block Storage service provides:

1. **cinder-api**. Authenticates and routes requests throughout the Block Storage service. It supports the OpenStack APIs only, although there is a translation that can be done through Compute's EC2 interface, which calls in to the Block Storage client.

2. **cinder-scheduler**. Schedules and routes requests to the appropriate volume service. Depending upon your configuration, this may be simple round-robin scheduling to the running volume services, or it can be more sophisticated through the use of the Filter Scheduler. The Filter Scheduler is the default and enables filters on things like Capacity, Availability Zone, Volume Types, and Capabilities as well as custom filters.

3. **cinder-volume**. Manages Block Storage devices, specifically the back-end devices themselves.

4. **cinder-backup**. Provides a means to back up a Block Storage volume to OpenStack Object Storage (swift).

The Block Storage service contains the following components:

* **Back-end Storage Devices**. The Block Storage service requires some form of back-end storage that the service is built on. The default implementation is to use LVM on a local volume group named "cinder-volumes." In addition to the base driver implementation, the Block Storage service also provides the means to add support for other storage devices to be utilized such as external Raid Arrays or other storage appliances. These back-end storage devices may have custom block sizes when using KVM or QEMU as the hypervisor.

* **Users and Tenants**. The Block Storage service can be used by many different cloud computing consumers or customers (tenants on a shared system), using role-based access assignments. Roles control the actions that a user is allowed to perform. In the default configuration, most actions do not require a particular role, but this can be configured by the system administrator in the appropriate policy.json file that maintains the rules. A user's access to particular volumes is limited by tenant, but the user name and password are assigned per user. Key pairs granting access to a volume are enabled per user, but quotas to control resource consumption across available hardware resources are per tenant.

* **Volumes, Snapshots, and Backups**. The basic resources offered by the Block Storage service are volumes and snapshots which are derived from volumes and volume backups:

  * **Volumes**. Allocated block storage resources that can be attached to instances as secondary storage or they can be used as the root store to boot instances. Volumes are persistent R/W block storage devices most commonly attached to the compute node through iSCSI.

  * **Snapshots**. A read-only point in time copy of a volume. The snapshot can be created from a volume that is currently in use or in an available state. The snapshot can then be used to create a new volume through create from snapshot.

  * **Backups**. An archived copy of a volume currently stored in OpenStack Object Storage (swift).

### Implementing Cinder

Install the Cinder components
```
# yum install -y openstack-cinder
```
The configuration file ``/etc/cinder/cinder.conf`` contains all the relevant parameters for the Cinder service.
```
# cp /etc/cinder/cinder.conf /etc/cinder/cinder.conf.orig
```

Source the keystonerc_admin file to enable authentication with administrative privileges
```
# source /root/keystonerc_admin
# openstack-db --init --service cinder --password <password> --rootpw <password>
```

Create the _cinder_ user and the services. Both service versions 1 and 2 need to be created
```
# keystone user-create --name cinder --pass <password>
# keystone user-role-add --user cinder --role admin --tenant services
# keystone service-create --name=cinder --type=volume --description="Cinder Volume Service V1"
# keystone service-create --name=cinderv2 --type=volumev2 --description="Cinder Volume Service V2"
# keystone service-list
+----------------------------------+----------+----------+---------------------------+
|                id                |   name   |   type   |        description        |
+----------------------------------+----------+----------+---------------------------+
| e843e9c5dca4403686debd7e4279add6 |  cinder  |  volume  |  Cinder Volume Service V1 |
| 8df7ea55e7cc4d32aab5a3d5f3bbd628 | cinderv2 | volumev2 |  Cinder Volume Service V2 |
+----------------------------------+----------+----------+---------------------------+

# keystone endpoint-create --service-id e843e9c5dca4403686debd7e4279add6 --publicurl 'http://caldara01:8776/v1/%(tenant_id)s' --adminurl 'http://caldara01:8776/v1/%(tenant_id)s' --internalurl 'http://caldara01:8776/v1/%(tenant_id)s'
# keystone endpoint-create --service-id 8df7ea55e7cc4d32aab5a3d5f3bbd628 --publicurl 'http://caldara01:8776/v2/%(tenant_id)s' --adminurl 'http://caldara01:8776/v2/%(tenant_id)s' --internalurl 'http://caldara01:8776/v2/%(tenant_id)s'

```
Edit the configuration file
```
vi /etc/cinder/cinder.conf

[DEFAULT]
debug=True
verbose=True
log_dir=/var/log/cinder
use_syslog=False
auth_strategy = keystone
enabled_backends=lvm

rabbit_userid = guest
rabbit_password = guest
rabbit_host = caldara01
rabbit_use_ssl = false
rabbit_port = 5672

[database]
sql_connection = mysql://cinder:<password>@localhost/cinder
idle_timeout=3600
min_pool_size=1
max_retries=10
retry_interval=10

[keystone_authtoken]
admin_tenant_name = services
admin_user = cinder
admin_password = <password>
auth_host = caldara01
auth_port = 35357
auth_protocol = http

[lvm]
iscsi_helper=lioadm
iscsi_ip_address=10.10.10.97
volume_driver=cinder.volume.drivers.lvm.LVMISCSIDriver
volume_backend_name=LVM
volume_group=storage
```

Cinder uses by default a LVM storage backend. Before to start the service, make sure that a Volume Group is defined on the node hosting the service. By default, Cinder will look for a Volume Group called *cinder-volumes*. Use the directive ``volume_group`` in the configuration file to force for a different Volume Group name. For example: 
```
#vgs
  VG      #PV #LV #SN Attr   VSize   VFree
  os        2   3   0 wz--n- 232.39g      0
  storage   1   1   0 wz--n- 232.88g 227.88g

```
In the above example, use: ``volume_group=storage``

Start and enable the services. Check for any errors.
```
# systemctl enable openstack-cinder-api
# systemctl enable openstack-cinder-scheduler
# systemctl enable openstack-cinder-volume
# openstack-service start cinder
# openstack-status

== Cinder services ==
openstack-cinder-api:                   active
openstack-cinder-scheduler:             active
openstack-cinder-volume:                active
openstack-cinder-backup:                inactive  (disabled on boot)
```

The ``cinder`` CLI command is used to manage the service
```
# cinder type-create lvm
+--------------------------------------+------+
|                  ID                  | Name |
+--------------------------------------+------+
| 9c1a4ea8-3e46-4706-9030-057fe36ae853 | lvm  |
+--------------------------------------+------+
#  cinder type-key lvm set volume_backend_name=LVM
# cinder create --display-name vol1 --volume-type lvm 10
+---------------------+--------------------------------------+
|       Property      |                Value                 |
+---------------------+--------------------------------------+
|     attachments     |                  []                  |
|  availability_zone  |                 nova                 |
|       bootable      |                false                 |
|      created_at     |      2015-05-18T15:04:41.922747      |
| display_description |                 None                 |
|     display_name    |                 vol1                 |
|      encrypted      |                False                 |
|          id         | 0ad8da3b-1744-4c5c-ba40-53908367b0dd |
|       metadata      |                  {}                  |
|         size        |                  10                  |
|     snapshot_id     |                 None                 |
|     source_volid    |                 None                 |
|        status       |               creating               |
|     volume_type     |                 lvm                  |
+---------------------+--------------------------------------+
# cinder list
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+
|                  ID                  |   Status  | Display Name | Size | Volume Type | Bootable | Attached to |
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+
| 0ad8da3b-1744-4c5c-ba40-53908367b0dd | available |     vol1     |  10  |     lvm     |  false   |             |
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+
```

The volume is a LVM volume available to be used
```
# lvscan
  ACTIVE            '/dev/storage/volume-fc96dc1c-6990-4ace-9523-8c57772c2e01' [10.00 GiB] inherit
  ACTIVE            '/dev/os/root' [50.00 GiB] inherit
  ACTIVE            '/dev/os/swap' [3.89 GiB] inherit
  ACTIVE            '/dev/os/data' [178.50 GiB] inherit

# mkfs.ext3 /dev/storage/volume-fc96dc1c-6990-4ace-9523-8c57772c2e01
# mount /dev/storage/volume-fc96dc1c-6990-4ace-9523-8c57772c2e01 /mnt
# df -Th
Filesystem          Type      Size  Used Avail Use% Mounted on
/dev/mapper/os-root xfs        50G  2.7G   48G   6% /
devtmpfs            devtmpfs  3.8G     0  3.8G   0% /dev
tmpfs               tmpfs     3.8G     0  3.8G   0% /dev/shm
tmpfs               tmpfs     3.8G   17M  3.8G   1% /run
tmpfs               tmpfs     3.8G     0  3.8G   0% /sys/fs/cgroup
/dev/mapper/os-data xfs       179G   26G  154G  15% /data
/dev/sda1           xfs       497M  271M  227M  55% /boot
/dev/dm-2           ext3      9.8G   23M  9.2G   1% /mnt
#
# umount /mnt
```

Deleting a volume may take some time, depending on the size, because Cinder fill with zero the volume when delete it.
```
# cinder delete vol1
# cinder list
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+
|                  ID                  |   Status  | Display Name | Size | Volume Type | Bootable | Attached to |
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+
```
### Implementing multiple storage backends
Configuring multiple-storage backends, allows to create several backend storage solutions that serve the same OpenStack configuration and one cinder-volume service is launched for each backend storage. In a multiple-storage backend configuration, eachback end has a name. Several back ends can have the same name. In that case, the scheduler properly decides which backend the volume has to be created in.

The name of the backend is declared as an extra-specification of a volume type. When a volume is created, the scheduler chooses an appropriate backend to handle the request, according to the volume type specified by the user.
To enable a multiple-storage backends, the ``enabled_backends`` option in the cinder configuration file need to be used. This option defines the names of the configuration groups for the different backends. Each name is associated to one configuration group for a backend. The configuration group name is not related to the name of the backend.

As example, we define two LVM backends named **LOLLO** and **LOLLA** respctively. In the cinder configuration file, we are going to declare two configuration groups, one for each backend, called **lvm1** and **lvm2** respectively. We set the multiple backend option as  ``enabled_backends=lvm1,lvm2`` in the configuration file. It will look like:

```
[DEFAULT]
debug=True
verbose=True
log_dir=/var/log/cinder
use_syslog=False
auth_strategy = keystone
enabled_backends=lvm1,lvm2

rabbit_userid = guest
rabbit_password = guest
rabbit_host = caldara01
rabbit_use_ssl = false
rabbit_port = 5672

[database]
sql_connection = mysql://cinder:caldara123@localhost/cinder
idle_timeout=3600
min_pool_size=1
max_retries=10
retry_interval=10

[keystone_authtoken]
admin_tenant_name = services
admin_user = cinder
admin_password = caldara123
auth_host = caldara01
auth_port = 35357
auth_protocol = http

[lvm1]
iscsi_helper=lioadm
iscsi_ip_address=10.10.10.97
volume_driver=cinder.volume.drivers.lvm.LVMISCSIDriver
volume_backend_name=LOLLA
volume_group=storage

[lvm2]
iscsi_helper=lioadm
iscsi_ip_address=10.10.10.97
volume_driver=cinder.volume.drivers.lvm.LVMISCSIDriver
volume_backend_name=LOLLO
volume_group=storage

```
To enable the new configuration, restart the cinder service
```
# openstack-service restart cinder
```
Create two new volume types and associate them with the two backends
```
# cinder type-create lvm_gold
+--------------------------------------+----------+
|                  ID                  |   Name   |
+--------------------------------------+----------+
| 60ee47a6-ebaa-4d7d-8586-b722fb00677f | lvm_gold |
+--------------------------------------+----------+
# cinder type-create lvm_silver
+--------------------------------------+------------+
|                  ID                  |    Name    |
+--------------------------------------+------------+
| e67d309c-a3e7-42a0-b8ba-f34485582734 | lvm_silver |
+--------------------------------------+------------+
# cinder type-list
+--------------------------------------+------------+
|                  ID                  |    Name    |
+--------------------------------------+------------+
| 60ee47a6-ebaa-4d7d-8586-b722fb00677f |  lvm_gold  |
| e67d309c-a3e7-42a0-b8ba-f34485582734 | lvm_silver |
+--------------------------------------+------------+
# cinder type-key lvm_silver set volume_backend_name=LOLLO
# cinder type-key lvm_gold set volume_backend_name=LOLLA

```
Now the two new backends can be used to create volumes
```
# cinder create --display-name vol1 --volume-type lvm_gold 5
# cinder create --display-name vol2 --volume-type lvm_silver 5
# cinder list
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+
|                  ID                  |   Status  | Display Name | Size | Volume Type | Bootable | Attached to |
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+
| 0b1acd71-651f-41eb-a4e8-7f1fe8f39825 | available |     vol1     |  5   |   lvm_gold  |  false   |             |
| 9ae1f05e-ccf0-4532-96be-c1c21a293130 | available |     vol2     |  5   |  lvm_silver |  false   |             |
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+
```

Note that each backend has its own cinder volume service
```
# cinder-manage service list
Binary           Host                                 Zone             Status     State Updated At
cinder-scheduler caldara01                            nova             enabled    :-)   2015-05-18 18:05:26
cinder-volume    caldara01@lvm2                       nova             enabled    :-)   2015-05-18 18:05:27
cinder-volume    caldara01@lvm1                       nova             enabled    :-)   2015-05-18 18:05:27

```

### Haking Cinder service
The Cinder service uses a MySQL database called **cinder**.
```
# mysql -u cinder -p
Enter password:
Welcome to the MariaDB monitor.  Commands end with ; or \g.
MariaDB [(none)]> use cinder;

Database changed

MariaDB [cinder]> show tables;
+--------------------------+
| Tables_in_cinder         |
+--------------------------+
| backups                  |
| cgsnapshots              |
| consistencygroups        |
| encryption               |
| iscsi_targets            |
| migrate_version          |
| quality_of_service_specs |
| quota_classes            |
| quota_usages             |
| quotas                   |
| reservations             |
| services                 |
| snapshot_metadata        |
| snapshots                |
| transfers                |
| volume_admin_metadata    |
| volume_glance_metadata   |
| volume_metadata          |
| volume_type_extra_specs  |
| volume_types             |
| volumes                  |
+--------------------------+
21 rows in set (0.00 sec)

MariaDB [cinder]> select host from volumes;
+----------------------+
| host                 |
+----------------------+
| caldara01@lvm1#LOLLA |
| caldara01@lvm2#LOLLO |
+----------------------+
2 rows in set (0.00 sec)

MariaDB [cinder]> exit
Bye

```


