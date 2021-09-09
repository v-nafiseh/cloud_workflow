### before all:
```sh
sudo yum update -y
```
#### if planning to set external network>>>>change the system Ip to static as following:
- recognise the ethernet interface with:
 ```sh
 ip a
 ```
- edit the config file:
```sh
vim /etc/sysconfig/network-scripts/ifcfg-eth0
```
- follow the instructions for setting a static IP in the link below:
- https://www.cyberciti.biz/faq/howto-setting-rhel7-centos-7-static-ip-configuration/
```sh
sudo systemctl disable firewalld
sudo systemctl stop firewalld
sudo systemctl disable NetworkManager
sudo systemctl stop NetworkManager
sudo systemctl enable network
sudo systemctl start network
```
	
- install openstack train release or other considered release:
```sh
sudo yum install -y centos-release-openstack-train
```
- install packstack installer:
```sh
sudo yum install -y openstack-packstack
```
	
- before running packstack --allinone do the following to prevent neutron problems with ovn:
- Since stein release packstack runs neutron with ovn backend by default, if you want to deploy with ovs as neutron backend, instead do:
```sh
packstack --allinone --os-neutron-l2-agent=openvswitch --os-neutron-ml2-mechanism-drivers=openvswitch --os-neutron-ml2-tenant-network-types=vxlan --os-neutron-ml2-type-drivers=vxlan,flat --provision-demo=n --os-neutron-ovs-bridge-mappings=extnet:br-ex --os-neutron-ovs-bridge-interfaces=br-ex:eth0
```

- follow description in link below:
- https://www.rdoproject.org/networking/neutron-with-existing-external-network/
	
-As the [rdoproject](https://www.rdoproject.org/networking/neutron-with-existing-external-network/) implies Once the process is complete, you can log in to the OpenStack web interface Horizon by going to http://$YOURIP/dashboard. The user name is admin.
The password can be found in the file keystonerc_admin in the /root directory of the control node.

### brief explanation on core services

#### Services and API

#### Backend services:

##### Rabbitmq:

- To pass messages between cloud components. In case of failure, the management is gonna be impossible.
- Contact information must be included in service configuration:
```sh
Vim nova.conf 
```
##### Mariadb:

- Keystone doesn’t itself save information, it uses database

##### Storage:

- Openstack storage is ephemeral which does not keep everything permanently.
- Disk files are written to /var/lib/nova/instances on the compute node Can’t move instances to other compute nodes cause the configuration doesn’t move and this action needs extra configuration on storage.
To make it available all around, we can use NFS or other shared storage as a backend.
- Use cinder volumes for really persistent cloud storage

##### Keystone:

- Used for authentication and authorization
- Stores information at in the database
- Verifying identity services:
```sh
Cat /etc/keystone/keystone.conf
Cat /root/keystonerc_admin
Keystone service-list
Openstack-service status keystone
Mysql -e ‘use keystone;show tables;
```

##### Keystone backend services:
- SQL backend
- LDAP backend

###### Keystone in consisted of different services
- Identity
- Resource
- Assignment
- Token
- Catalog
- policy

###### Troubleshooting keystone services:
- Firstly check /var/log/keystone.log this way:
```sh
grep -v INFO keystone.log
```
- Secondly, if authentication doesn’t work, (invalid openstack credentials)
- Export:
```sh
export SERVICE_TOKEN=$(crudini --get /etc/keystone/keystone.conf DEFAULT admin_token)
export SERVICE_ENDPOINT=http://servern.example.com:portnumber/v
```

##### There are 2 different object storages available in openstack:
- Swift: is the original openstack object storage invented by rackspace
- Ceph:officially not a part of openstack, but could be useful	
- The major difference between these two is the access type.
- Swift has API access. However, beside having API access, it also has block device, cephfs

##### Understanding swift object storage
- Disk based storage in a cloud is not elastic and difficult to scale
- Even distributed file systems are not elastic enough
- In obj storage data is written in multiple binary nodes not files and replicated over many storage nodes
- The metadata is spreaded over multiple nodes also
- The application makes an api call to access the obj storage
- Rest api > swift proxy > many storage nodes(multiple binary objects on each node)
- Manual procedure for deploying swift object storage:
###### Install rpm packages:
```sh
Yum install -y openstack-swift-proxy openstack-swift-object openstack-swift-container openstack-swift-account python-swiftclient memcached
Source root/keystonerc_admin
Keystone user-create --name swift -pass password --tenant services

Keystone role-create --name SwiftOperator

Keystone user-role-add --role admin --tenant services --user swift
```

##### Configuration for object storage
###### Swift rings:
- They determine where data is stored in a cluster
###### There are 3 rings:
- Object: binary objects
- Container: storage of binary objects
- Account services: permissions
- Storage devices are divided in partitions and Each partition is a directory on xfs filesystem
###### Swift ring config
- Swift proxy
- Managing objects in swift store

##### Glance:

- In Iaas cloud, instances are not installed, they are deployed, they are built from images.

![](https://github.com/v-nafiseh/cloud_workflow/blob/master/glanceArch.png)


- As the picture above represents, glance is consisted of different components:
- Glance-api: which accepts api requests for images from either client and nova components.
- Glance registry: includes information of each image metadata.(store, process, retrieve).
- Database: stores image metadata

- Glance stores its files in filesystem by default, moreover it can store them in object storage or block storage.

- Glance store, is the communication layer between glance and external storage backends or local file system > /var/lib/glance/images/
- While uploading images to glance, we should specify the disk formats, which some of them are listed below:

###### Different disk formats:
- raw(without any metadata)
- Vhd
- Vmdk(vmware)
-vdi(virtualbox and QEMU)
- Iso
- qcow2(QEMU default)
- aki(Amazon kernel image)
- Ari
- ami

###### Container formats(container actually contains the image file with metadata):
- Bare (no metadata)
- ovf(open virtualization format)
- Ari
- Ami
- Ova
- Docker

###### Download the source image:
```sh
wget http://download.cirros-cloud.net/0.3.5/cirros-0.3.5-x86_64-disk.img
```

- Creating an image in openstack cli:
```sh
Glance image-create --name “cirros” --file /path/to/image --disk-format qcow2 --container bare
```
- Confirm upload
```sh
Glance image list
```
##### Verifying Glance image services:
- To check what packstack has configured:
```sh
Grep Glance /root/answers.txt
```




