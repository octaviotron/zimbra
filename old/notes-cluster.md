# Fencing Proxmox KVM host CUPS Cluster

This documentation is intended to have a full guide of how to install a CUPS service running in a corosync/pacemaker cluster. Servers are KVM hosts with CentOS 7 running in a Proxmox Hypervisor. One shared filesystem stores services common files and split brain condition will be avoided using STONITH through fence_pve agent.

This is our sample IP addeses:

```
cups01.domain.tld  192.168.0.1
cups02.domain.tld  192.168.0.2
cups03.domain.tld  192.168.0.3

proxmox.domain.tld  192.168.0.254
```

## Prepare Proxmox OS

Install Fence Agents On Proxmox KVM host, this is needed to be done on the KVM hypervisor Operating Sistem (in a root console in Proxmox host):
```
apt install fence-agents
```

Also, you need to add a shared storage resource enabled as a device for each virtual host you are going to configure as cluster. If the web UI does not let to make this, you will need to add it manually:

```
cd /etc/pve/qemu-server/
qm set 101 -ide1 /dev/sdX
```

Change "101" for the VM ID in proxmox and /dev/sdX for the filesystem device you want to link to VMs.

## Virtual Machines OS Preparation

In this example all virtual hosts are a fresh install of CentOS 7. 

This are common steps needed to be executed on each node (on all of them):

```
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm
yum -y update
yum -y upgrade
```

Install needed packages in all nodes too:

```
yum -y install ipa-client unzip net-tools sysstat openssh-clients \
    perl-core libaio nmap-ncat libstdc++.so.6 wget vim \
    pacemaker pcs corosync resource-agents pacemaker-cli fence-agents-all \
    git gcc make automake autoconf libtool pexpect python-requests
```

It is important to set an FQDN hostname. In cups01 node do:

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

Next, put the propper hostnames and ip in **/etc/hosts** in all nodes, when DNS service are unavailable, this helps to keep the cluster working:

```
192.168.0.1    cups01.domain.tld     cups01
192.168.0.2    cups02.domain.tld     cups02
192.168.0.3    cups03.domain.tld     cups03
192.168.0.254  proxmox.domain.tld    proxmox
```

Disable SELinux Policies in all nodes:
```
setenforce 0
```

Also disable SELinux for next reboots, changing the following line in **/etc/selinux/config** in all nodes:
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
(change "your_password" and remember it for later)

On all nodes start cluster system service:
```
systemctl start pcsd
systemctl status pcsd
```
Corosync service has a bug in CentOS 7 which sometimes starts corosync too early when some needed system resources are not already available. To avoid this, add a 3 seconds delay before service starts, to do this, in all nodes modify **/usr/lib/systemd/system/corosync.service** file adding "**ExecStartPre=/usr/bin/sleep 3**" after "**[service]**" section. The file must be as follow on all nodes:

```
[Service]
ExecStartPre=/usr/bin/sleep 3
ExecStart=/usr/share/corosync/corosync start
ExecStop=/usr/share/corosync/corosync stop
Type=forking
...
```

And in each node reload daemons after modify this file:

```
systemctl daemon-reload
```

## Install PVE fence agent 

CentOS does not include "pve" fencing agent, so is needed to compile it. Do it in each cluster node:

```
cd
git clone https://github.com/ClusterLabs/fence-agents.git
cd fence-agents/
./autogen.sh
./configure --with-agents=pve
make && make install
fence_pve --version
```

To test it, ask fence_pve from each cluster node. You will get a "STATUS: OK" message:
```
/usr/sbin/fence_pve --ip=<proxmox_ip> --username=root@pam --password=<proxmox_passwd> --plug=<vm_id> --action=status
```

## Create cluster


On any active node make auth keys share between nodes:
```
pcs cluster auth cups01.domain.tld cups02.domain.tld cups03.domain.tld
```

This will ask for a user and a password. Put "**hacluster**" as user and the password you set in previous steps.


Set a name for the cluster:

```
pcs cluster setup --name cluster_cups cups03.domain.tld cups03.domain.tld cups03.domain.tld
```

Start the cluster:
```
pcs cluster start --all
```

To check cluster status you can see the output of this commands:

```
pcs status cluster
corosync-cmapctl | grep members
pcs status corosync
```

Disable stonith. It is temporary, needed to configure and will be activated later:
```
pcs property set stonith-enabled=false
```

And disable the quorum policy, also later it will be activated:

```
pcs property set no-quorum-policy=ignore
pcs property
```

## Create cluster Virtual IP resource

Now, create a VIRTUAL IP resource:
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
virtual_ip     (ocf::heartbeat:IPaddr2):       Started cups01.domain.tls
```

This tells cups01 host has the virtual ip assigned. you can ping and open a SSH session to verify it.


# Create cluster CUPS daemon control resource

There is no a built-in cups resource available in heartbeat, so create an **/usr/lib/ocf/resource.d/heartbeat/cups_ctl** file in all nodes with this into it:

```bash
#!/bin/sh
#
# Resource script for CUPS
# Description:  Manages CUPS as an OCF resource in
#               an high-availability setup.
# Author:       Octavio Rossell <octavio@gnu.org.ve> modified version of RRMP <tigerlinux@gmail.com> work
# License:      GNU General Public License (GPL)
#
#       usage: $0 {start|stop|reload|monitor|validate-all|meta-data}
#       The "start" arg starts a CUPS instance
#       The "stop" arg stops it.
#
# OCF parameters:
#  OCF_RESKEY_binary
#  OCF_RESKEY_config_dir
#  OCF_RESKEY_parameters
#
##########################################################################

# Initialization:

: ${OCF_FUNCTIONS_DIR=${OCF_ROOT}/lib/heartbeat}
. ${OCF_FUNCTIONS_DIR}/ocf-shellfuncs

: ${OCF_RESKEY_cups_dir="/etc/cups"}
USAGE="Usage: $0 {start|stop|reload|status|monitor|validate-all|meta-data}";

##########################################################################

usage() {
	echo $USAGE >&2
}

meta_data() {
	cat <<END
<?xml version="1.0"?>
<!DOCTYPE resource-agent SYSTEM "ra-api-1.dtd">
<resource-agent name="cups">
<version>0.1</version>
<longdesc lang="en">
This script manages CUPS as an OCF resource in a high-availability setup.
</longdesc>
<shortdesc lang="en">Manages a highly available CUPS mail server instance</shortdesc>

<parameters>

<parameter name="binary" unique="0" required="0">
<longdesc lang="en">
Short name to the CUPS control script.
For example, "cuspinit" of "/etc/init.d/cupsd".
</longdesc>
<shortdesc lang="en">
Short name to the CUPS control script</shortdesc>
<content type="string" default="zmcontrol" />
</parameter>

<parameter name="cups_dir" unique="1" required="0">
<longdesc lang="en">
Full path to CUPS directory.
For example, "/etc/cups".
</longdesc>
<shortdesc lang="en">
Full path to CUPS directory</shortdesc>
<content type="string" default="/etc/cups" />
</parameter>

<parameter name="cups_user" unique="1" required="0">
<longdesc lang="en">
CUPS username.
For example, "cups".
</longdesc>
<shortdesc lang="en">CUPS username</shortdesc>
<content type="string" default="cups" />
</parameter>

<parameter name="cups_group"
 unique="1" required="0">
<longdesc lang="en">
CUPS group.
For example, "cups".
</longdesc>
<shortdesc lang="en">CUPS group</shortdesc>
<content type="string" default="cups" />
</parameter>

</parameters>

<actions>
<action name="start"   timeout="360s" />
<action name="stop"    timeout="360s" />
<action name="reload"  timeout="360s" />
<action name="monitor" depth="0"  timeout="40s"
 interval="60s" />
<action name="validate-all"  timeout="360s" />
<action name="meta-data"  timeout="5s" />
</actions>
</resource-agent>
END
}

case $1 in
meta-data)
	meta_data
	exit $OCF_SUCCESS
	;;

usage|help)
	usage
	exit $OCF_SUCCESS
	;;
start)
	echo "Starting CUPS Services"
	echo "0" > /var/log/db-svc-started.log
	rm -f /var/log/cups-svc-stopped.log
	systemctl start cups
	ocf_log info "CUPS started."
	exit $OCF_SUCCESS
	;;
stop)
	echo "Stopping CUPS Services"
	rm -f /var/log/db-svc-started.log
	echo "0" > /var/log/cups-svc-stopped.log
	systemctl stop cups
	/bin/killall -9 -u cups
	ocf_log info "CUPS stopped."
	exit $OCF_SUCCESS
	;;
status|monitor)
	echo "CUPS Services Status"
	if [ -f /var/log/cups-svc-started.log ]
	then
		exit $OCF_SUCCESS
	else
		exit $OCF_NOT_RUNNING
	fi
	;;
restart|reload)
	echo "CUPS Services Restart"
	ocf_log info "Reloading CUPS."
	systemctl stop cups
	/bin/killall -9 -u cups
	systemctl start cups
	exit $OCF_SUCCESS
	;;
validate-all)
	echo "Validating CUPS"
	exit $OCF_SUCCESS
	;;
*)
	usage
	exit $OCF_ERR_UNIMPLEMENTED
	;;
esac
```

And give 755 permission:

```
chmod 755 /usr/lib/ocf/resource.d/heartbeat/cups_ctl
```

Copy the resource file to the other nodes:
```
scp /usr/lib/ocf/resource.d/heartbeat/cups_ctl root@cups02.domain.tld:/usr/lib/ocf/resource.d/heartbeat/cups_ctl
scp /usr/lib/ocf/resource.d/heartbeat/cups_ctl root@cups03.domain.tld:/usr/lib/ocf/resource.d/heartbeat/cups_ctl
```

And in each node do:
```
chmod 755 /usr/lib/ocf/resource.d/heartbeat/cups_ctl
```

Now in any active node create cusp "svc_cups" resource:

```
pcs resource create svc_cups ocf:heartbeat:cups_ctl op monitor interval=30s
```

Disable resource monitor to make it more independent from cluster status:
```
pcs resource op remove svc_cups monitor
```

And set resource order:
```
pcs constraint colocation add svc_cups virtual_ip INFINITY
pcs constraint order virtual_ip then svc_cups
```

Now "svc_cups" is a cluster resource. It will be present only if virtual IP is activated. 

To check if it is loaded as cluster resource:
```
pcs status
```

# Create shared filesystem resource

Create shared filesystem resource. It will make /etc/cups service files mounted only in the active node:

In any active node do:

```
cd /
pcs cluster cib add_fs
pcs -f add_fs resource create shared_fs Filesystem device="/dev/sdX" directory="/etc/cups" fstype="ext4"
pcs -f add_fs constraint colocation add svc_cups shared_fs INFINITY
pcs -f add_fs constraint order shared_fs then svc_cups
pcs cluster cib-push add_fs
```

And on each node enable services:

```
systemctl enable pcsd
systemctl enable corosync
systemctl enable pacemaker
```


## Configure Fencing resources

When a node fails (loose connection, hangs, crash, etc) pacemaker needs to fence it. In the following example is created a "fence_cups01" stonith rule, calling Proxmox KVM system to make actions over "cups01.domain.tld" virtual machine:


On any active node make the stonith rules for each one:

```
pcs stonith create fence_cups01 fence_pve ipaddr=<proxmox_ip> inet4_only="true" vmtype="qemu" \
  login="root@pam" passwd=<proxmox_passwd> delay="15" port=<vm_id> pcmk_host_check=static-list \
  pcmk_host_list="cups01.domain.tld" node_name="pve"

pcs stonith create fence_cups02 fence_pve ipaddr=<proxmox_ip> inet4_only="true" vmtype="qemu" \
  login="root@pam" passwd=<proxmox_passwd> delay="15" port=<vm_id> pcmk_host_check=static-list \
  pcmk_host_list="cups02.domain.tld" node_name="pve"

pcs stonith create fence_cups03 fence_pve ipaddr=<proxmox_ip> inet4_only="true" vmtype="qemu" \
  login="root@pam" passwd=<proxmox_passwd> delay="15" port=<vm_id> pcmk_host_check=static-list \
  pcmk_host_list="cups03.domain.tld" node_name="pve"
```

Next, set the stonith resource to be (when possible) active in its own node. This is optional, but allows a node to call its hypervisor to shut it down, there are environments (a proxmox cluster where VM can be running in different hosts) where this si the most secure configuration to ensure sucessful stonith:
```
pcs constraint location fence_cups01 prefers cups01.domain.tld
pcs constraint location fence_cups02 prefers cups02.domain.tld
pcs constraint location fence_cups03 prefers cups03.domain.tld
```

For a node to be online it must see at least one more node, it is, needs a quorum > 1. Enable stonith in cluster, set a shut down action when quorum is not satisfied:

```
pcs property set stonith-enabled=true
pcs property set no-quorum-policy=suicide
pcs stonith update fence_cups01 action="off" --force
pcs stonith update fence_cups02 action="off" --force
pcs stonith update fence_cups03 action="off" --force
```

Resources and stonith actions are now completely configured. Restart all cluster services in all nodes (execute the following in each one):
```
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



## Links (consulted documentation):

- https://www.alteeve.com/w/Fencing_KVM_Virtual_Servers
- https://access.redhat.com/solutions/293183
- https://clusterlabs.org/pacemaker/doc/en-US/Pacemaker/1.1/html/Clusters_from_Scratch/_configure_the_cluster_for_stonith.html
- Proxmox: servicio fence_virtd https://www.lisenet.com/2018/libvirt-fencing-on-a-physical-kvm-host/
- Proxmox: agente fence_virsh https://linux.die.net/man/8/fence_virsh
- CentOS VMs: agente fence_xvm https://www.alteeve.com/w/Fencing_KVM_Virtual_Servers
- KVM fencing: https://www.alteeve.com/w/Fencing_KVM_Virtual_Servers
- Fencing Pacemaker: https://www.unixarena.com/2016/01/rhel-7-configure-fencing-pacemaker.html/
- How to configure Red Hat Cluster with fencing of two KVM guests running on two different KVM hosts https://access.redhat.com/solutions/293183 
- https://icicimov.github.io/blog/virtualization/Pacemaker-VM-cluster-fencing-in-Proxmox-with-fence-pve/
- https://www.lisenet.com/2018/libvirt-fencing-on-a-physical-kvm-host/
- https://www.epilis.gr/en/blog/2018/07/02/fencing-linux-vm-cluster/
- https://clusterlabs.org/pacemaker/doc/en-US/Pacemaker/1.1/html/Clusters_from_Scratch/_configure_the_cluster_for_stonith.html

## NOTES

The following command shows the quorum configuration.
```
pcs quorum
pcs quorum [config]
pcs quorum expected-votes 2
```

The following command shows the quorum runtime status:
```
pcs quorum status
```

Quorum information:
```
corosync-quorumtool 
```



