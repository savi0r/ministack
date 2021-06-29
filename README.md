
# Abstract:

This project is a small scale cloud management solution which basically has three main parts:
1. Virtualization
2. Distributed storage
3. Monitoring

Through these building blocks you can create VMs seamlessly and manage them via a web API and by utilising distributed storage you are able to live migrate your VMs and provide a highly available service

![network diagram](img/Network-Diagram.jpg)

## Storage:

We will create a glusterfs cluster as our storage backend and serve files through Ganesha which basically is an abstraction layer to plug in different storage backends. Then we will create a high available cluster out of it using pacemaker and corosync by providing a floating ip in order to reach one of our gluster servers and in case if one of them failed pacemaker and corosync will replace will replace it immediately so users won't notice any outage in our service. This guide is based on the resource provided by Oracle.
https://oracle.github.io/linux-labs/HA-NFS/

Our components in more detail are :
    Corosync provides clustering infrastructure to manage which nodes are involved, their communication and quorum
    Pacemaker manages cluster resources and rules of their behavior
    Gluster is a scalable and distributed filesystem
    Ganesha is an NFS server which can use many different backing filesystem types including Gluster


Set DNS to Shecan

```
vi /etc/resolve.conf
nameserver 178.22.122.100
```

Install necessary packages

```
sudo -s
dnf update

#https://www.server-world.info/en/note?os=CentOS_8&p=glusterfs7&f=4

dnf -y install centos-release-nfs-ganesha30 
sed -i -e "s/enabled=1/enabled=0/g" /etc/yum.repos.d/CentOS-NFS-Ganesha-3.repo 
 dnf --enablerepo=centos-nfs-ganesha3 -y install nfs-ganesha-gluster 
 mv /etc/ganesha/ganesha.conf /etc/ganesha/ganesha.conf.org 
 
 #https://www.server-world.info/en/note?os=CentOS_8&p=pacemaker&f=1
 
dnf --enablerepo=HighAvailability -y install corosync pacemaker pcs 

#https://kifarunix.com/install-and-setup-glusterfs-storage-cluster-on-centos-8/

dnf -y install centos-release-gluster
dnf config-manager --set-enabled PowerTools
dnf -y install glusterfs-server
```

Create the Gluster volume
I attached a 20GB spare disk to my vms on /dev/vdb so change it with your own spare volume which can be seen through utilising `lsblk` command
(On all masters) Create an XFS filesystem on /dev/vdb with a label of gluster-000

```
mkfs.xfs -f -i size=512 -L gluster-000 /dev/vdb

```

(On all masters) Create a mountpoint, add an fstab(5) entry for a disk with the label gluster-000 and mount the filesystem 

```
mkdir -p /data/glusterfs/sharedvol/mybrick
echo 'LABEL=gluster-000 /data/glusterfs/sharedvol/mybrick xfs defaults  0 0' >> /etc/fstab
mount /data/glusterfs/sharedvol/mybrick
```

(On all masters) Enable and start the Gluster service 

```
systemctl enable --now glusterd
```

On master1: Create the Gluster environment by adding peers - I changed /etc/hosts by hand and entered ip addrreses of my nodes to that so that master2 and 3 can be resolvable- 

```
gluster peer probe master2
gluster peer probe master3
#you should see peer probe: success. as the output
gluster peer status
#you should see Number of Peers: 2 as the output
```

Show that our peers have joined the environment

On master2&3:
```
gluster peer status
#you should see Number of Peers: 2 as the output
```

On master1: Create a Gluster volume named sharedvol which is replicated across our three hosts: master1, master2 and master3. 

```
gluster volume create sharedvol replica 3 master{1,2,3}:/data/glusterfs/sharedvol/mybrick/brick
```

On master1: Enable our Gluster volume named sharedvol

```
gluster volume start sharedvol
```

Our replicated Gluster volume is now available and can be verified from any master so make sure you see the same output by running this command on every master node

```
gluster volume info
```

Configure Ganesha

Ganesha is the NFS server that shares out the Gluster volume. In this example we allow any NFS client to connect to our NFS share with read/write permissions. 

(On all masters) Populate the file /etc/ganesha/ganesha.conf with our configuration 

```
vi /etc/ganesha/ganesha.conf
EXPORT{
    Export_Id = 1 ;       # Unique identifier for each EXPORT (share)
    Path = "/sharedvol";  # Export path of our NFS share

    FSAL {
        name = GLUSTER;          # Backing type is Gluster
        hostname = "localhost";  # Hostname of Gluster server
        volume = "sharedvol";    # The name of our Gluster volume
    }

    Access_type = RW;          # Export access permissions
    Squash = No_root_squash;   # Control NFS root squashing
    Disable_ACL = FALSE;       # Enable NFSv4 ACLs
    Pseudo = "/sharedvol";     # NFSv4 pseudo path for our NFS share
    Protocols = "3","4" ;      # NFS protocols supported
    Transports = "UDP","TCP" ; # Transport protocols supported
    SecType = "sys";           # NFS Security flavors supported
}
```

Create a Cluster

You will create and start a Pacemaker/Corosync cluster made of our three master nodes

(On all masters) Set a shared password of your choice for the user hacluster

```
passwd hacluster
```

(On all masters) Enable the Corosync and Pacemaker services. Enable and start the configuration system service 

```
systemctl enable corosync
systemctl enable pacemaker
systemctl enable --now pcsd
```

On master1: Authenticate with all cluster nodes using the hacluster user and password defined above

```
pcs host auth master1 master2 master3 -u hacluster -p examplepassword
```

On master1: Create a cluster named HA-NFS 

```
pcs cluster setup HA-NFS master1 master2 master3
```

On master1: Start the cluster on all nodes 

```
pcs cluster start --all
```

On master1: Enable the cluster to run on all nodes at boot time 

```
pcs cluster enable --all
``` 

On master1: Disable STONITH 
based on Wikipedia
> STONITH ("Shoot The Other Node In The Head" or "Shoot The Offending Node In The Head"), sometimes called STOMITH ("Shoot The Other Member/Machine In The Head"), is a technique for fencing in computer clusters.
Fencing is the isolation of a failed node so that it does not cause disruption to a computer cluster. As its name suggests, STONITH fences failed nodes by resetting or powering down the failed node. 

```
pcs property set stonith-enabled=false
```

Our cluster is now running

On any master by running the following command you should see the same output:

```
pcs cluster status
```

Create Cluster services

You will create a Pacemaker resource group that contains the resources necessary to host NFS services from the hostname nfs.vagrant.vm (192.168.99.100)

On master1: Create a systemd based cluster resource to ensure nfs-ganesha is running 
    
```
pcs resource create nfs_server systemd:nfs-ganesha op monitor interval=10s
```

On master1: Create a IP cluster resource used to present the NFS server 

```
pcs resource create nfs_ip ocf:heartbeat:IPaddr2 ip=192.168.99.100 cidr_netmask=24 op monitor interval=10s
```

On master1: Join the Ganesha service and IP resource in a group to ensure they remain together on the same host 

```
pcs resource group add nfs_group nfs_server nfs_ip
```

Our service is now running , you can make sure about that by running :

```
pcs status
```

You may test the availability of your service by following these steps,
On master1: Identify the host running the nfs_group resources and put it in standby mode to stop running services

```
pcs node standby master1
#Verify that the nfs_group resources have moved to another node
pcs status
#Bring our standby node back into the cluster
pcs node unstandby master1
```
At this point we have a virtual IP address which serves our distributed stroage service , with that in hand we can move on to the next stage.

## Virtualization:

Because we want to run our vms on a bridge mode network so we can access them from other nodes which reside in the same network and based on research and what articles over internet suggest you can't get ip address of vms in bridge mode easily through commands like ` virsh domifaddr [vm_name] ` so we need a dhcp server in place I went with dnsmasq because it provides the functionality of both a DHCP and a DNS server and it is lightweight nature.
I prepared a single cpu and 2 GB ram Centos8 ready system for running dnsmasq

```
yum install dnsmasq
systemctl start dnsmasq
systemctl enable dnsmasq
systemctl status dnsmasq
```

Configuring dnsmasq Server in CentOS and RHEL Linux

```
cp /etc/dnsmasq.conf /etc/dnsmasq.conf.orig
vi /etc/dnsmasq.conf 
```
change listen-address which is used to set the IP address, where dnsmasq will listen on

```
listen-address=::1,127.0.0.1,10.47.100.5 #change the second address with your dns server IP
interface=eth1 #you can restrict the interface dnsmasq listens on using the interface option
expand-hosts #If you want to have a domain (which you can set as shown next) automatically added to simple names in a hosts-file, uncomment the expand-hosts option.
domain=arvan.lan # set the domain name
# Google's nameservers ,upstream DNS server for non-local domains 
server=8.8.8.8
server=8.8.4.4
#force your local domain to an IP address(es) using the address option as shown
address=/arvan.lan/127.0.0.1 
address=/arvan.lan/10.47.100.5

#enable the DHCP server
dhcp-range=10.47.200.1,10.47.200.50,12h #set your ip range
dhcp-leasefile=/var/lib/dnsmasq/dnsmasq.leases # where DHCP server will keep its lease database
dhcp-authoritative

```

check the configuration file syntax for errors as shown

```
dnsmasq --test
```



Configuring dnsmasq with /etc/resolv.conf File

```
vi /etc/resolv.conf
nameserver 127.0.0.1
```

The /etc/resolv.conf file is maintained by a local daemon especially the NetworkManager, therefore any user-made changes will be overwritten. To prevent this

```
chattr +i /etc/resolv.conf
lsattr /etc/resolv.conf
```

 The Dnsmasq reads all the DNS hosts and names from the /etc/hosts file, so add your DNS hosts IP addresses and name pairs as shown.
 
```
compute01 10.47.100.3
compute02 10.47.100.4
```

To apply the above changes, restart the dnsmasq service as shown.

```
systemctl restart dnsmasq
```

Testing Local DNS

To test if the local DNS server or forwarding is working fine, you need to use tools such as dig or nslookup for performing DNS queries. 

```
yum install bind-utils # remember to install it on every compute node as well because we need it later on
dig arvan.lan
```

The VMs will not have any internet access because the default route in them is set to the ip address of our local DNS which is `10.47.100.5` in our case so in order to make sure DNS server act as a router we need to install iptables on it - centos8 does not have it by default - so follow along.

Enable ip forwarding

```
sysctl -w net.ipv4.ip_forward=1
```

Installing iptables 

```
sudo yum install iptables-services -y
sudo systemctl start iptables
sudo systemctl enable iptables
```

Set iptables rules accordingly

```
iptables -F # flush the default rules
iptables --table nat --append POSTROUTING --out-interface eth0 -j MASQUERADE
```


Virtual Machine Host Configuration
Each virtual machine should be setup identically, with only the hostname and local IP addresses being different. First, since we are putting the VMs on the same network segment as the host machine, we need to setup a network bridge. 

Bridged mode operates on Layer 2 of the OSI model. When used, all of the guest virtual machines will appear on the same subnet as the host physical machine. The most common use cases for bridged mode include:
Deploying guest virtual machines in an existing network alongside host physical machines making the difference between virtual and physical machines transparent to the end user.
Deploying guest virtual machines without making any changes to existing physical network configuration settings.
Deploying guest virtual machines which must be easily accessible to an existing physical network. Placing guest virtual machines on a physical network where they must access services within an existing broadcast domain, such as DHCP.
Connecting guest virtual machines to an exsting network where VLANs are used.


CentOS 8 uses NetworkManager as it's default networking daemon, so we will be configuring that.

```
sudo nmcli con add type bridge ifname br0
sudo nmcli con add type bridge-slave ifname eth1 master br0

#I will also disable Spanning Tree Protocol (STP) on the bridge to speed up network startup significantly. Make sure there are no loops in your network! If you can't remove the loops in your network, then you need to leave STP enabled.
sudo nmcli con modify bridge-br0 bridge.stp no
```

Now setup the IP configuration like your primary networking device, along with your DNS settings. If you don't know what to put in for ipv4.dns-search, then you don't need to set it. You would want it only if your home network uses a domain (my network is set to use arvan.lan).

```
sudo nmcli con modify bridge-br0 ipv4.addresses 10.47.100.3/16
sudo nmcli con modify bridge-br0 ipv4.gateway 10.47.0.1
sudo nmcli con modify bridge-br0 ipv4.dns 10.47.0.7
sudo nmcli con modify bridge-br0 ipv4.dns-search arvan.lan
```

If your network bridge slave device is being used already, then the bridge will not start. Simply activate the network slave device to bring up your bridge. If your network is misconfigured, this is where you may lose your SSH session!

```
sudo nmcli con up bridge-slave-eth1
```

Check to make sure everything is configured correctly.You might need to wait ~10 seconds for any input to be returned:

```
watch ip a show br1
```

Now we need to install and configure libvirt. 
```
dnf update
sudo yum install -y libvirt-daemon libvirt-admin libvirt-client \
  libvirt-daemon-kvm libvirt-daemon-driver-qemu \
  libvirt-daemon-driver-network libvirt-daemon-driver-storage \
  libvirt-daemon-driver-storage-core virt-install nfs-utils
sudo systemctl enable libvirtd
sudo systemctl start libvirtd
```

You need to disable selinux as default security driver because it gets in our way and we are testing it in a lab environment we can simply disable selinux completely or change the security driver settings as described below:

```
vi /etc/libvirt/qemu.conf 
#change 
security_driver = "none"
systemctl restart libvirtd
```

Now we need to setup the storage backend, with the local directory being at /data/vms:

```
sudo mkdir -p /data/vms
sudo virsh pool-define-as \
  --name vmstorage --type netfs \
  --source-host 192.168.99.100 --source-path /sharedvol \
  --source-format auto --target /data/vms
sudo virsh pool-autostart vmstorage
sudo virsh pool-start vmstorage
```


[According to RedHat](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/virtualization_administration_guide/chap-virtualization_administration_guide-storage_pools-storage_pools):

> Suppose a storage administrator responsible for an NFS server creates a share to store guest virtual machines' data. The system administrator defines a pool on the host physical machine with the details of the share (nfs.example.com:/path/to/share should be mounted on /vm_data). When the pool is started, libvirt mounts the share on the specified directory, just as if the system administrator logged in and executed mount nfs.example.com:/path/to/share /vmdata. If the pool is configured to autostart, libvirt ensures that the NFS share is mounted on the directory specified when libvirt is started.
Once the pool starts, the files that the NFS share, are reported as volumes, and the storage volumes' paths are then queried using the libvirt APIs. The volumes' paths can then be copied into the section of a guest virtual machine's XML definition file describing the source storage for the guest virtual machine's block devices. With NFS, applications using the libvirt APIs can create and delete volumes in the pool (files within the NFS share) up to the limit of the size of the pool (the maximum storage capacity of the share). Not all pool types support creating and deleting volumes. Stopping the pool negates the start operation, in this case, unmounts the NFS share. The data on the share is not modified by the destroy operation, despite the name. See man virsh for more details. 


At this point you can create working virtual machines manually, but that takes too long! I want to spin up virtual machines fast. To do that, we need to use cloud-init enabled images. I will be using the ubuntu Focal cloud-init image here as an example.


```
sudo curl -L \
http://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.img -o /data/vms/focal-server-cloudimg-amd64.qcow2
sudo virsh pool-refresh vmstorage
```

Don't use the base image directly! You want to create copies of that file for each VM, which can either be done with cp for normal copies, or qemu-img if you want to create a copy-on-write copy which reduces disk space significantly by only storing the difference between the base image and the VM.

```
sudo su -
git clone https://github.com/savi0r/Ministack.git
cd Ministack
cp -r libvirt-scripts /home/centos/
cd /home/centos/libvirt-scripts
```

for the sake of test run the script using the following command 

```
./create-vm.sh test arvan.lan 1 $((2*1024)) 10G \
  /data/vms/focal-server-cloudimg-amd64.qcow2 debian $(./gen-mac-address.sh)

virsh list # you must see your vm name in the ouput
```

