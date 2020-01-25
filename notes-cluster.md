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
```
pcs stonith create xvmfence_mds-01-1 fence_xvm pcmk_host_list="mds-01 mds-02" action="reboot" key_file=/etc/cluster/fence_xvm.key multicast_address=225.0.1.12
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
