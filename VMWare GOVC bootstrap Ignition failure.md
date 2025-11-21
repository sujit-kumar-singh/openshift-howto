### Govc ignition failure for bootstrap node when ignition file is large > 128KB (Linux) in size

The node ignition might fail if the ignition file size is more than 128KB on Linux platform. This might be seen as Ignition not provided when the system is booted up for the first time.

#### Ref: https://docs.fedoraproject.org/en-US/fedora-coreos/provisioning-vmware/

Fix

```bash

export GOVC_URL='10.10.11.112'
export GOVC_USERNAME='XXXXXXXXXXXXXXXXXXXXXXXXXX'
export GOVC_PASSWORD='XXXXXXXXXXXXXXXX'
export GOVC_INSECURE=1
export cluster='/datacenter/host/cluster01'
export datacenter='datacenter'
export template='/datacenter/vm/test-ocp-govc/coreos-ova' # CoreOS template
export esx_host='/datacenter/host/cluster01/raptor.ucmcswg.com'
export datastore='/datacenter/datastore/datastore1-1'
export vm_folder='/datacenter/vm/test-ocp-govc'
export vm_net_name='/datacenter/network/VM Network'
export resources='/datacenter/host/cluster01/Resources'
```

Create the virtual machine by cloning from an OVA template.

```bash
govc vm.clone -on=false -ds=$datastore -dc $datacenter -vm $template -folder $vm_folder BOOTSTRAP
govc vm.disk.change -dc $datacenter -size=120G -disk.key=0 -vm=BOOTSTRAP
govc vm.change -dc $datacenter -vm BOOTSTRAP -cpu-hot-add-enabled=1 -c 6 -m 24768 -memory-hot-add-enabled=1
```

Create gzipped ignition configuration file

```bash
CONFIG_ENCODING="gzip+base64"
CONFIG_FILE="bootstrap.ign"
CONFIG_FILE_ENCODED="${CONFIG_FILE}.gz.b64"
gzip -9c "${CONFIG_FILE}" | base64 -w0 - > "${CONFIG_FILE_ENCODED}"
```

Configure the virtual Machine ignition and other properties

```bash
export IPCFG="ip=10.10.11.156::10.10.11.1:255.255.255.0:::none nameserver=10.10.11.5"
govc vm.change -vm BOOTSTRAP -dc $datacenter -e "guestinfo.afterburn.initrd.network-kargs=${IPCFG}"
govc vm.change -dc $datacenter  -e="disk.enableUUID=1" -dc $datacenter -vm BOOTSTRAP
govc vm.change -dc $datacenter -vm BOOTSTRAP -e "stealclock.enable=TRUE"
govc vm.change -vm BOOTSTRAP -dc $datacenter -e "guestinfo.ignition.config.data.encoding=${CONFIG_ENCODING}"
govc vm.change -vm BOOTSTRAP -dc $datacenter -f "guestinfo.ignition.config.data=${CONFIG_FILE_ENCODED}"
```

Power on the VM

```bash
govc vm.power -dc $datacenter -on BOOTSTRAP
```
