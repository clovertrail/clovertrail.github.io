---
layout: post
title:  "KVM Paravirt Spinlocks"
date:   2017-09-12 13:05:14 +0800
categories: jekyll theme
tail: I'm a tail
---
Spinlocks on paravirtualization environment suffer from two issues: [Lock Holder Preemption and Lock Waiter Preemption][LHP_LWP]. Those problems have not yet been perfectly resolved. If you want to study its current status for prevelant VM, for example, KVM, you will find you cannot enable PV spinlocks for KVM, because it is disabled default.

You should know how to [install KVM][KVM_Install] and create a VM on bare metal Linux. Be default, the KVM host disabled CPU feature of KVM_FEATURE_PV_UNHALT. 
{% highlight ruby linenos %}
void __init kvm_spinlock_init(void)
{
        if (!kvm_para_available())
                return;
        /* Does host kernel support KVM_FEATURE_PV_UNHALT? */
        if (!kvm_para_has_feature(KVM_FEATURE_PV_UNHALT))
                return;

        __pv_init_lock_hash();
        pv_lock_ops.queued_spin_lock_slowpath = __pv_queued_spin_lock_slowpath;
        pv_lock_ops.queued_spin_unlock = PV_CALLEE_SAVE(__pv_queued_spin_unlock);
        pv_lock_ops.wait = kvm_wait;
        pv_lock_ops.kick = kvm_kick_cpu;

        if (kvm_para_has_feature(KVM_FEATURE_STEAL_TIME)) {
                pv_lock_ops.vcpu_is_preempted =
                        PV_CALLEE_SAVE(__kvm_vcpu_is_preempted);
        }
}
{% endhighlight %}

In the above code snippet for kernel 4.12, the kvm_spinlock_init() returns in line 7 because KVM_FEATURE_PV_UNHALT is disabled. You have to add "-cpu host" on qemu command line. But how to add this command line arguments is another problem. I found a workable method. Use "virsh edit XXX" to change the XML configuration for your VM. Here "XXX" is your VM name. In addition, you can dump this XML by virsh dumpxml.

{% highlight ruby linenos %}
hongjiang@lis-f2334:~$ virsh edit ubuntu1704
hongjiang@lis-f2334:~$ virsh dumpxml --inactive --security-info ubuntu1704 > my_kvm.xml
{% endhighlight %}

See the following libvirt XML configuration. You just need to add "qemu:commandline" as the last element for the domain. You can get [the XML configuration here]({{site.url}}/assets/ubuntun1704.xml)

{% highlight ruby linenos %}
<domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>
  <name>ubuntu1704</name>
  <uuid>beac3d12-0942-4cae-bfb3-3d00164012de</uuid>
  <memory unit='KiB'>41943040</memory>
  <currentMemory unit='KiB'>41943040</currentMemory>
  <vcpu placement='static' current='28'>56</vcpu>
  <os>
    <type arch='x86_64' machine='pc-i440fx-zesty'>hvm</type>
    <boot dev='hd'/>
  </os>
  <features>
    <acpi/>
    <apic/>
  </features>
  <cpu mode='host-model'>
    <model fallback='allow'/>
  </cpu>
  <clock offset='utc'>
    <timer name='rtc' tickpolicy='catchup'/>
    <timer name='pit' tickpolicy='delay'/>
    <timer name='hpet' present='no'/>
  </clock>
  <on_poweroff>destroy</on_poweroff>
  <on_reboot>restart</on_reboot>
  <on_crash>restart</on_crash>
  <pm>
    <suspend-to-mem enabled='no'/>
    <suspend-to-disk enabled='no'/>
  </pm>
  <devices>
    <emulator>/usr/bin/kvm-spice</emulator>
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2'/>
      <source file='/var/lib/libvirt/images/ubuntu1704.qcow2'/>
      <target dev='hda' bus='ide'/>
      <address type='drive' controller='0' bus='0' target='0' unit='0'/>
    </disk>
    <disk type='file' device='cdrom'>
      <driver name='qemu' type='raw'/>
      <target dev='hdb' bus='ide'/>
      <readonly/>
      <address type='drive' controller='0' bus='0' target='0' unit='1'/>
    </disk>
    <controller type='usb' index='0' model='ich9-ehci1'>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x05' function='0x7'/>
    </controller>
    <controller type='usb' index='0' model='ich9-uhci1'>
      <master startport='0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x05' function='0x0' multifunction='on'/>
    </controller>
    <controller type='usb' index='0' model='ich9-uhci2'>
      <master startport='2'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x05' function='0x1'/>
    </controller>
    <controller type='usb' index='0' model='ich9-uhci3'>
      <master startport='4'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x05' function='0x2'/>
    </controller>
    <controller type='pci' index='0' model='pci-root'/>
    <controller type='ide' index='0'>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x01' function='0x1'/>
    </controller>
    <controller type='virtio-serial' index='0'>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x06' function='0x0'/>
    </controller>
    <interface type='direct'>
      <mac address='52:54:00:c5:45:73'/>
      <source dev='eno3' mode='bridge'/>
      <model type='rtl8139'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
    </interface>
    <serial type='pty'>
      <target port='0'/>
    </serial>
    <console type='pty'>
      <target type='serial' port='0'/>
    </console>
    <channel type='spicevmc'>
      <target type='virtio' name='com.redhat.spice.0'/>
      <address type='virtio-serial' controller='0' bus='0' port='1'/>
    </channel>
    <input type='mouse' bus='ps2'/>
    <input type='keyboard' bus='ps2'/>
    <graphics type='spice' autoport='yes'>
    <sound model='ich6'>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x0'/>
    </sound>
    <video>
      <model type='qxl' ram='65536' vram='65536' vgamem='16384' heads='1' primary='yes'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x02' function='0x0'/>
    </video>
    <redirdev bus='usb' type='spicevmc'>
      <address type='usb' bus='0' port='1'/>
    </redirdev>
    <redirdev bus='usb' type='spicevmc'>
      <address type='usb' bus='0' port='2'/>
    </redirdev>
    <memballoon model='virtio'>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x07' function='0x0'/>
    </memballoon>
  </devices>
  <qemu:commandline>
    <qemu:arg value='-cpu'/>
    <qemu:arg value='host'/>
  </qemu:commandline>
</domain>
{% endhighlight %}
[LHP_LWP]: http://dl.acm.org/citation.cfm?id=3064180&dl=ACM&coll=DL&CFID=984392879&CFTOKEN=64855677
[KVM_Install]: https://help.ubuntu.com/community/KVM/Installation
