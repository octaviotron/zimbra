# Cluster de Zimbra Community Suite en hosts KVM (Proxmox)

(english version in https://github.com/octaviotron/zimbra/blob/master/README.en.md)

Esta documentación es una guía completa de cómo instalar Zimbra (versión LTS 8.8.15) en un cluster corosync/pacemaker de máquinas virtuales en Proxmox. Incluye los pasos necesarios para permitir auto-provisión de cuentas usando un árbol LDAP externo e instrucciones para configurar Fencing (STONITH).

## Prefacio

La versión comunitaria de Zimbra (ZCS) no provee funciones para trabajar en cluster. No existe documentación oficial para que los mensajes almacenados (zimlet mailboxd) tenga replicación y alta disponibilidad. La documentación solo trae instrucciones para hacer una configuración multi-maestro del servicio LDAP lo cual es insuficiente para un ambiente seguro de producción.

La solución propuesta en esta documentación consiste en configurar un cluster de 3 nodos donde el servicio Zimbra sea un recurso de pacemaker. Por suerte todos los archivos necesarios para que funcione Zimbra están en /opt/zimbra de manera que creando en esa ruta un punto de montaje a un recurso de almacenamiento compartido entre los nodos y asegurando que sólo uno de estos pueda leer y escribir (evitando una situación de split-brain) se hace el truco de clusterizar Zimbra.

## Diseño de Arquitectura

El panorama completo de la solución propuesta en este documento se puede obaservar en la siguiente ilustración:

![diagrama](imgs/Diagrama2.png)

Las siguientes son las direcciones IP y nombres de hosts que usaremos como ejemplo:

```
mbox01.domain.tld  192.168.0.1
mbox02.domain.tld  192.168.0.2
mbox03.domain.tld  192.168.0.3
mbox.domain.tld    192.168.0.4  <--- IP virtual del Cluster de mailboxd
proxy01.domin.tld  192.168.0.5
proxy02.domin.tld  192.168.0.6
proxy03.domin.tld  192.168.0.7
mail.domian.tld    192.168.0.8  <--- IP virtual en roud-robin para los proxies

proxmox.domain.tld 192.168.0.10 <--- Hypervisor Proxmox

```

## Preparación del Sistema Operativo donde corre Proxmox

Es necesaria una unidad de almacenamiento SAN o NAS añadida como un dispositivo en cada máquiona virtual del cluster. Si se usa otro recurso (disco RAID compartido, montaje NFS, etc) es posible que la interfaz web de Proxmox no permita añadirlo a mas de una máquina virtual. en ese caso para hacerlo manualmente se pueden ejecutar las siguientes instrucciones:

```
cd /etc/pve/qemu-server/
qm set 101 -ide1 /dev/sdX
```

Notese que hay que cambiar "**101**" por el ID de la máquina virtual que aplique en su caso y **/dev/sdX** por el dispositivo de almacenamiento que se ve desde el sistema operativo donde corre Proxmox.

**NOTA:** es necesario que esté habilitado este recurso de almacenamiento **ANTES** de proceder a hacer la instalación del cluster. Se debe asegurar ANTES que las máquinas virtuales al levantar vean el dispositivo para poder proseguir con los siguientes pasos.

## Preparación del Sistema Operativo de los nodos del cluster

El proceso siguiente debe realizarse simultáneamente en todos los nodos del cluster. Es muy importante que el número de nodos sea exactamente 3 para que las instrucciones de esta documentación garanticen (completamente) que en su entorno resultante no habrá una situación de split-brain. Muchas recetas en internet (la mayoría de ellas) muestran como montar un cluster con sólo 2 nodos (inhabilitando el quorum) y eso tarde o temprano se traducirá en la corrupción de la data.

Se usará en este ejemplo CentOS 7 recién instalado en cada nodo, sólo con los paquetes básicos y escenciales.

En cada uno de los nodos se deben ejecutar los siguientes comandos, para añadir repositorios adicionales y actualizar el listado de paqquetes disponibles para ser instalados:

```
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm
yum -y update
yum -y upgrade
```

En cada uno de los nodos, se instalan todos los paquetes necesarios:

```
yum -y install ipa-client unzip net-tools sysstat openssh-clients \
    perl-core libaio nmap-ncat libstdc++.so.6 wget vim \
    pacemaker pcs corosync resource-agents pacemaker-cli fence-agents-all \
    git gcc make automake autoconf libtool pexpect python-requests
```

Es muy importante que cada nodo tenga asignado un FQDN. En el nodo "mbox01":

```
hostnamectl set-hostname "mbox01.domain.tld" && exec bash 
```
(por supuesto, debe sustituir "domain.tld" por el nombre de su dominio)

En "mbox02":

```
hostnamectl set-hostname "mbox02.domain.tld" && exec bash 
```

En "mbox03":

```
hostnamectl set-hostname "mbox03.domain.tld" && exec bash 
```

Seguidamente, se deben colocar las direcciones de los demás componentes del sistema de correo, de manera que ante un fallo del DNS continúen funcionando los servicios y el cluster. Esto debe estar en **/etc/hosts** en cada uno de los nodos:

```
192.168.0.1    mbox01.domain.tld     mbox01
192.168.0.2    mbox02.domain.tld     mbox02
192.168.0.3    mbox03.domain.tld     mbox03
192.168.0.4    mbox.domain.tld       mbox
192.168.0.5    proxy01.domin.tld     proxy01
192.168.0.6    proxy02.domin.tld     proxy02
192.168.0.7    proxy03.domin.tld     proxy03
192.168.0.8    mail.domian.tld       mail
192.168.0.10   proxmox.domain.tld    proxmox
```

Es necesario, en cada uno de los nodos, deshabilitar las políticas SELinux que trae CentOS por defecto:
```
setenforce 0
```

Para que SELinux quede en ese estado en los próximos inicios del sistema operativo, es necesario cambiar la siguiente línea del archivo **/etc/selinux/config** en cada uno de los nodos:
```
SELINUX=permissive
```

En cada uno de los nodos, el cortafuegos necesita tener habilitados los puertos necesarios:

```
firewall-cmd --permanent --add-port={25,80,110,143,389,443,465,587,993,995,5222,5223,9071,7071}/tcp
firewall-cmd --reload
```

Dependiendo de cada caso, es posible que sea necesario habilitar otros puertos adicionales o el proceso puede fallar, se puede por tanto inahbilitar temporalmente el cortafuegos de la siguiente manera en cada nodo:

```
systemctl stop firewalld
systemctl disable firewalld
```

Al terminar todos los pasos de esta documentación, es importante asegurar cuáles puertos nos realmente necesarios tener abiertos y volver a levantar el servicio:

```
systemctl start firewalld
systemctl enable firewalld
```

Es necesario que no haya ningún otro servicio activo que use el puerto 25/tcp (SNMP) y CentOS por defecto provee un servicio postfix para el manejo de la mensajería interna del sistema operativo, así que es necesario detenerlo e inhabilitarlo en cada uno de los nodos:
```
systemctl stop postfix
systemctl disable postfix
```

Todos los clientes deben encontrar la dirección IP de los servicios, por lo tanto debe haber un registro en el DNS de cada uno de los nodos del cluster. Esto debe hacerse manualmente para cada caso.

Si se usa FreeIPA para lograr esta resolución de nombres, en cada nodo debe realizarse el enroll mediante el siguiente comando:
```
ipa-client-install --enable-dns-updates
```

Para incorporar cada nodo como miembro del cluster se debe colocar en cada uno la misma contraseña para el usuario "hacluster":

```
echo "hacluster:tu_password"|chpasswd
```
Se debe cambiar "tu_password" por alguna contraseña arbitraria la cual debe ser la misma en todos los nodos.


Seguidamente en cada nodo se levanta el cluster:
```
systemctl start pcsd
systemctl status pcsd
```

El sistema de cluster en CentOS presenta un mal funcionamiento en algunos casos, sobre todo cuando corre en hardware con procesadores muy veloces, por lo cual es necesario retrasar levemente su inicio de manera de asegurar que a otros servicios les de tiempo de levantar antes de ser consiltados por corosync. Esto se logra modificando el archivo **/usr/lib/systemd/system/corosync.service**, añadiendo la directiva "**ExecStartPre=/usr/bin/sleep 3**" en la sección "**[service]**" del script en systemd. Esta sección del archivo debe quedar así entonces en todos los nodos:

```
[Service]
ExecStartPre=/usr/bin/sleep 3
ExecStart=/usr/share/corosync/corosync start
ExecStop=/usr/share/corosync/corosync stop
Type=forking
```

Para que el cambio surja efecto, luego de haber hecho la modificación debe renovarse la configuración que reside en memoria en cada uno de los nodos:

```
systemctl daemon-reload
```

## Creación de los recursos del cluster


En uno solo de los nodos, se autorizan los hosts que componen el cluster:

```
pcs cluster auth mbox01.domain.tld mbox02.domain.tld mbox03.domain.tld
```

La instrucción anterior pedirá un usuario, donde se debe colocar "**hacluster**" y seguidamente se solicitará una contraseña, donde debe colocarse la suministrada en los pasos anteriores (donde se sustituyo "tu_password").


Se debe suministrar un nomnre al cluster ("cluster_zimbra" en este ejemplo), esto se hace en uno solo de los nodos:

```
pcs cluster setup --name cluster_zimbra mbox01.domain.tld mbox02.domain.tld mbox03.domain.tld
```

De la misma manera en uno de los nodos se levanta el cluster:
```
pcs cluster start --all
```

Con las siguientes instrucciones se puede verificar el estado de los componentes que hasta ahora se han configurado en cluster, estos comandos deben dar la misma salida al ejecutarse en cualquiera de los nodos:

```
pcs status cluster
pcs status corosync
```

En el diseño actual no se considera necesario configurar el fencing de los nodos, por lo cual se desactiva STONITH. En una sección posterior de este documento se explica detalladamente cómo habilitarlo en caso que se considere realmente bnecesario:
```
pcs property set stonith-enabled=false
```

## Definición de la IP virtual del cluster

Para crear el recurso de IP virtual, se ejecuta el siguiente comando en uno de los nodos del cluster:
```
pcs resource create virtual_ip ocf:heartbeat:IPaddr2 ip=192.168.0.4 cidr_netmask=32 nic=eth0:0 op monitor interval=30s
```

En este caso se indica "eth0" como la interfaz para crear el alias "et0:0" que tendrá asociada la IP Virtual. Se debe verificar que es "eth0" el nombre del dispositivo de red que usa el Sistema Operativo del nodo.

Se verifica que se ha creado satisfactoriamente este recusro ejecutando el siguiente comando:
```
pcs status resources
```

En la salida obtenida se verá una línea similar a esta, indicando que el recurso ha sido asingado al host mbox01:
```
virtual_ip     (ocf::heartbeat:IPaddr2):       Started mbox01.domain.tls
```


# Creación del recurso en el cluster para el servicio ZIMBRA

Se crea un archivo **/usr/lib/ocf/resource.d/heartbeat/zimbractl** con el siguiente contenido:


```bash
#!/bin/sh
#
# Resource script for Zimbra
# Description:  Manages Zimbra as an OCF resource in
#               an high-availability setup.
# Author:       RRMP <tigerlinux@gmail.com>
# License:      GNU General Public License (GPL)
#
#       usage: $0 {start|stop|reload|monitor|validate-all|meta-data}
#       The "start" arg starts a Zimbra instance
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

: ${OCF_RESKEY_binary="zmcontrol"}
: ${OCF_RESKEY_zimbra_dir="/opt/zimbra"}
: ${OCF_RESKEY_zimbra_user="zimbra"}
: ${OCF_RESKEY_zimbra_group="zimbra"}
USAGE="Usage: $0 {start|stop|reload|status|monitor|validate-all|meta-data}";

##########################################################################

usage() {
	echo $USAGE >&2
}

meta_data() {
	cat <<END
<?xml version="1.0"?>
<!DOCTYPE resource-agent SYSTEM "ra-api-1.dtd">
<resource-agent name="postfix">
<version>0.1</version>
<longdesc lang="en">
This script manages Zimbra as an OCF resource in a high-availability setup.
</longdesc>
<shortdesc lang="en">Manages a highly available Zimbra mail server instance</shortdesc>

<parameters>

<parameter name="binary" unique="0" required="0">
<longdesc lang="en">
Short name to the Zimbra control script.
For example, "zmcontrol" of "/etc/init.d/zimbra".
</longdesc>
<shortdesc lang="en">
Short name to the Zimbra control script</shortdesc>
<content type="string" default="zmcontrol" />
</parameter>

<parameter name="zimbra_dir" unique="1" required="0">
<longdesc lang="en">
Full path to Zimbra directory.
For example, "/opt/zimbra".
</longdesc>
<shortdesc lang="en">
Full path to Zimbra directory</shortdesc>
<content type="string" default="/opt/zimbra" />
</parameter>

<parameter name="zimbra_user" unique="1" required="0">
<longdesc lang="en">
Zimbra username.
For example, "zimbra".
</longdesc>
<shortdesc lang="en">Zimbra username</shortdesc>
<content type="string" default="zimbra" />
</parameter>

<parameter name="zimbra_group"
 unique="1" required="0">
<longdesc lang="en">
Zimbra group.
For example, "zimbra".
</longdesc>
<shortdesc lang="en">Zimbra group</shortdesc>
<content type="string" default="zimbra" />
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
	echo "Starting Zimbra Services"
	echo "0" > /var/log/db-svc-started.log
	rm -f /var/log/zimbra-svc-stopped.log
	if [ -f /etc/init.d/zimbra ]
	then
		/etc/init.d/zimbra start
	fi
	ocf_log info "Zimbra started."
	exit $OCF_SUCCESS
	;;
stop)
	echo "Stopping Zimbra Services"
	rm -f /var/log/db-svc-started.log
	echo "0" > /var/log/zimbra-svc-stopped.log
	if [ -f /etc/init.d/zimbra ]
	then
		/etc/init.d/zimbra stop
		/bin/killall -9 -u zimbra
	fi
	ocf_log info "Zimbra stopped."
	exit $OCF_SUCCESS
	;;
status|monitor)
	echo "Zimbra Services Status"
	if [ -f /var/log/zimbra-svc-started.log ]
	then
		exit $OCF_SUCCESS
	else
		exit $OCF_NOT_RUNNING
	fi
	;;
restart|reload)
	echo "Zimbra Services Restart"
	ocf_log info "Reloading Zimbra."
	if [ -f /etc/init.d/zimbra ]
	then
		/etc/init.d/zimbra stop
		/bin/killall -9 -u zimbra
		/etc/init.d/zimbra start
	fi
	exit $OCF_SUCCESS
	;;
validate-all)
	echo "Validating Zimbra"
	exit $OCF_SUCCESS
	;;
*)
	usage
	exit $OCF_ERR_UNIMPLEMENTED
	;;
esac
```

Creado este archivo se le otorgan permisos de ejecución:

```
chmod 755 /usr/lib/ocf/resource.d/heartbeat/zimbractl
```

Seguidamente se copia el archivo en los otros nodos:
```
scp /usr/lib/ocf/resource.d/heartbeat/zimbractl root@mbox02.domain.tld:/usr/lib/ocf/resource.d/heartbeat/zimbractl
scp /usr/lib/ocf/resource.d/heartbeat/zimbractl root@mbox03.domain.tld:/usr/lib/ocf/resource.d/heartbeat/zimbractl
```

Luego en cada nodo se le deben otorgar permisos de ejecución:
```
chmod 755 /usr/lib/ocf/resource.d/heartbeat/zimbractl
```

Para añadir el recurso (llamdo "zimbractl" en este ejemplo) se ejecutan las siguientes instrucciones:
```
pcs resource create svczimbra ocf:heartbeat:zimbractl op monitor interval=30s
pcs resource op remove svczimbra monitor
pcs constraint colocation add svczimbra virtual_ip INFINITY
pcs constraint order virtual_ip then svczimbra
```

Se puede comprobar que se ha cargado el recurso mediente el siguiente comando:

```
pcs status
```


# Definición de la ruta con los archivos comunes

La ruta donde residen todos los archivos que necesitan los servicios relacionados con Zimbra se encuentran en **/opt/zimbra**. Se crea este directorio:

```
mkdir -p /opt/zimbra
```

Ese directorio debe crearse como un recurso del cluster, de manera que sólo el nodo activo lo tendrá montado. Esto debe hacerse **sólo en el nodo activo**:

```
cd /
pcs cluster cib add_fs
pcs -f add_fs resource create zimbra_fs Filesystem device="/dev/sdX" directory="/opt/zimbra" fstype="ext4"
pcs -f add_fs constraint colocation add svczimbra zimbra_fs INFINITY
pcs -f add_fs constraint order zimbra_fs then svczimbra
pcs cluster cib-push add_fs
```

NOTA: es necesario cambiar "**/dev/sdX**" para ajustarlo al nombre correcto del dispositivo de almacenamiento.

Finalmente, se concluye la configuración del cluster habilitando los servicios permanentemente:

```
systemctl enable pcsd
systemctl enable corosync
systemctl enable pacemaker
```

## Instalación de Zimbra

Se realizará la instalación de los pauetes de Zimbra Community Suite v8.8.15. Para eso, es necesario crear una entrada temporal en **/etc/hosts** que apunte a "**mail.domain.tld**":

```
127.0.0.1	mail.domain.tld mail 
```

Es importante que en **/opt/zimbra** se encuentre montado (dispositivo de almacenamiento que se compartirá entre los nodos).

En el primer nodo (el que se encuentra activo) se descarga e instala el Software:

```
mkdir /root/zimbra && cd /root/zimbra
wget https://files.zimbra.com/downloads/8.8.15_GA/zcs-8.8.15_GA_3869.RHEL7_64.20190918004220.tgz
tar zxpf zcs-8.8.15_GA_3869.RHEL7_64.20190918004220.tgz
cd zcs-8.8.15_GA_3869.RHEL7_64.20190918004220
./install.sh -s
```

Nóese la opción "**-s**", la cual realizará la instalación sin configurar el sistema, lo cual se realizará posteriormente. El instalador realizará unas preguntas, que deben responderse como se especifican a continuación:

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
  Install zimbra-imapd (BETA - for evaluation only) [N]     <--- ENTER
  Install zimbra-chat [Y]
  The system will be modified.  Continue? [N] Y
```

Despùés de seleccionar la última de esas opciones, se descargarán y actualizarán los paquetes necesarios. En este punto puede ir por un café, porque Zimbra usa Java y todo lo que usa Java tarda mucho tiempo y consume mucho procesador y memoria, hasta para ejecutar los procesos mas simples.

Antes de configurar Zimbra, es necesario corregir la ruta en la cual se encuentran la unidad certificadora (CA):

```
mkdir -p /opt/zimbra/java/jre/lib/security/
ln -s /opt/zimbra/common/etc/java/cacerts /opt/zimbra/java/jre/lib/security/cacerts
chown -R  zimbra.zimbra /opt/zimbra/java
```

Ahora si, se puede correr el programa que configura Zimbra:

```
/opt/zimbra/libexec/zmsetup.pl
```

Puede que aparezca un error como este:
```
DNS ERROR resolving MX for mbox01.domain.tld
It is suggested that the domain name have an MX record configured in DNS
Change domain name? [Yes]
```

Esto quiere decir que no hay un registro MX para el hostname desde el cual se ejecuta el instalador que por defecto lo toma como el dominio de correo.

```
Change domain name? [Yes] <---- ENTER
Create domain: [mbox01.domain.tld] domain.tld <----- Se coloca acá el dominio (sin nombre de host)
```
Cuando termnia este proceso, se muestra un menú en el cual es necesario como primer paso definir una contraseña de administrador de Zimbra. Para hacer eso, se escogen las sigientes opciones cuando en en diálogo aparezca el mensaje **"Address unconfigured (++) items (? - help)"**
- Seleccionar la opción 7: **zimbra-store** en el menú principal
- Seleccionar la opción 4: **Admin Password**

Allí se coloca una contraseña o se toma nota de la que Zimbra aleatoriamente propone como opción. Para regresar al menú principal se presiona ENTER en el diálogo "**Select, or 'r' for previous menu [r]" prompt message to go to main menu**"

Es necesario verificar que el dominio que se creará en el árbl LDAP coincide con el que se definió anteriormente. Si se omite este paso, es posible que los nombres difieran y los buzones de correo sean "**usuario@mbox01.domain.tld**" y no "**usuario@domain.tld**":
- Seleccionar la opción 2: **"zimbra-ldap"** en el menú principal
- Seleccionar la opción 3: **"Domain to create"** y verificar que coincida con el dominio raíz o cambiarlo en caso que nea necesario.

Se regresa al menú principal se presiona ENTER en el diálogo "**Select, or 'r' for previous menu [r]" prompt message to go to main menu**" y desde allí se inicia la instalación sleccionado la opción "a":

```
  Select from menu, or press 'a' to apply config (? - help) a 
```

A continuación se guarda el archivo de configuración. Es importante acá tomar nota del nombre, el cual contiene una extensión aleatoriamente asignada por el instalador
```
  Save configuration data to a file? [Yes]
  Save config in file: [/opt/zimbra/config.21593]
```

Para proceder con la instalación en el Sistema Operativo de Zimbra, se responde "Yes":

```
  The system will be modified - continue? [No] Yes
```

Zimbra se comenzará a a instalar... da tiempo para tomarse otro café, Java lo invita. Al terminar (después de haber tenido tiempo de hablar sobre religión o sobre el movimiento perpetuo en la física), saldrá el siguiente mensaje:

```
  Notify Zimbra of your installation? [Yes]
  Configuration complete - press return to exit
```

Ahora se copia el archivo de configuración a los demás nodos:

```
scp /opt/zimbra/config.21593 mbox02.domain.tld:/root/zmconfig.log
scp /opt/zimbra/config.21593 mbox03.domain.tld:/root/zmconfig.log
```

En este momento se debe borrar la línea temporal asignada a "**mail.domain.tld**" en el archivo **/etc/hosts**

## Instalación de Zimbra en los nodos mbox02 y mbox03

**CUIDADO:** Este procedimiento debe hacerse cuando el nodo esté en modo **OFFLINE** en el cluster, para evitar que la ruta /opt/zimbra esté siendo usada. Para detener el cluster en el nodo se ejecuta:

```
pcs cluster stop mbox02.domain.tld
pcs cluster stop mbox03.domain.tld
```

Para comenzar a instalar, al ejecutar el comando
```
pcs status
```

Se obtendrá como respuesta
```
Error: cluster is not currently running on this node
```

Con el nodo en **offline** hará falta de nuevo colocar "**mail.domain.tld**" en **/etc/hosts** apuntando a **127.0.0.1**.

Se procede a instalar Zimbra:

```
mkdir /opt/zimbra
mkdir /root/zimbra && cd /root/zimbra
wget https://files.zimbra.com/downloads/8.8.15_GA/zcs-8.8.15_GA_3869.RHEL7_64.20190918004220.tgz
tar zxpf zcs-8.8.15_GA_3869.RHEL7_64.20190918004220.tgz
cd zcs-8.8.15_GA_3869.RHEL7_64.20190918004220
./install.sh -s
```

Hay que asegurar que las respuestas siguientes se respondan exactamente igual que en la instalación del primer nodo:
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
  Install zimbra-chat [Y] 
  The system will be modified.  Continue? [N] Y
```

Tiempo para otro café (bueno... quizás no sea muy bueno para la salud... puede funcionar té o limonada en su lugar).

Cuando finalice la instalación:

```
mkdir -p /opt/zimbra/java/jre/lib/security/
ln -s /opt/zimbra/common/etc/java/cacerts /opt/zimbra/java/jre/lib/security/cacerts
chown -R  zimbra.zimbra /opt/zimbra/java
```

En este punto, se realiza la configuración usando el archivo copiado (via SCP) desde el primer nodo:
```
/opt/zimbra/libexec/zmsetup.pl -c /root/zmconfig.log
```

Al finalizar, se detiene Zimbra y se eliminan los archivos de /opt pues se usarán sólo los almacenados en el dispositivo común:
```
/etc/init.d/zimbra stop
killall -9 -u zimbra
mv /opt/zimbra /root/old-zimbra
mkdir /opt/zimbra
```

Se elimina la línea temporal en **/etc/hosts** que apunta a "**mail.domain.tld**".

Ahora, se levanta el cluster en este nodo:
```
pcs cluster start mbox02.domain.tld
pcs cluster start mbox03.domain.tld
```

Llegado a este punto, Zimbra trabajará en cluster activo-pasivo. Se puede hacer la prueba accediendo desde un navegador a **https://mail.domain.tld**, deteniendo (o apagando) el nodo que esté activo hará que el cluster pase todos los servicios a otro nodo. Se puede observar el proceso (que demora unos 2 minutos) de la siguiente forma:
```
watch pcs status
```

(CONTOL + C para salir)

# Auto-Provisión de cuentas vía LDAP (Opcional)

Si se desea usar un Arbol LDAP externo para la autenticación de cuentas de usuario, se deben seguir los siguientes pasos. En este ejemplo se usa la estructura de un servidor FreeIPA, pero para cualquier otro caso es necesario tener los mismos datos:

- La URL del servicio LDAP, en este ejemplo **ldap://freeipa.domain.tld:389**
- La base de búsqueda (LDAP Search Base) donde pueden ser encontradas las cuentas, por ejemplo **cn=accounts,dc=domain,dc=tld**
- El filtro LDAP con el cual se obtiene una cuenta. Es importante que este filtro arroje siempre un solo resultado: **(uid=%u)**

Para configurar la auto-provisión se abre la interfaz de administración de Zimbra **https://mail.domain.tld:7071** y se siguen los siguientes pasos:
- Ir a **Admin > Configuration > Domain** y seleccionar el dominio, en nuestro caso **domain.tld**
- Ir a **Authentication** en el menú de la izquierda y luego presionar el **ícono en forma de engranaje** en la esquina superior derecha de la ventana y allí seleccionar **Autentication**
- En la ventana emergente que se abrirá se seleeciona la opción **Use external LDAP** y se da click en **siguiente**
- Colocar el nombre de hos o la IP del servicio LDAP: **freeipa.domain.tld**
- Colocar **(uid=%u)** en el campo donde pide el filtro LDAP
- Colocar **cn=users,cn=accounts,dc=domain,dc=tld** en el campo "Base DN" y seleccionar **siguiente**
- Probar los valores suministrados usando el botón de pruebas, donde habrá que suministrar un usuario y contraeeñas válidos en el LDAP
- Presionar **Finnish** en el diálogo de configuración

Ahora desde una consola como root en el nodo activo del cluster se ejecuta la siguiente serie de comandos:

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
zmprov md prue.ba zimbraAutoProvLdapURL "ldap://freeipa.domain.tld:389"
zmprov md prue.ba zimbraAutoProvMode LAZY
zmprov md prue.ba zimbraAutoProvNotificationBody "Your account has been auto provisioned.  Your email address is ${ACCOUNT_ADDRESS}."
zmprov md prue.ba zimbraAutoProvNotificationFromAddress prov-admin@prue.ba
zmprov md prue.ba zimbraAutoProvNotificationSubject "New account auto provisioned"
```
Como es lógico, hay que adaptar los campos "cn=accounts,dc=domain,dc=tld", (uid=%u)" y "ldap://freeipa.domain.tld:389" siministrados acá como ejemplo para adaptarlo a las variantes del LDAP externo que se esté usando.

Con eso se ha configurado la auto-provisión externa. Ahora se puede abrir el buzón de cualquier usuario válido registrado en el Arbol LDAP externo.

## Configuración de Fencing (opcional)

EL "fencing" es usado para aislar un nodo que el cluster determina que no está en condiciones de poder ser integrado y es posible, opcionalmente, configurar un método llamado STONITH para este fin.

Sólo realice esta configuración si considera que es necesario desahacerse de la presencia del sistema operativo de un nodo, en caso que no cumpla con el quorum requerido:

En este ejemplo, el Sistema Operativo donde corren las máquinas virtuales es el GNU/Linux de Proxmox, Se instalan en ese equipo los agentes del fencing:
```
apt install fence-agents
```

CentOS no incluye este agente, por lo cual es necesario descargar sus fuentes y compilarlas (en cada uno de los nodos del cluster):

```
cd
git clone https://github.com/ClusterLabs/fence-agents.git
cd fence-agents/
./autogen.sh
./configure --with-agents=pve
make && make install
fence_pve --version
```

Para probar el funcionamiento del agente, desde los nodos del cluster se ejecuta el siguiente comando:
```
/usr/sbin/fence_pve --ip=<proxmox_ip> --username=root@pam --password=<proxmox_passwd> --plug=<vm_id> --action=status
```

En este ejemplo, hay que cambiar <proxmox_ip> por la dirección IP del Hypervisor KVM, "root@pam" es necesario dejarlo sin modificaciones, en <proxmox_passowrd> se coloca la contraseña de root de Hypervisor y en <vm_id> el número de identificación de la máquina virtual, por ejemplo:
```
/usr/sbin/fence_pve --ip=192.168.0.10 --username=root@pam --password="EstaEsMiClave" --plug=101 --action=status
```

Se obtendrá un mensaje "STATUS: OK" si todo funciona correctamente.

Cuando un nodo falla (se cuelga, pierde conexión, etc) pacemaker procederá a aislarlo (fencing). En el siguiente ejemplo están las reglas para activar STONITH en ese caso. En cualquier nodo activo se ejecuta:

```
pcs stonith create fence_mboxs01 fence_pve ipaddr=<proxmox_ip> inet4_only="true" vmtype="qemu" \
  login="root@pam" passwd=<proxmox_passwd> delay="15" port=<vm_id> pcmk_host_check=static-list \
  pcmk_host_list="mbox01.domain.tld" node_name="pve"

pcs stonith create fence_mbox02 fence_pve ipaddr=<proxmox_ip> inet4_only="true" vmtype="qemu" \
  login="root@pam" passwd=<proxmox_passwd> delay="15" port=<vm_id> pcmk_host_check=static-list \
  pcmk_host_list="mbox02.domain.tld" node_name="pve"

pcs stonith create fence_mbox03 fence_pve ipaddr=<proxmox_ip> inet4_only="true" vmtype="qemu" \
  login="root@pam" passwd=<proxmox_passwd> delay="15" port=<vm_id> pcmk_host_check=static-list \
  pcmk_host_list="mbox03.domain.tld" node_name="pve"
```

Para que un nodo permanezca en línea, debe ver al menos un nodo mas. Con el siguiente comando se activa esa directiva:
```
pcs stonith update fence_mbox01 action="off" --force
pcs stonith update fence_mbox02 action="off" --force
pcs stonith update fence_mbox03 action="off" --force
pcs property set stonith-enabled=true
pcs property set no-quorum-policy=suicide
```

Finalmente, se reinicia los servicios de cluster para que la configuración tome efecto:
```
systemctl enable pcsd
systemctl enable corosync
systemctl enable pacemaker
```

Ahora se puede probar el fencing, suspendiendo la configuración de red en una máquina virtual (lo cual desconectará el nodo y perderá el quorum):
```
systemctl stop networking
```

Se puede verificar así:

```
watch pcs status
```
(CONTROL-C para salir)


## Instalación de Servidores Zimbra-Proxy (opcional)

Primero, hay que configurar el nombre de la máquina para que sea un FQDN:
```
hostnamectl set-hostname "proxy01.domain.tld" && exec bash 
```
Luego, verificar que exista el proxy en **/etc/hosts** al igual que los demás nodos:

```
192.168.0.1    mbox01.domain.tld     mbox01
192.168.0.2    mbox02.domain.tld     mbox02
192.168.0.3    mbox03.domain.tld     mbox03
192.168.0.4    mbox.domain.tld       mbox
192.168.0.5    proxy01.domin.tld     proxy01
192.168.0.6    proxy02.domin.tld     proxy02
192.168.0.7    proxy03.domin.tld     proxy03
192.168.0.8    mail.domian.tld       mail
192.168.0.10   proxmox.domain.tld    proxmox
```

Seguidamente hay que añadir el proxy en el DNS. Si se está usando FreeIPA:
```
ipa-client-install --enable-dns-updates
```

Desactivar la política SELinux:
```
setenforce 0
```

Para que SELinux se desactive permanentemente, cambiar la siguiente línea en **/etc/selinux/config**:
```
SELINUX=permissive
```

Luego, correr el instalador de Zimbra con la opción "**-s**"

```
mkdir /root/zimbra && cd /root/zimbra
wget https://files.zimbra.com/downloads/8.8.15_GA/zcs-8.8.15_GA_3869.RHEL7_64.20190918004220.tgz
tar zxpf zcs-8.8.15_GA_3869.RHEL7_64.20190918004220.tgz
cd zcs-8.8.15_GA_3869.RHEL7_64.20190918004220
./install.sh -s

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
Install zimbra-memcached [Y] N
Install zimbra-proxy [Y] Y <----------------- "y" sólo responder Y en esta opción

The system will be modified.  Continue? [N] Y
```

En muchas recetas en internet, "zimbra-memcached" es instalada junto a "zimbra-proxy", pero la verdad es que sólo una instancia de "zimbra-memcached" es necesaria y en el servicio principal ya se ha instalado uno.

```
mkdir -p /opt/zimbra/java/jre/lib/security/
ln -s /opt/zimbra/common/etc/java/cacerts /opt/zimbra/java/jre/lib/security/cacerts
chown -R  zimbra.zimbra /opt/zimbra/java
/opt/zimbra/libexec/zmsetup.pl
```

Es necesario conocer la clave de acceso para Nginx. Esta clave fue generada automáticamente en el proceso de instalación de los nodos, así que será necesario conocerla. Desde el nodo activo del cluster se ejecuta:
```
su - zimbra
zmlocalconfig -s ldap_nginx_password
```

Ahora, en el instalador del proxy, se accede a la opción 1 "**Common Configuration**" y allí se va a la opción 2 "**Ldap master host**". Allí se coloca el nombre de host asignado a la IP Virtual: "**mail.domain.tld**"

Posteriormente, se accede a la opción 4 "**Ldap Admin password**" y se proporciona la clave obtenida el el proceso anterior. Hecho esto el instalador se conectará y tomará del servicio LDAP principal todas las configuraciones necesarias:

```
Setting defaults from ldap...done.
```

Para regresar al menú principal se presiona ENTER ante el mensaje **"Select, or 'r' for previous menu [r]"** y allí se selecciona la opción 2 **"zimbra-proxy"** en la cual se provee la contraseña de nginx a través de la opción 12 **"Bind password for nginx ldap user"**. Se coloca la misma obtenida en el paso anterior.

Una vez que se ha realizado este paso, se regresa al menú principal y se procede a realizar la instalación:

```
*** CONFIGURATION COMPLETE - press 'a' to apply
Select from menu, or press 'a' to apply config (? - help) a  <------ press "a" and ENTER here
Save configuration data to a file? [Yes]
Save config in file: [/opt/zimbra/config.15941] 
Saving config in /opt/zimbra/config.15941...done.
The system will be modified - continue? [No] Yes <---- "Yes" and ENTER
...
Notify Zimbra of your installation? [Yes]
...
Configuration complete - press return to exit 
```

Para finalizar, es necesario actualizar las llaves SSH entre los servidores. En el nodo activo y en el nuevo proxy instalado se ejecutan los siguientes comandos:

```
su - zimbra
/opt/zimbra/bin/zmsshkeygen
/opt/zimbra/bin/zmupdateauthkeys
exit;
```

### Configuración de RSYSLOG

Para obtener las estadísticas y estado de funcionamiento de los servicios, es necesario realizar la configuración de RSYSLOG. 

En cada uno de los proxies se ejecuta:
```
/opt/zimbra/libexec/zmsyslogsetup
```

## Reparación de chat-zimlet:

El componente de Zimbra encargado del servicio de mensajería instantánea (chat) presenta fallas al terminar de ser instalado y no funciona correctamente. El error consiste en una librería Java mal empaquetada. El problema se resuelve sustituyando el paquete defectuoso de la siguiente manera:

```
mv /opt/zimbra/lib/ext/openchat/zal.jar /tmp
cp -rp /opt/zimbra/lib/ext/zimbradrive/zal.jar /opt/zimbra/lib/ext/openchat/zal.jar
su - zimbra
zmmailboxdctl restart
```

## Documentación consultada para realizar el presente trabajo:

- Zimbra Cluster: https://github.com/tigerlinux/tigerlinux-extra-recipes/tree/master/recipes/ispapps/zimbra-cluster-centos7
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


