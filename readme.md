# Foreman ESX Install Scripts

These templates provision ESXi (tested on 5.5 & 6) with Foreman. While the VMWare kernel does not support Puppet indirect management via puppet is available using https://forge.puppet.com/vmware/vcenter.

## Building an Install Repo

Extract your install ISO to a friendly web server, this will be your OS media path in Foreman. If you are using a case sensitive OS for hosting, you will need to rename all the files & folders to lowercase as VMWare have used lowercase in config files;
```sh
find -depth -exec rename 's/(.*)\/([^\/]*)/$1\/\L$2/' {} \;
```

The boot.cfg also requires some minor amends as it is set to look in the root of the server for files;
```sh
sed -i "s/\///g" boot.cfg
```

Finally increase the timeout for loading files;
```sh
echo "timeout=5" >> boot.cfg
```

## iPXE
You will need to make two changes to iPXE in order for it to support booting the ESXi installer, and then make it;
* src/arch/i386/image/multiboot.c change "MAX_MODULES 8" to "MAX_MODULES 100"
* src/config/local/general.h add "#define IMAGE_COMBOOT"

This can be scripted as;
```sh
git clone git://git.ipxe.org/ipxe.git
sed -i "s/MAX_MODULES 8/MAX_MODULES 100/g" ipxe/src/arch/i386/image/multiboot.c
echo "#define IMAGE_COMBOOT" >> ipxe/src/config/local/general.h
cd ipxe/src
make bin/undionly.kpxe
cp bin/undionly.kpxe /var/lib/tftpboot
```

## Extending Kickstart

Post-setup steps such as joining to a cluster, configuring storage etc can be added to the end of the provision template. http://www.virtuallyghetto.com is a really good resource for finding these scripts.

### Example: Licensing a system

```sh
<%if !@host.params['license_key'].empty?%>
    # assign license
    vim-cmd vimsvc/license --set <%=@host.params['license_key']%>
<%end%>
```

### Example: Enabling iSCSI and setting the WWN

```sh
esxcli iscsi software set -e true
ADAPTER=`esxcli iscsi adapter list | grep Software | awk '{print $1;}'`
HOSTNAME=esx01-automation
esxcli iscsi adapter set -A $ADAPTER --name iqn.1998-01.com.vmware:$HOSTNAME

esxcli iscsi networkportal add -A $ADAPTER -n vmk1
esxcli iscsi networkportal add -A $ADAPTER -n vmk2
```

### Example: Configuring networks

```sh
esxcli network vswitch standard add -v vSwitch1
esxcli network vswitch standard portgroup add -v vSwitch1 -p iscsi1
esxcli network vswitch standard portgroup add -v vSwitch1 -p iscsi2
esxcli network vswitch standard uplink add -v vSwitch1 -u vmnic1
esxcli network vswitch standard uplink add -v vSwitch1 -u vmnic2
esxcli network vswitch standard portgroup add -v vSwitch0 -p vMotion
esxcli network vswitch standard uplink add -v vSwitch0 -u vmnic3
esxcli network ip interface add -i vmk1 -p iscsi1
esxcli network ip interface add -i vmk2 -p iscsi2
esxcli network ip interface ipv4 set -i vmk1 -t dhcp
esxcli network ip interface ipv4 set -i vmk2 -t dhcp
esxcli network ip interface add -i vmk3 -p vMotion
esxcli network ip interface ipv4 set -i vmk3 -t dhcp
vim-cmd hostsvc/vmotion/vnic_set vmk3
esxcli network vswitch standard set -v vSwitch0 -m 9000
esxcli network vswitch standard set -v vSwitch1 -m 9000
esxcli network ip interface set -i vmk0 -m 9000
esxcli network ip interface set -i vmk1 -m 9000
esxcli network ip interface set -i vmk2 -m 9000
esxcli network ip interface set -i vmk3 -m 9000
esxcli network vswitch standard portgroup policy failover set -p iscsi1 -a vmnic2
esxcli network vswitch standard portgroup policy failover set -p iscsi2 -a vmnic1
esxcli network vswitch standard portgroup policy failover set -p iscsi1 -a vmnic1
esxcli network vswitch standard portgroup policy failover set -p iscsi2 -a vmnic2
```