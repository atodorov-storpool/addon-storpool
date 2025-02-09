## Deploy tweaks

### Setup

#### Installation

The scripts are installed with the default installation of addon-storpool


#### Configuration

The deploy-tweaks script is activated by replacing the upstream deploy script with a local one executed on the front-end node.

```
VM_MAD = [
  ARGUMENTS = "... -l deploy=deploy-tweaks",
]
```


### Usage

The deploy-tweaks script is called on the active front-end node. The script call all found executable helpers in the `deploy-tweaks.d` folder one by one passing two files as arguments - a copy of the VM domain XML file and the OpenNebula's VM definition in XML format. The helper scripts overwrite the VM domain XML copy and on successful return code the VM domain XML is updated from the copy (and passed to the next helper).

Once all helper scripts are processed the native _vmm/kvm/deploy_ script on the destination KVM host id called with the tweaked domain XML for VM deployment.

#### context-cache.py

Replaces the `devices/disk/driver` attributes _cache=none_ and _io=native_ on all found cdrom disks that had _type=file_ and _device=cdrom_

#### cpu.py

* `domain/vcpu element` if missing (with default value '1')
* definition for VCPU topology using defined with `USER_TEMPLATE/T_CPU_THREADS`, `USER_TEMPLATE/T_CPU_SOCKETS`
* definition for `domain/cpu/feature` using defined variable `USER_TEMPLATE/T_CPU_FEATURES`
* definition for `domain/cpu/model` using defined variable `USER_TEMPLATE/T_CPU_MODEL`
* definition for `domain/cpu/vendor` using defined variable `USER_TEMPLATE/T_CPU_VENDOR`
* definition for `domain/cpu`attribute `check` using defined variable `USER_TEMPLATE/T_CPU_CHECK`
* definition for `domain/cpu`attribute `match` using defined variable `USER_TEMPLATE/T_CPU_MATCH`
* definition for `domain/cpu`attribute `mode` using defined variable `USER_TEMPLATE/T_CPU_MODE`
The script create/tweak the _cpu_ element of the domain XML.

> Default values could be exported via the `kvmrc` file.


##### T_CPU_SOCKETS and T_CPU_THREADS

Set the number of sockets and VCPU threads of the CPU topology.
For example the following configuration represent VM with 8 VCPUs, in single socket with 2 threads:

```
VCPU = 8
T_CPU_SOCKETS = 1
T_CPU_THREADS = 2
```

```xml
  <cpu mode='host-passthrough' check='none'>
    <topology sockets='1' cores='4' threads='2'/>
    <numa>
      <cell id='0' cpus='0-7' memory='33554432' unit='KiB'/>
    </numa>
  </cpu>
```

##### Advanced

The following variables were made available for use in some corner cases.

> For advanced use only! A special care should be taken when mixing the variables as some of the options are not compatible when combined.

###### T_CPU_MODE

Possible options: _custom_, _host-model_, _host-passthrough_.

Special keyword _delete_ will instruct the helper to delete the element from the domain XML.

###### T_CPU_FEATURES

Comma separated list of supported features. The policy could be added using a colon (':') as separator.

###### T_CPU_MODEL

The optional _fallback_ attribute could be set after the model, separated with a colon (':').
The special keyword _delete_ will instruct the helper to delete the element.

###### T_CPU_VENDOR

Could be set only when a `model` is defined.

###### T_CPU_CHECK

Possible options: _none_, _partial_, _full_.
The special keyword _delete_ will instruct the helper to delete the element.

###### T_CPU_MATCH

Possible options: _minimum_, _exact_, _strict_.
The special keyword _delete_ will instruct the helper to delete the element.


#### cpuShares.py

The script will reconfigure _/domain/cputune/shares_ using the folloing formula

`( VCPU * multiplier ) * 1024`

Default for the _multiplier_ is _0.1_ and to change per VM set _USER_TEMPLATE/T_CPU_MUL_ variable

##### T_CPUTUNE_MUL

Override VCPU _multiplier_ for the given VM

`domain/cputune/shares` = VCPU * `USER_TEMPLATE/T_CPUTUNE_MUL` * 1024

##### T_CPUTUNE_SHARES

To override the above calculations and use the value for _/domain/cputune/shares/_ for the given VM.

`domain/cputune/shares` =`USER_TEMPLATE/T_CPUTUNE_SHARES`

#### disk-cache-io.py

Set disk `cache=none` and `io=native` for all StorPool backed disks

#### feature_smm.py

See [UEFI boot](uefi_boot.md)

#### hv-enlightenments.py

Set HYPERV enlightenments features at `domain/features/hyperv` when defined `T_HV_SPINLOCKS`,T_HV_RELAXED`,`T_HV_VAPIC`,`T_HV_TIME`,`T_HV_CRASH`,`T_HV_RESET`,`T_HV_VPINDEX`,`T_HV_RUNTIME`,`T_HV_SYNC`,`T_HV_STIMER`,`T_HV_FEQUENCIES` (`T_HV_REENLIGHTENMENT`,`T_HV_TLBFLUSH`,`T_HV_IPI` and `T_HV_EVMCS`) with value _on_ or _1_ 

#### hyperv-clock.py

The script add additional tunes to the _clock_ entry when the _hyperv_ feature is enabled in the domain XML.

> Sunstone -> Templates -> VMs -> Update -> OS Booting -> Features -> HYPERV = YES

```xml
<domain>
  <clock offset="utc">
    <timer name="hypervclock" present="yes"/>
    <timer name="rtc" tickpolicy="catchup"/>
    <timer name="pit" tickpolicy="delay"/>
    <timer name="hpet" present="no"/>
  </clock>
</domain>
```

Note: Same could be done by editing `HYPERV_OPTIONS` variable in `/etc/one/vmm_exec/vmm_exec_kvm.conf`

#### iothreads.py

The script will define an [iothread](https://libvirt.org/formatdomain.html#elementsIOThreadsAllocation) and assign it to all virtio-blk disks and vitio-scsi controllers.

```xml
<domain>
  <iothreads>1</iothreads>
  <devices>
    <disk device="disk" type="block">
      <source dev="/var/lib/one//datastores/0/42/disk.2" />
      <target dev="vda" />
      <driver cache="none" discard="unmap" io="native" iothread="1" name="qemu" type="raw" />
    </disk>
    <controller index="0" model="virtio-scsi" type="scsi">
      <driver iothread="1" />
    </controller>
  </devices>
</domain>
```

With OpenNebula 6.0+ use `USER_TEMPLATE/T_IOTHREADS_OVERRIDE` to override the default values)

#### kvm-hide.py

Set `domain/features/kvm/hidden[@state=on]` when `USER_TEMPLATE/T_KVM_HIDE` is defined

#### os.py

Set/update `domain/os` element

see [UEFI boot](uefi_boot.md)

#### q35-pcie-root.py

Update PCIe root ports when machine type `q35` is defined with the number defined in `USER_TEMPLATE/Q35_PCIE_ROOT_PORTS`

#### resizeCpuMem.py

See [Live resize CPU and Memory](resizeCpuMem.md)

#### scsi-queues.py

Set the number of `nqueues` for virtio-scsi controllers to match the number of VCPUs. This code is triggered when there are alredy defined number of quieues for a given VM.

```xml
<domain>
  <vcpu>4</vcpu>
  <devices>
    <controller index="0" model="virtio-scsi" type="scsi">
      <driver queues="4" />
    </controller>
  </devices>
</domain>
```

#### uuid.py

Override/define `domain/uuid` with the value from `USER_TEMPLATE/T_UUID`

Note: With OpenNebula 6.0+ same could be aciheved with `OS/DEPLOY_ID`.

#### vfhostdev2interface.py

OpenNebula provides generic PCI passthrough definition using _hostdev_. But
when it is used for NIC VF passthrough the VM's kernel assign random MAC
address (on each boot). Replacing the definition with _interface_ it
is possible to define the MAC addresses via configuration variable in
_USER_TEMPLATE_. The T_VF_MACS configuration variable accept comma separated
list of MAC addresses to be asigned to the VF pass-through NICs. For example to
set MACs on two PCI passthrough VF interfaces, define the addresses in the
T_VF_MACS variable and (re)start the VM:

```
T_VF_MACS=02:00:11:ab:cd:01,02:00:11:ab:cd:02
```

The following section of the domain XML

```xml
  <hostdev mode='subsystem' type='pci' managed='yes'>
    <source>
      <address  domain='0x0000' bus='0xd8' slot='0x00' function='0x5'/>
    </source>
    <address type='pci' domain='0x0000' bus='0x01' slot='0x01' function='0'/>
  </hostdev>
  <hostdev mode='subsystem' type='pci' managed='yes'>
    <source>
      <address  domain='0x0000' bus='0xd8' slot='0x00' function='0x6'/>
    </source>
    <address type='pci' domain='0x0000' bus='0x01' slot='0x02' function='0'/>
  </hostdev>
```

will be converted to

```xml
  <interface managed="yes" type="hostdev">
    <driver name="vfio" />
    <mac address="02:00:11:ab:cd:01" />
    <source>
      <address bus="0xd8" domain="0x0000" function="0x5" slot="0x00" type="pci" />
    </source>
    <address bus="0x01" domain="0x0000" function="0" slot="0x01" type="pci" />
  </interface>
  <interface managed="yes" type="hostdev">
    <driver name="vfio" />
    <mac address="02:00:11:ab:cd:02" />
    <source>
      <address bus="0xd8" domain="0x0000" function="0x6" slot="0x00" type="pci" />
    </source>
    <address bus="0x01" domain="0x0000" function="0" slot="0x02" type="pci" />
  </interface>
```

#### video.py

Redefine `domain/devices/video` and using `USER_TEMPLATE/T_VIDEO_MODEL` with space separated list of attributes (_name=value_)

#### volatile2dev.py

The script will reconfigure the volatile disks from file to device when the VM disk's TM_MAD is _storpool_.

```xml
    <disk type='file' device='disk'>
        <source file='/var/lib/one//datastores/0/4/disk.2'/>
        <target dev='sdb'/>
        <driver name='qemu' type='raw' cache='none' io='native' discard='unmap'/>
        <address type='drive' controller='0' bus='0' target='1' unit='0'/>
    </disk>
```
to

```xml
    <disk type="block" device="disk">
        <source dev="/var/lib/one//datastores/0/4/disk.2" />
        <target dev="sdb" />
        <driver name="qemu" cache="none" type="raw" io="native" discard="unmap" />
        <address type="drive" controller="0" bus="0" target="1" unit="0" />
    </disk>
```


To do the changes when hot-attaching a volatile disk the original attach_disk script (`/var/lib/one/remotes/vmm/kvm/attach_disk`) should be overrided too:

```
VM_MAD = [
    ...
    ARGUMENTS = "... -l deploy=deploy-tweaks,attach_disk=attach_disk.storpool",
    ...
]
```

