# Proxied Zimbra Cluster

**Zimbra 8.8 (ZCS) in active-passive cluster, with multiple Proxy Servers and external LDAP (FreeIPA) accounts provisioning**

This documentation is intended to have a full guide of how to install one server with Zimbra Community Suite (8.8.15 LTS) and many Zimbra-Proxy Servers, with external LDAP provisioning (FreeIPA in this case).

Theese are the hostnames and IPs we will use:
```
freeipa.domain.tld	192.168.0.1 	(LDAP service)
zimbra01.domain.tld 	192.168.0.2	(Zimbra Active Server)
zimbra02.domain.tld	192.168.0.3	(Zimbra Passive Server)
proxy1.domain.tld 	192.168.0.4	(first zimbra-proxy server)
proxy2.domain.tld 	192.168.0.5	(second zimbra-proxy server)
```

Also, you will need a free IP address to assign virtual IP when configuring the cluster (192.168.0.254, for example)

Harrdware Requirements for Zimbra Servers:

- 3 servers (may be virtual) with 8+ CPU cores, 16+GB RAM and 20+GB Free Space
- SAN/NAS storage for mailboxes in cluster servers with about 1TB of free space (depending on your traffic)


## Common Process (for all servers)

This are the needed steps for installing all zimbra nodes (both mailbox and proxies):

1) First, install Centos 7 with support for English and your native language (if any) and update all your systems: 

```
yum -y update
```

2) It is important to set an FQDN hostname:

```
hostnamectl set-hostname "mail.domain.tld" && exec bash
```

3) Next, put the propper hostname and ip in /etc/hosts

```
192.168.0.1     mail.domain.tld     mail
```

5) Disable SELinux Policies in all systems:

First, disable SELinux in the current running system:

```
setenforce 0
```

Then, disable it in the next boot, changing the following line in /etc/selinux/config
```
SELINUX=permissive
```

6) OPTIONAL: If you are using FreeIPA as LDAP external service, it is necessary to install the IPA agent and enroll the system:

```
yum -y install ipa-client
ipa-client-install --enable-dns-updates
```

7) DNS records have to give propper answer to MX requests:

If using FreeIPA, next step is needed, otherwise add MX record in whichever DNS server are you using:

```
kinit admin
ipa dnsrecord-add domain.tld @ --mx-rec="0 mail.domain.tld."
```

To verify MX record, ask the DNS:

```
dig @freeipa.domain.tld domain.tld mx
```

8) Install system dependencies:

```
yum install unzip net-tools sysstat openssh-clients perl-core libaio nmap-ncat libstdc++.so.6 wget -y
```

9) Disable postfix: 

By default CentOS has a postfix running service. It will be needed to disable in order to make IP 25/tcp port available.

```
systemctl stop postfix
systemctl disable postfix
```

9) Enable prots in Firewall:

```
firewall-cmd --permanent --add-port={25,80,110,143,389,443,465,587,993,995,5222,5223,9071,7071}/tcp
firewall-cmd --reload
```

It is possible you need some other ports, depending services you want to have in Zimbra Server. In this case you can temporary disable the firewall for testing and later decide which services you need to add. To make this:

```
systemctl stop firewalld
systemctl disable firewalld
```

Remember when you finnish all processes to enable it again:

```
systemctl start firewalld
systemctl enable firewalld
```

10) Download and Install the Software:

```
mkdir /root/zimbra && cd /root/zimbra
wget https://files.zimbra.com/downloads/8.8.15_GA/zcs-8.8.15_GA_3869.RHEL7_64.20190918004220.tgz
tar zxpf zcs-8.8.15_GA_3869.RHEL7_64.20190918004220.tgz
cd zcs-8.8.15_GA_3869.RHEL7_64.20190918004220
./install.sh -s
```
Note "-s" option: it will install the software without configure it. We will make it later.


## Installing MASTER Zimbra Server 

At this point, you will need to have SAN/NAS storage resource mounted on **/opt/zimbra**, Otherwise cluser will not work when passive server takes requests from your clients when master server fails.

If you don't have SAN/NAS you can use a dedicaed RAID partition for this. On PROXMOX VMs, you can do:

```
cd /etc/pve/qemu-server/
qm set 101 -ide1 /dev/md0
```

This will add /dev/md0 to 101 VM id

11) The instaler will ask you several questions, choose the following options:

```
  Do you agree with the terms of the software license agreement? [N] y
  Use Zimbra's package repository [Y]
  Install zimbra-ldap [Y] 
  Install zimbra-logger [Y] 
  Install zimbra-mta [Y] 
  Install zimbra-dnscache [Y] 
  Install zimbra-snmp [Y] 
  Install zimbra-store [Y] 
  Install zimbra-apache [Y] 
  Install zimbra-spell [Y] 
  Install zimbra-memcached [Y] 
  Install zimbra-proxy [Y] 
  Install zimbra-drive [Y] 
  Install zimbra-imapd (BETA - for evaluation only) [N]     <--- press ENTER
  Install zimbra-chat [Y] N  <----- choose N
  The system will be modified.  Continue? [N] Y
```

Note the "N" option in both zimbra-imapd and zimbra-chat. The impad component is BETA and is not recommemded for production environments, and Zimbra Chat provided with 8.8.15 installer provides a buggy zimlet: we will install a fixed one later.

Zimbra will download updates and pathces, go for a coffe, because it is Java and all Java stuff always delays a lot, even running simple procecess.

12) Fix CA paths:

```
mkdir -p /opt/zimbra/java/jre/lib/security/
ln -s /opt/zimbra/common/etc/java/cacerts /opt/zimbra/java/jre/lib/security/cacerts
chown -R  zimbra.zimbra /opt/zimbra/java
```

Then run the installer configuration:

```
/opt/zimbra/libexec/zmsetup.pl
```

You may get an error message like this, informing your about resolving MX record, so you will need to change it an give the right domain name. Rememeber MX record points to mail.domain.tld and not zimbra01.domain.tld:

```
DNS ERROR resolving MX for zimbra01.domain.tld
It is suggested that the domain name have an MX record configured in DNS
Change domain name? [Yes] <---- press ENTER
Create domain: [zimbra01.domain.tld] domain.tld <----- your MX host here
```


12) Set Zimbra Admin Password:

When prompt shows text **"Address unconfigured (++) items (? - help)"** press 7 **zimbra-store** and ENTER, then 4 **Admin Password** and ENTER

After it press ENTER in "Select, or 'r' for previous menu [r]" prompt message to go to main menu

13) Set Domain in LDAP:

If you skip this step, your domain name will be "zimbra01.domain.tld" so mailboxes will have addresses like "user@zimbra01.domain.tld" and you maybe preffer to have "user@domain.tld" mail accounts instead, so change the default config:

Go to option 2 **"zimbra-ldap"** and then option 3 **"Domain to create"** and change default domain to "domain.tld"

14) Install Zimbra server:

When you have set it, return to main menu pressing ENTER in "Select, or 'r' for previous menu [r]" prompt message.

```
  Select from menu, or press 'a' to apply config (? - help) a
  Save configuration data to a file? [Yes]
  Save config in file: [/opt/zimbra/config.21593]
  The system will be modified - continue? [No] Yes
```

Zimbra will start to install, go for a coffe, because it uses Java and all Java always delay a lot even in simple procecess. When completed, you will have this on prompt:

```
  Notify Zimbra of your installation? [Yes]
  Configuration complete - press return to exit
```

Now reboot the system:

```
  init 6
```

16) Open a root shell account and start all zimbra services:

```
su - zimbra
zmcontrol start
```

Test if all services are running OK:

```
zmcontrol status
```

17) Set FREE IPA LDAP Auto-Provision:

This step is required to have all external LDAP accounts available in Zimbra MailBox:

 17.1) Open https://mail.domain.tld:7071 to get into Zimbra Admin Interface.
 17.2) Go to "Admin" > "Configuration" > "Domain"
 17.3) Click in "domain.tld" in domain list.
 17.4) Go to "Authentication" in the left menu
 17.5) Click on the gear icon on the top right corner and select "Autentication"
 17.6) Options:
   17.6.1) Use external LDAP (click "next")
   17.6.2) Put the LDAP (FreeIPA) hostname or IP
   17.6.3) Put "(uid=%u)" into Filter Option (without quotation marks)
   17.6.4) Put "cn=users,cn=accounts,dc=domain,dc=tld" on Base DN (change domain components to fit yours)
   17.6.5) Click Next
   17.6.6) Optionally put DN variables if you have configured it in your LDAP server
   17.6.7) Test your LDAP connection using a user/password
   17.6.8) Finnish the auth config dialog
 17.7) Open a root shell in Zimbra server and write next commands:

```
su - zimbra
zmprov md prue.ba +zimbraAutoProvAttrMap description=description
zmprov md prue.ba +zimbraAutoProvAttrMap displayName=displayName
zmprov md prue.ba +zimbraAutoProvAttrMap givenName=givenName
zmprov md prue.ba +zimbraAutoProvAttrMap cn=cn
zmprov md prue.ba +zimbraAutoProvAttrMap sn=sn
zmprov md prue.ba zimbraAutoProvAuthMech LDAP
zmprov md prue.ba zimbraAutoProvLdapSearchBase "cn=accounts,dc=domain,dc=tld"
zmprov md prue.ba zimbraAutoProvLdapSearchFilter "(uid=%u)"
zmprov md prue.ba zimbraAutoProvLdapURL "ldap://192.168.0.100:389"
zmprov md prue.ba zimbraAutoProvMode LAZY
zmprov md prue.ba zimbraAutoProvNotificationBody "Your account has been auto provisioned.  Your email address is ${ACCOUNT_ADDRESS}."
zmprov md prue.ba zimbraAutoProvNotificationFromAddress prov-admin@prue.ba
zmprov md prue.ba zimbraAutoProvNotificationSubject "New account auto provisioned"
```
Remember to change "cn=accounts,dc=domain,dc=tld" and "ldap://192.168.0.100:389" accornding to your needs

Thats is ! you have configured a Zimbra Server with external LDAP accounts

DIsable TLS:
```
zmlocalconfig -e ssl_allow_untrusted_certs=true 
zmlocalconfig -e ldap_starttls_supported=0
zmlocalconfig -e ldap_starttls_required=false
zmlocalconfig -e ldap_common_require_tls=0
zmcontrol restart
```
## Install Zimbra Proxy Server

Repeat common process steps, but in ./install.sh choose only to install memcached and proxy components: 
```
Do you agree with the terms of the software license agreement? [N] Y
Use Zimbra's package repository [Y]
Install zimbra-ldap [Y] N
Install zimbra-logger [Y] N
Install zimbra-mta [Y] N
Install zimbra-dnscache [N] N
Install zimbra-snmp [Y] N
Install zimbra-store [Y] N
Install zimbra-apache [Y] N
Install zimbra-spell [Y] N
**Install** **zimbra-memcached** **[Y]** **Y**
**Install** **zimbra-proxy** **[Y]** **Y**
The system will be modified.  Continue? [N] Y
```

1: 2) Ldap master host
   4) Ldap Admin password

2: 12) Bind password for nginx ldap user
zmlocalconfig -s ldap_nginx_password
ldap_nginx_password = zxsARJ8G

*** CONFIGURATION COMPLETE - press 'a' to apply
Select from menu, or press 'a' to apply config (? - help) a
Save configuration data to a file? [Yes]
Save config in file: [/opt/zimbra/config.15941] 
Saving config in /opt/zimbra/config.15941...done.
The system will be modified - continue? [No] Yes
Notify Zimbra of your installation? [Yes]

Configuration complete - press return to exit 

init 6

## important

Init at boot:

```
update-rc.d zimbra defaults
```

On all zimbra servers do:

```
zmsshkeygen
zmupdateauthkeys
```

Disable TLS:
```
zmlocalconfig -e ssl_allow_untrusted_certs=true 
zmlocalconfig -e ldap_starttls_supported=0
zmlocalconfig -e ldap_starttls_required=false
zmlocalconfig -e ldap_common_require_tls=0
zmcontrol restart
```

Enable Logs:
```
/opt/zimbra/libexec/zmsyslogsetup

/etc/sysconfig/rsyslog: "-r" 

    Uncomment the following lines in /etc/rsyslog.conf

    $modload imudp
    $UDPServerRun 514

    Restart rsyslog

For rsyslog on RHEL or CentOS:

    Uncomment the following lines in /etc/rsyslog.conf.

    # Provides UDP syslog reception
    #$ModLoad imudp
    #$UDPServerRun 514

    # Provides TCP syslog reception
    #$ModLoad imtcp
    #$InputTCPServerRun 514
```


16) OPTIONAL: configure OpenDKIM service:

Open a root console in Zimbra server and put the following commands:

```
/opt/zimbra/libexec/zmdkimkeyutil -a -d domain.tld


35BC9092-38DA-11EA-965D-1D43A6F2AEB9._domainkey	IN	TXT	( "v=DKIM1; k=rsa; "
	  "p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA1nSZ6IAfTfZVjvcYgAd0lZpIupoxuMpEmT/2+QedgukuUBVP9kYLVVI0cUxpnXDgtKpRPNhQtVAATn2KFGySABIx3Jin8EU3/FSYGYMQM9BKzTjM3HfueFIJFF5kzmd5FgLgHHOY2C6EPMWI/GFBpDs3QrcA9J/7HCgqYESA9DmT+9JhsnLeVaj2/3X09xfSPHv/A8Avp74aCm"
	  "i4h0LplL3TeCpWTti2nQgkWdTNsj3Oh0EICHEupLkv0bAB5CiSeXTqPkMQ/lGdr2F6T9l5sb1jISVRiWbOhzaN1A5HDDBcU3v2Tb/LToyKi937BdPysKh3+QFP4jdcKpvcf+/CnQIDAQAB" )  ; ----- DKIM key 35BC9092-38DA-11EA-965D-1D43A6F2AEB9 for prue.ba



registro: 35BC9092-38DA-11EA-965D-1D43A6F2AEB9._domainkey
tipo: TXT
valor: v=DKIM1; k=rsa; p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA1nSZ6IAfTfZVjvcYgAd0lZpIupoxuMpEmT/2+QedgukuUBVP9kYLVVI0cUxpnXDgtKpRPNhQtVAATn2KFGySABIx3Jin8EU3/FSYGYMQM9BKzTjM3HfueFIJFF5kzmd5FgLgHHOY2C6EPMWI/GFBpDs3QrcA9J/7HCgqYESA9DmT+9JhsnLeVaj2/3X09xfSPHv/A8Avp74aCmi4h0LplL3TeCpWTti2nQgkWdTNsj3Oh0EICHEupLkv0bAB5CiSeXTqPkMQ/lGdr2F6T9l5sb1jISVRiWbOhzaN1A5HDDBcU3v2Tb/LToyKi937BdPysKh3+QFP4jdcKpvcf+/CnQIDAQAB

```





PROXY:
 en el proxy:
	/opt/zimbra/libexec/zmproxyconfig -e -m -H proxy.node.service.hostname

 en el servidor mailbox: 
	restart mailboxd

	/opt/zimbra/libexec/zmproxyconfig -e -m -H mailbox.node.service.hostname
	/opt/zimbra/libexec/zmproxyconfig -e -w -H mailbox.node.service.hostname

Used documentation links:

* Zimbra Installation Guide: https://zimbra.github.io/installguides/8.8.12/multi.html
* Zimbra Admin Guide: https://zimbra.github.io/adminguide/8.8.15/adminguide-8.8.15.pdf
* Mulli Server Guide: https://zimbra.github.io/installguides/latest/multi.html

* zmprov attributes: https://wiki.zimbra.com/wiki/How_to_get_details_of_zmprov_attributes/zmlocalconfig_attributes

REVISAR: https://github.com/tigerlinux/tigerlinux-extra-recipes/tree/master/recipes/ispapps/zimbra-cluster-centos7

* https://www.zimbra.com/docs/ne/8.6.0/multi_server_install/wwhelp/wwhimpl/js/html/wwhelp.htm#href=multi_server_install.Multiple-Server_Installation.html
* https://zimbra.github.io/installguides/8.8.9/multi.html
* https://www.zimbra.com/docs/os/5.0.0/administration_guide/3_Zimbra%20Servers.4.1.html
* https://wiki.zimbra.com/wiki/Open_Source_Edition_Backup_Procedure
* https://wiki.zimbra.com/wiki/How_to_configure_autoprovisioning_with_external_LDAP
* https://linoxide.com/linux-how-to/howto-install-configure-zimbra-8-6-multi-server-centos-7/
* https://computingforgeeks.com/zimbra-multi-server-installation-on-centos-7/
* https://www.zimbra.com/docs/ne/4.0.5/multi_server_install/clustering.7.1.html#1066634

BACKUP: https://forums.zimbra.org/viewtopic.php?t=65435
	https://wiki.zimbra.com/wiki/Zimbra_DR_Strategy


DKIM: https://wiki.zimbra.com/wiki/Best_Practices_on_Email_Protection:_SPF,_DKIM_and_DMARC

Zimbra Desktop: https://wiki.zimbra.com/wiki/Installing_Zimbra_Desktop_on_64bit_Linux
		https://computingforgeeks.com/how-to-install-zimbra-desktop-on-ubuntu-18-04-bionic-beaver/

PROXY:	https://wiki.zimbra.com/wiki/Zimbra_Proxy_Guide

SYSTEMD: https://github.com/Zimbra-Community/zimbra-tools/blob/master/zimbra.service

LOGS: https://wiki.zimbra.com/wiki/Using_log4j_to_Configure_mailboxd_Logging

# Notes

- zmprov ms mail.domain.tld zimbraMtaMyNetworks "127.0.0.0/8 10.0.0.0/24 [::1]/128 [fe80::]/64"
- Get LDAP password: zmlocalconfig -s zimbra_ldap_password ldap_master_url
- Show config parameter: zmprov gcf [parameter]


