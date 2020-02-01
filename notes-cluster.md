# Fencing Proxmox Virtual KVM hosts Cluster

This documentation is intended to have a full guide of how to install a corosync/pacemaker cluster of KVM hosts with CentOS 7, running in a Proxmox Hypervisor with STONITH using fence_pve agent.

This is our sample IP addeses:

```
cups01.domain.tld   192.168.0.1
cups02.domain.tld	192.168.0.2
cups03.domain.tld	192.168.0.3

proxmox.domain.tld  192.168.0.254
```

## On hypervisor:
```
apt install fence-agents
```

## OS Preparation

In this example all KVM hosts has a fresh install of CentOS 7. This are common steps needed to be executed in all nodes:

```
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm
yum -y update
yum -y upgrade
```

Install all needed packages:

```
yum -y install ipa-client unzip net-tools sysstat openssh-clients \
    perl-core libaio nmap-ncat libstdc++.so.6 wget vim \
    pacemaker pcs corosync resource-agents pacemaker-cli fence-agents-all \
    git gcc make automake autoconf libtool pexpect python-requests
```

It is important to set an FQDN hostname. In cups01 do:

```
hostnamectl set-hostname "cups01.domain.tld" && exec bash 
```

In cups02 do:

```
hostnamectl set-hostname "cups02.domain.tld" && exec bash 
```
And in cups03 do:

```
hostnamectl set-hostname "cups03.domain.tld" && exec bash 
```

Next, put the propper hostname and ip in /etc/hosts in all nodes, when DNS service are unavailable, this helps to keep the cluster working:

```
192.168.0.1    cups01.domain.tld     cups01
192.168.0.2    cups02.domain.tld     cups02
192.168.0.3    cups03.domain.tld     cups03
```

Disable SELinux Policies in all nodes:
```
setenforce 0
```

Then, disable it for next reboots, changing the following line in **/etc/selinux/config** in all nodes:
```
SELINUX=permissive
```

Enable prots in Firewall:
```
firewall-cmd --permanent --add-port={25,80,110,143,389,443,465,587,993,995,5222,5223,9071,7071}/tcp
firewall-cmd --reload
```

It is possible you need some other ports, depending services you want to install. In this case you can temporary disable the firewall for testing and later decide which services you need to add. To make this in all nodes:

```
systemctl stop firewalld
systemctl disable firewalld
```

Remember later when you finnish to enable it again:
```
systemctl start firewalld
systemctl enable firewalld
```

Set the "hacluster" account password in al nodes:

```
echo "hacluster:your_password"|chpasswd
```
(change "your_password" and put there your password)

On all nodes start the cluster:
```
systemctl start pcsd
systemctl status pcsd
```
Corosync service has a bug in CentOS 7, so to avoid id is needed to add a 3 seconds delay, so on all nodes modify **/usr/lib/systemd/system/corosync.service** file adding "ExecStartPre=/usr/bin/sleep 10" after "[service]" line. The file section must be as follow on all nodes:

```
[Service]
ExecStartPre=/usr/bin/sleep 3
ExecStart=/usr/share/corosync/corosync start
ExecStop=/usr/share/corosync/corosync stop
Type=forking
```

And in each node reload daemons after modify this file:

```
systemctl daemon-reload
```

## Install PVE fence agent 

In all nodes:

```
cd
git clone https://github.com/ClusterLabs/fence-agents.git
cd fence-agents/
./autogen.sh
./configure --with-agents=pve
make && make install
fence_pve --version
```

Ask fence_pve from a cluster node:
```
/usr/sbin/fence_pve --ip=<proxmox_ip> --username=root@pam --password=<proxmox_passwd> --plug=<vm_id> --action=status
```

## Create cluster resources


On any active node:
```
pcs cluster auth cups01.domain.tld cups02.domain.tld cups03.domain.tld
```
Create the cluster:

```
pcs cluster setup --name cluster_zimbra zimbra01 zimbra02
pcs cluster start --all
```

Check cluster status:

```
pcs status cluster
corosync-cmapctl | grep members
pcs status corosync
```

Disable stonith. It is temporary, needed to configure and activated later:
```
pcs property set stonith-enabled=false
```

And disable the quorum policy, also later will be reactivated:

```
pcs property set no-quorum-policy=ignore
pcs property
```

Now, create the VIRTUAL IP resource:
```
pcs resource create virtual_ip ocf:heartbeat:IPaddr2 ip=192.168.0.254 cidr_netmask=32 nic=eth0:0 op monitor interval=30s
```

We are using "eth0" here to create "eth0:0" alias. Please verify your network interface is this or rename as needed.

Verify the creation of the virtual IP:
```
pcs status resources
```

You will get a message with a line like this:
```
virtual_ip     (ocf::heartbeat:IPaddr2):       Started instance-172-16-70-51
```

On each node enable services:

```
systemctl enable pcsd
systemctl enable corosync
systemctl enable pacemaker
```


## Configure Fencing resources


On any active node:

```
pcs stonith create fence_host01_id fence_pve ipaddr=<proxmox_ip> inet4_only="true" vmtype="qemu" \
  login="root@pam" passwd=<proxmox_passwd> delay="15" port=<vm_id> pcmk_host_check=static-list \
  pcmk_host_list="hostname01.domain.tld" node_name="pve"
  
pcs constraint location fence_host01_id prefers hostname01.domain.tld

(repeat for each node)

pcs property set stonith-enabled=true
pcs property set no-quorum-policy=suicide
pcs stonith update fence_host01_id action="off" --force

systemctl enable pcsd
systemctl enable corosync
systemctl enable pacemaker

```

Now, test the cluster fencing, disconecting one node, turning off the network interface:

```
systemctl stop networking
```

Check the cluster status, when this node loose comunication with the cluster, the fencing agent will send a signal to VM hypervisor and a STOINITH will be done over this absent node. You can watch the process seeing happening changes:

```
watch pcs status
```
(CONTROL-C to exit)


## Old notes


Node 1:
```
pcs cluster auth cups01.prue.ba cups02.prue.ba cups03.prue.ba
pcs cluster setup --name cluster_cups cups01.prue.ba cups02.prue.ba cups03.prue.ba
pcs cluster start --all
pcs status cluster

pcs cluster cib stonith_cfg
pcs -f stonith_cfg stonith create MyFence03 fence_virt

pcs stonith create MyFence1 fence_virt port="cups01" pcmk_host_list="cups01.prue.ba"
pcs stonith create MyFence2 fence_virt port="cups02" pcmk_host_list="cups02.prue.ba"
pcs stonith create MyFence3 fence_virt port="cups03" pcmk_host_list="cups03.prue.ba"

pcs stonith confirm cups02.prue.ba

```

On KVM host:
```
mkdir -p /etc/cluster
cd /etc/cluster/
dd if=/dev/urandom of=/etc/cluster/fence_xvm.key bs=4k count=1

```

votequorum:
Quorum = 2

3rd node just for quorum <-- !

no-quorum-policy property:

    ignore - Do nothing when quorum is lost.
    stop (default) - stop all resources in the affected cluster partition.
    freeze - continue running existing resources, but don’t start any stopped ones.
    suicide - fence all nodes in the affected partition.

The configuration file that holds the quorum-related options of the cluster is /etc/corosync/corosync.conf. All quorum-related options are set in the quorum directive. The default quorum directive in /etc/corosync/corosync. conf without any special options set is as follows:

quorum { 
provider: corosync_votequorum 
}

 The following command shows the quorum configuration.
pcs quorum [config]
pcs quorum expected-votes 2

The following command shows the quorum runtime status.
pcs quorum status

# corosync-quorumtool 
Quorum information 

Remote management (for stonith)
Fujitsu servers have IPMI, HP servers have iLO, Dell servers have DRAC,
IBM servers have RSA and so on
Some are IBM servers mentioning VMK (optional) and others are Fujitsu mentioning iRMC. 
IPMI, iLO, RSA, iDRAC and the like.
Alternatives are switched PDUs, like APC's AP7900. 

## Fencing:

- https://www.alteeve.com/w/Fencing_KVM_Virtual_Servers
- https://access.redhat.com/solutions/293183
- https://clusterlabs.org/pacemaker/doc/en-US/Pacemaker/1.1/html/Clusters_from_Scratch/_configure_the_cluster_for_stonith.html

```
pcs stonith create xvmfence_mds-01-1 fence_xvm pcmk_host_list="mds-01 mds-02" action="reboot" key_file=/etc/cluster/fence_xvm.key multicast_address=225.0.1.12


pcs stonith create fence_pcmk1_xvm fence_xvm port="pcmk1" pcmk_host_list="pcmk1.alteeve.ca"
pcs stonith create fence_pcmk2_xvm fence_xvm port="pcmk2" pcmk_host_list="pcmk2.alteeve.ca"
pcs stonith create fence_pcmk3_xvm fence_xvm port="pcmk3" pcmk_host_list="pcmk3.alteeve.ca"
```

- Proxmox: servicio fence_virtd https://www.lisenet.com/2018/libvirt-fencing-on-a-physical-kvm-host/
- Proxmox: agente fence_virsh https://linux.die.net/man/8/fence_virsh
- CentOS VMs: agente fence_xvm https://www.alteeve.com/w/Fencing_KVM_Virtual_Servers


- KVM fencing: https://www.alteeve.com/w/Fencing_KVM_Virtual_Servers
- Fencing Pacemaker: https://www.unixarena.com/2016/01/rhel-7-configure-fencing-pacemaker.html/

## KVM-PACEMAKER
- How to configure Red Hat Cluster with fencing of two KVM guests running on two different KVM hosts https://access.redhat.com/solutions/293183 
- https://icicimov.github.io/blog/virtualization/Pacemaker-VM-cluster-fencing-in-Proxmox-with-fence-pve/
- https://www.lisenet.com/2018/libvirt-fencing-on-a-physical-kvm-host/
                  
               
- https://www.epilis.gr/en/blog/2018/07/02/fencing-linux-vm-cluster/

Stonith:
- https://clusterlabs.org/pacemaker/doc/en-US/Pacemaker/1.1/html/Clusters_from_Scratch/_configure_the_cluster_for_stonith.html













alteeve.com/w/Fencing_KVM_Virtual_Servers

wiki.clusterlabs.org/wiki/Guest_Fencing

3 proxmox hypervisor: access.redhat.com/solutions/293183

MAN:

redhatlinux.guru/2018/05/22/pacemaker-cheat-sheet

linux.die.net/man/8/fence_virt

DOCS

pve.proxmox.com/wiki/Two-Node_High_Availability_Cluster

pve.proxmox.com/wiki/Fencing

linux-ha.org/wiki/OCF_Resource_Agents

epilis.gr/en/blog/2018/07/02/fencing-linux-vm-cluster

lisenet.com/2018/libvirt-fencing-on-a-physical-kvm-host

clusterlabs.org/pacemaker/doc/en-US/Pacemaker/2.0/html/Pacemaker_Explained/ch04.html

clusterlabs.org/pacemaker/doc/en-US/Pacemaker/2.0/html/Pacemaker_Explained/_special_treatment_of_stonith_resources.html

clusterlabs.org/pacemaker/doc/en-US/Pacemaker/2.0/html/Clusters_from_Scratch/_configure_the_cluster_for_stonith.html

clusterlabs.org/pacemaker/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/_differences_of_stonith_resources.html

access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/configuring_the_red_hat_high_availability_add-on_with_pacemaker/ch-fencing-haar

clusterlabs.org/pacemaker/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/s-failure-handling.html

lisenet.com/2015/active-passive-cluster-with-pacemaker-corosync-and-drbd-on-centos-7-part-4

itenlightens.blogspot.com/2017/03/pacemaker-cluster-on-rhel7-with-virtual.html

POLICY:

access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/configuring_the_red_hat_high_availability_add-on_with_pacemaker/s1-resourceoperate-haar

ninjaskills.in/2017/09/pacemaker-cluster-on-rhel7-with-virtual.html

doc.pegasi.fi/wiki/doku.php?id=pacemaker_cluster
