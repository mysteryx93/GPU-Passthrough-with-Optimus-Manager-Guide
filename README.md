# Guide for GPU Passthrough on laptop with Optimus Manager

After spending 5 full days trying to get GPU passthrough to work on my laptop, I wanted to share the solution. There are plenty of guides out there, and I'll focus mainly on the information that is missing from those guides.

This guide applies if you're running Linux and use Optimus Manager to manage your dual-graphics.

My laptop: `Acer Helios 300, Intel Core i7-10750H, NVidia RTX 2060`    
OS: `Garuda Linux` (which comes with Optimus Manager by default)

## Why VM? Setup Objectives

I want to use Linux as my main OS. There are just very few cases where I still need Windows:
- Fortnite, for example, doesn't work on Linux.
- I can replace Adobe Premiere with Linux alternatives, but if I need to render an Adobe project, I'll need Windows.
- If I want to open a .NET WPF or WinForms demo project, that only works on Windows.
 
My goal is to use it as little as possible.

Dual-booting is definitely an option, and I came to a hair of giving up on VM and dual-booting.

Achieving GPU passthrough on a VM however has some advantages over dual-booting:
- Can use both Linux and Windows at the same time, and copy/paste in-between.
- Better encapsulation of Windows. I can be sure it won't screw up my system if it decides to auto-upgrade to Windows 11.
- Some quick tasks are easier in a VM; I won't need to maintain 2 separate Windows installations.
- Avoiding 2 separate Windows installations means saving 25GB for a clean Windows.

My VM setup works like this. I have my Linux displaying on the HDMI output (only the NVidia GPU can display over HDMI). When I start the VM, it transfers Linux to my laptop internal display, logs me out, and boots Windows on the TV. I can press both CTRL to switch keyboard and mouse between Linux on the laptop and Windows on the TV. When I shut down Windows, it logs me out of Linux again, and transfers Linux back to the TV.

I actually have 2 VM configurations setup: win10 and win10-gpu. If I don't want to play games, then I don't need the GPU and don't need to kill my session. I just start the non-GPU session.

## Optimus Guides?

Achieving GPU passthrough on Muxless laptops with Optimus GPU is difficult.

If you have an Optimus GPU, then you need to search "Optimus GPU Passthrough" to find the right solutions. As of writing this, Google will return only 2 guides.

[Lan Tian has a guide from 2020.](https://lantian.pub/en/article/modify-computer/laptop-intel-nvidia-optimus-passthrough.lantian/) It's good to achieve static binding, but you cannot bind or unbind dynamically.

[Misairu-G has a guide from 2018.](https://gist.github.com/Misairu-G/616f7b2756c488148b7309addc940b28) It has a solution for dynamic binding using Bumblebee, which is a dead project that has no update since 2018.

## The Problem

[From the Optimus Manager project:](https://github.com/Askannz/optimus-manager)

> GPU offloading and power management with Nvidia cards are not properly supported on Linux (though there has been some great progress recently), which can make it hard to use your Optimus laptop at full performance. optimus-manager provides a workaround to this problem by allowing you to run your whole desktop session on the Nvidia GPU, while the Intel/AMD GPU only acts as a "relay" between the Nvidia GPU and your screen.

That can be a problem if you attempt to do GPU passthrough. Your system will hang when trying to detach the GPU.

To confirm the problem, use this command to list processes using the GPU.

    nvidia-smi
    
It says that `/usr/lib/Xorg` is using it. Bummer. That's your whole user session. Unless you want to get rid of Optimus Manager, there will be no way of releasing the GPU without killing your session. Also note that to switch between Integrated and Hybrid mode with Optimus Manager, it also logs you out.
 
About the Bumblebee solution in Misairu-G's guide, here's another quote from Optimus Manager project

> optimus-manager is incompatible with Bumblebee since both tools would be trying to control GPU power switching at the same time. If Bumblebee is installed, you must disable its daemon (sudo systemctl disable bumblebeed.service, then reboot). This is particularly important for Manjaro users since Bumblebee is installed by default.

You really are facing 2 separate problems: achieving GPU passthrough on a Muxless Optimus laptop, and getting Optimus Manager to release the GPU.

## Initial Steps

Some people get their GPU passthrough setup in 30 minutes. You won't be one of those people. You're about to embark on a challenging journey. I recommend you to take the time to understand the various gears, and to do it one step at a time.

GPU passthrough is difficult because you get different problems based on your hardware (Optimus Muxless laptop) and software (Optimus Manager). In this case, both the hardware and software bring a separate set of issues.

Start by reading this [pinned post](https://www.reddit.com/r/VFIO/comments/m9xa6o/help_people_help_you_put_some_effort_in/) on the VFIO reddit group, which is a great place for support.

> Youtube tutorial videos inevitably skip some steps because the person making the video hasn't hit a certain problem, has different hardware, whatever. Written resources are the thing you're going to need. This shouldn't be hard to accept; after all, you're asking for help on a text-based medium. If you cannot accept this, you probably should give up on running Windows with GPU passthrough in a VM.

The [VFIO Discord channel](https://discord.gg/QgrqzYqQ) is another great place for support, but first do a lot of reading on your own. You need to understand the gears before asking for help.

I recommend you to configure your VM in various steps:

1. Setup a VM called `win10` without GPU passthrough, using the [Arch page](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF) as main reference (and perhaps simpler guides to start). This Arch page has the most accurate information, but doesn't solve every problems. Install Windows on that VM.
2. Setup a VM called `win10-static` with static vfio-pci binding, meaning Linux won't have access to the GPU at all. Attach it to the same storage. Keep the QXL display. For an Optimus Muxless laptop, [Lan Tian](https://lantian.pub/en/article/modify-computer/laptop-intel-nvidia-optimus-passthrough.lantian/) and [Misairu-G](https://gist.github.com/Misairu-G/616f7b2756c488148b7309addc940b28) guides are your only options. There will be several issues to work-around. You'll have succeeded when you get Windows displaying both in QXL and on the TV.
3. Setup a VM called `win10-gpu` with dynamic binding and unbinding. Using this guide.

## Setting Up Static Binding

You need to [isolate your GPU](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF#Isolating_the_GPU). Add `vfio-pci` values to `/etc/default/grub`. There are plenty of guides explaining the steps.

In my case, I faced 2 issues to get static binding to work.

First, adding vfio-pci devices to grub kernel parameters wouldn't take effect. I had to [take these steps to load VFIO before loading the GPU driver.](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF#Loading_vfio-pci_early)

Then, since the GPU would have a '0000' ID, I needed to add vendor-id and device-id manually, otherwise the GPU would appear in Device Manager, but installing the NVidia driver would fail to recognize the device.

```xml
<domain xmlns:qemu="http://libvirt.org/schemas/domain/qemu/1.0" type="kvm">
...
<qemu:commandline>
  <qemu:arg value="-set"/>
  <qemu:arg value="device.hostdev0.x-pci-sub-vendor-id=0x1025"/>
  <qemu:arg value="-set"/>
  <qemu:arg value="device.hostdev0.x-pci-sub-device-id=0x1442"/>
</qemu:commandline>
```

With QXL display, to unlock resolution above 800x600, go to Device Manager and update the driver for "Basic Display Adapter" with Virtio drivers. You should have already done similar steps to install your hard drive with Virtio drivers.

If GPU passthrough is working, your GPU will appear in Device Manager. Install the official NVidia drivers and cross your fingers.

Some laptops have an issue related to the battery. I did not have that issue.

NVidia recently fixed the infamous `Error 43` so that removes some steps.

I did not have the VBIOS issue. If you do, it won't boot at all.

When booting Windows, I get no display until Windows is fully loaded. To be fair, I'm not sure native BIOS and Windows display on the TV until it is fully loaded. Linux won't display on the TV until after I log in. This can be an issue if Windows does a long update on a black TV without telling me anything. Not sure if there's something that can be done here.

## Setting Up Dynamic Binding

Now that you know your GPU passthrough works, we need to release the GPU to be able to switch it back and forth between Linux and Windows.

We'll use [libvirt hooks](https://passthroughpo.st/simple-per-vm-libvirt-hooks-with-the-vfio-tools-hook-helper/) to run bind and unbind scripts. Run these commands to install libvirt hooks and make it executable.

```
sudo wget 'https://raw.githubusercontent.com/PassthroughPOST/VFIO-Tools/master/libvirt_hooks/qemu' \
     -O /etc/libvirt/hooks/qemu
sudo chmod +x /etc/libvirt/hooks/qemu
```

Add these 2 scripts and make them executable with `chmod +x`. These scripts will automatically run when starting and stopping the `win10-gpu` VM.

The idea is to perform these steps:
1. Switch Optimus Manager to Integrated, otherwise `modprobe -r nvidia` will fail.
2. Unload NVidia drivers
3. Detach devices
4. Load VFIO drivers

The unbind script will perform the exact opposite.

Edit the device IDs in the script to match your hardware! Some laptops have 2 devices, mine has 4. They all need to be transferred together.

Use this command to get your device IDs

    lspci -nnv

Look for your NVidia GPU and note the device id

    01:00.0 VGA compatible controller [0300]: NVIDIA Corporation TU106M [GeForce RTX 2060 Mobile] [10de:1f15] (reva1) (prog-if 00 [VGA controller])

`[10de:1f15]` is what you needed for static binding. Here you need `01:00.0` which becomes `pci_0000_01_00_0`. I also have devices in the same group ending with 1, 2 and 3. List them all in the script. These devices need to stay together.

`/etc/libvirt/hooks/qemu.d/win10-gpu/prepare/begin/bind_vfio.sh`

```
#!/bin/bash
set -x

# Shut down display to release GPU
optimus-manager --switch --no-confirm integrated
systemctl stop display-manager.service

# Unload NVidia
modprobe -r nvidia_uvm
modprobe -r nvidia_drm
modprobe -r nvidia_modeset
modprobe -r nvidia

# Unbind the GPU from display driver
virsh nodedev-detach pci_0000_01_00_0
virsh nodedev-detach pci_0000_01_00_1
virsh nodedev-detach pci_0000_01_00_2
virsh nodedev-detach pci_0000_01_00_3

# Load VFIO
modprobe vfio_pci
modprobe vfio_iommu_type1
modprobe vfio

# Restart display
systemctl restart display-manager.service

```

`/etc/libvirt/hooks/qemu.d/win10-gpu/release/end/unbind_vfio.sh`
```
#!/bin/bash
set -x

# Unload VFIO
modprobe -r vfio_pci
modprobe -r vfio_iommu_type1
modprobe -r vfio

# Unbind the GPU from display driver
virsh nodedev-reattach pci_0000_01_00_0
virsh nodedev-reattach pci_0000_01_00_1
virsh nodedev-reattach pci_0000_01_00_2
virsh nodedev-reattach pci_0000_01_00_3

# Load NVidia
modprobe nvidia_uvm
modprobe nvidia_drm
modprobe nvidia_modeset
modprobe nvidia

# Restart session with GPU set to Hybrid
optimus-manager --switch --no-confirm hybrid
systemctl restart display-manager.service
```

To test your scripts, run `bind_vfio.sh` and the display should switch to your internal display. Run `unbind_vfio.sh` and the display should come back to the HDMI output. It's important to run with `nohup` otherwise the script execution will be killed alongside the session!

    sudo nohup ./etc/libvirt/hooks/qemu.d/win10-gpu/prepare/begin/bind_vfio.sh

Run this command to confirm that all devices are now bound to vfio-pci drivers

    lspci -kn | grep -A 2 01:00.

Undo.

    sudo nohup ./etc/libvirt/hooks/qemu.d/win10-gpu/release/end/unbind_vfio.sh

Try switching back and forth a few times.

It will now automatically switch the GPU when starting your `win10-gpu` VM! Great work.

## Performance Tuning

Hugepages: Transparent hugepages work by default. The experts who helped me said that there's no benefit in setting up static hugepages. Even though most guides tell you to do so.

CPU pinning: the pinning strategy depends on your CPU. I made some tests and was surprised by the results.
- No pinning (1 socket, 6 cores, 2 threats): severe stuttering in games!
- Pinning all 12 threads: only slight stuttering, much less than I'd expect
- Pinning 5 cores out of 6: very fluid games, but lower Geekbench score

I decided to pin 5 of 6 cores in `win10-gpu`, and to pin all 12 threads in the `win10` non-GPU VM. It's really a choice between max performance and max responsiveness. You can optimize both VMs differently.

## Final Configuration

Here's my final QEMU configuration for `win10-gpu`
```
<domain xmlns:qemu="http://libvirt.org/schemas/domain/qemu/1.0" type="kvm">
  <name>win10-gpu</name>
  <uuid>53068249-0936-4fc4-9ca9-f64a40820805</uuid>
  <metadata>
    <libosinfo:libosinfo xmlns:libosinfo="http://libosinfo.org/xmlns/libvirt/domain/1.0">
      <libosinfo:os id="http://microsoft.com/win/10"/>
    </libosinfo:libosinfo>
  </metadata>
  <memory unit="KiB">8192000</memory>
  <currentMemory unit="KiB">8192000</currentMemory>
  <vcpu placement="static">10</vcpu>
  <iothreads>1</iothreads>
  <cputune>
    <vcpupin vcpu="0" cpuset="1"/>
    <vcpupin vcpu="1" cpuset="7"/>
    <vcpupin vcpu="2" cpuset="2"/>
    <vcpupin vcpu="3" cpuset="8"/>
    <vcpupin vcpu="4" cpuset="3"/>
    <vcpupin vcpu="5" cpuset="9"/>
    <vcpupin vcpu="6" cpuset="4"/>
    <vcpupin vcpu="7" cpuset="10"/>
    <vcpupin vcpu="8" cpuset="5"/>
    <vcpupin vcpu="9" cpuset="11"/>
    <emulatorpin cpuset="0,6"/>
    <iothreadpin iothread="1" cpuset="0,6"/>
  </cputune>
  <os>
    <type arch="x86_64" machine="pc-q35-6.2">hvm</type>
    <loader readonly="yes" type="pflash">/usr/share/edk2-ovmf/x64/OVMF_CODE.fd</loader>
    <nvram>/var/lib/libvirt/qemu/nvram/win10-gpu_VARS.fd</nvram>
    <boot dev="hd"/>
  </os>
  <features>
    <acpi/>
    <apic/>
    <hyperv>
      <relaxed state="on"/>
      <vapic state="on"/>
      <spinlocks state="on" retries="8191"/>
      <vpindex state="on"/>
      <synic state="on"/>
      <stimer state="on"/>
      <reset state="on"/>
      <vendor_id state="on" value="1234567890ab"/>
      <frequencies state="on"/>
    </hyperv>
    <vmport state="off"/>
  </features>
  <cpu mode="host-passthrough" check="none" migratable="on">
    <topology sockets="1" dies="1" cores="5" threads="2"/>
    <cache mode="passthrough"/>
    <feature policy="require" name="topoext"/>
  </cpu>
  <clock offset="localtime">
    <timer name="rtc" tickpolicy="catchup"/>
    <timer name="pit" tickpolicy="delay"/>
    <timer name="hpet" present="no"/>
    <timer name="hypervclock" present="yes"/>
  </clock>
  <on_poweroff>destroy</on_poweroff>
  <on_reboot>restart</on_reboot>
  <on_crash>destroy</on_crash>
  <pm>
    <suspend-to-mem enabled="no"/>
    <suspend-to-disk enabled="no"/>
  </pm>
  <devices>
    <emulator>/usr/bin/qemu-system-x86_64</emulator>
    <disk type="file" device="disk">
      <driver name="qemu" type="qcow2" cache="none" io="native" discard="unmap" iothread="1" queues="8"/>
      <source file="/var/lib/libvirt/images/win10.qcow2"/>
      <target dev="vda" bus="virtio"/>
      <address type="pci" domain="0x0000" bus="0x04" slot="0x00" function="0x0"/>
    </disk>
    <disk type="file" device="cdrom">
      <driver name="qemu" type="raw"/>
      <source file="/home/hanuman/Downloads/virtio-win-0.1.208.iso"/>
      <target dev="sda" bus="sata"/>
      <readonly/>
      <address type="drive" controller="0" bus="0" target="0" unit="0"/>
    </disk>
    <controller type="usb" index="0" model="qemu-xhci" ports="15">
      <address type="pci" domain="0x0000" bus="0x02" slot="0x00" function="0x0"/>
    </controller>
    <controller type="sata" index="0">
      <address type="pci" domain="0x0000" bus="0x00" slot="0x1f" function="0x2"/>
    </controller>
    <controller type="pci" index="0" model="pcie-root"/>
    <controller type="pci" index="1" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="1" port="0x10"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x0" multifunction="on"/>
    </controller>
    <controller type="pci" index="2" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="2" port="0x11"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x1"/>
    </controller>
    <controller type="pci" index="3" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="3" port="0x12"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x2"/>
    </controller>
    <controller type="pci" index="4" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="4" port="0x13"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x3"/>
    </controller>
    <controller type="pci" index="5" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="5" port="0x14"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x4"/>
    </controller>
    <controller type="pci" index="6" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="6" port="0x15"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x5"/>
    </controller>
    <controller type="pci" index="7" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="7" port="0x16"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x6"/>
    </controller>
    <controller type="pci" index="8" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="8" port="0x17"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x7"/>
    </controller>
    <controller type="pci" index="9" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="9" port="0x18"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x03" function="0x0"/>
    </controller>
    <controller type="pci" index="10" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="10" port="0x8"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x01" function="0x0" multifunction="on"/>
    </controller>
    <controller type="pci" index="11" model="pcie-to-pci-bridge">
      <model name="pcie-pci-bridge"/>
      <address type="pci" domain="0x0000" bus="0x0a" slot="0x00" function="0x0"/>
    </controller>
    <controller type="pci" index="12" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="12" port="0x9"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x01" function="0x1"/>
    </controller>
    <controller type="virtio-serial" index="0">
      <address type="pci" domain="0x0000" bus="0x03" slot="0x00" function="0x0"/>
    </controller>
    <interface type="network">
      <mac address="52:54:00:f7:03:50"/>
      <source network="network"/>
      <model type="virtio"/>
      <address type="pci" domain="0x0000" bus="0x01" slot="0x00" function="0x0"/>
    </interface>
    <serial type="pty">
      <target type="isa-serial" port="0">
        <model name="isa-serial"/>
      </target>
    </serial>
    <console type="pty">
      <target type="serial" port="0"/>
    </console>
    <channel type="spicevmc">
      <target type="virtio" name="com.redhat.spice.0"/>
      <address type="virtio-serial" controller="0" bus="0" port="1"/>
    </channel>
    <input type="mouse" bus="ps2"/>
    <input type="keyboard" bus="ps2"/>
    <input type="evdev">
      <source dev="/dev/input/by-id/usb-1bcf_USB_Optical_Mouse-event-mouse"/>
    </input>
    <input type="evdev">
      <source dev="/dev/input/by-id/usb-Logitech_Gaming_Keyboard_G213_167C305A3330-event-kbd" grab="all" repeat="on"/>
    </input>
    <graphics type="spice" autoport="yes">
      <listen type="address"/>
      <image compression="off"/>
      <gl enable="no"/>
    </graphics>
    <sound model="ich9">
      <address type="pci" domain="0x0000" bus="0x00" slot="0x1b" function="0x0"/>
    </sound>
    <audio id="1" type="spice"/>
    <video>
      <model type="none"/>
    </video>
    <hostdev mode="subsystem" type="pci" managed="yes">
      <source>
        <address domain="0x0000" bus="0x01" slot="0x00" function="0x0"/>
      </source>
      <address type="pci" domain="0x0000" bus="0x06" slot="0x00" function="0x0"/>
    </hostdev>
    <hostdev mode="subsystem" type="pci" managed="yes">
      <source>
        <address domain="0x0000" bus="0x01" slot="0x00" function="0x1"/>
      </source>
      <address type="pci" domain="0x0000" bus="0x07" slot="0x00" function="0x0"/>
    </hostdev>
    <hostdev mode="subsystem" type="pci" managed="yes">
      <source>
        <address domain="0x0000" bus="0x01" slot="0x00" function="0x2"/>
      </source>
      <address type="pci" domain="0x0000" bus="0x08" slot="0x00" function="0x0"/>
    </hostdev>
    <hostdev mode="subsystem" type="pci" managed="yes">
      <source>
        <address domain="0x0000" bus="0x01" slot="0x00" function="0x3"/>
      </source>
      <address type="pci" domain="0x0000" bus="0x09" slot="0x00" function="0x0"/>
    </hostdev>
    <redirdev bus="usb" type="spicevmc">
      <address type="usb" bus="0" port="2"/>
    </redirdev>
    <redirdev bus="usb" type="spicevmc">
      <address type="usb" bus="0" port="3"/>
    </redirdev>
    <memballoon model="none"/>
  </devices>
  <qemu:commandline>
    <qemu:arg value="-set"/>
    <qemu:arg value="device.hostdev0.x-pci-sub-vendor-id=0x1025"/>
    <qemu:arg value="-set"/>
    <qemu:arg value="device.hostdev0.x-pci-sub-device-id=0x1442"/>
  </qemu:commandline>
  <qemu:capabilities>
    <qemu:del capability="device.json"/>
  </qemu:capabilities>
</domain>
```

Here's my final QEMU configuration for `win10` (no passthrough)
```
<domain type="kvm">
  <name>win10</name>
  <uuid>99105ad3-a74b-44c8-ab28-2967d30bed50</uuid>
  <metadata>
    <libosinfo:libosinfo xmlns:libosinfo="http://libosinfo.org/xmlns/libvirt/domain/1.0">
      <libosinfo:os id="http://microsoft.com/win/10"/>
    </libosinfo:libosinfo>
  </metadata>
  <memory unit="KiB">8192000</memory>
  <currentMemory unit="KiB">8192000</currentMemory>
  <vcpu placement="static">12</vcpu>
  <cputune>
    <vcpupin vcpu="0" cpuset="0"/>
    <vcpupin vcpu="1" cpuset="6"/>
    <vcpupin vcpu="2" cpuset="1"/>
    <vcpupin vcpu="3" cpuset="7"/>
    <vcpupin vcpu="4" cpuset="2"/>
    <vcpupin vcpu="5" cpuset="8"/>
    <vcpupin vcpu="6" cpuset="3"/>
    <vcpupin vcpu="7" cpuset="9"/>
    <vcpupin vcpu="8" cpuset="4"/>
    <vcpupin vcpu="9" cpuset="10"/>
    <vcpupin vcpu="10" cpuset="5"/>
    <vcpupin vcpu="11" cpuset="11"/>
  </cputune>
  <os>
    <type arch="x86_64" machine="pc-q35-6.2">hvm</type>
    <loader readonly="yes" type="pflash">/usr/share/edk2-ovmf/x64/OVMF_CODE.fd</loader>
    <nvram>/var/lib/libvirt/qemu/nvram/win10_VARS.fd</nvram>
    <boot dev="hd"/>
  </os>
  <features>
    <acpi/>
    <apic/>
    <hyperv>
      <relaxed state="on"/>
      <vapic state="on"/>
      <spinlocks state="on" retries="8191"/>
      <vpindex state="on"/>
      <synic state="on"/>
      <stimer state="on"/>
      <reset state="on"/>
      <vendor_id state="on" value="1234567890ab"/>
      <frequencies state="on"/>
    </hyperv>
    <vmport state="off"/>
  </features>
  <cpu mode="host-passthrough" check="none" migratable="on">
    <topology sockets="1" dies="1" cores="6" threads="2"/>
  </cpu>
  <clock offset="localtime">
    <timer name="rtc" tickpolicy="catchup"/>
    <timer name="pit" tickpolicy="delay"/>
    <timer name="hpet" present="no"/>
    <timer name="hypervclock" present="yes"/>
  </clock>
  <on_poweroff>destroy</on_poweroff>
  <on_reboot>restart</on_reboot>
  <on_crash>destroy</on_crash>
  <pm>
    <suspend-to-mem enabled="no"/>
    <suspend-to-disk enabled="no"/>
  </pm>
  <devices>
    <emulator>/usr/bin/qemu-system-x86_64</emulator>
    <disk type="file" device="disk">
      <driver name="qemu" type="qcow2"/>
      <source file="/var/lib/libvirt/images/win10.qcow2"/>
      <target dev="vda" bus="virtio"/>
      <address type="pci" domain="0x0000" bus="0x04" slot="0x00" function="0x0"/>
    </disk>
    <disk type="file" device="cdrom">
      <driver name="qemu" type="raw"/>
      <source file="/home/hanuman/Downloads/virtio-win-0.1.208.iso"/>
      <target dev="sda" bus="sata"/>
      <readonly/>
      <address type="drive" controller="0" bus="0" target="0" unit="0"/>
    </disk>
    <controller type="usb" index="0" model="qemu-xhci" ports="15">
      <address type="pci" domain="0x0000" bus="0x02" slot="0x00" function="0x0"/>
    </controller>
    <controller type="sata" index="0">
      <address type="pci" domain="0x0000" bus="0x00" slot="0x1f" function="0x2"/>
    </controller>
    <controller type="pci" index="0" model="pcie-root"/>
    <controller type="pci" index="1" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="1" port="0x10"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x0" multifunction="on"/>
    </controller>
    <controller type="pci" index="2" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="2" port="0x11"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x1"/>
    </controller>
    <controller type="pci" index="3" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="3" port="0x12"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x2"/>
    </controller>
    <controller type="pci" index="4" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="4" port="0x13"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x3"/>
    </controller>
    <controller type="pci" index="5" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="5" port="0x14"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x4"/>
    </controller>
    <controller type="pci" index="6" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="6" port="0x15"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x5"/>
    </controller>
    <controller type="pci" index="7" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="7" port="0x16"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x6"/>
    </controller>
    <controller type="pci" index="8" model="pcie-to-pci-bridge">
      <model name="pcie-pci-bridge"/>
      <address type="pci" domain="0x0000" bus="0x06" slot="0x00" function="0x0"/>
    </controller>
    <controller type="virtio-serial" index="0">
      <address type="pci" domain="0x0000" bus="0x03" slot="0x00" function="0x0"/>
    </controller>
    <interface type="network">
      <mac address="52:54:00:8c:a3:c9"/>
      <source network="network"/>
      <model type="virtio"/>
      <address type="pci" domain="0x0000" bus="0x01" slot="0x00" function="0x0"/>
    </interface>
    <serial type="pty">
      <target type="isa-serial" port="0">
        <model name="isa-serial"/>
      </target>
    </serial>
    <console type="pty">
      <target type="serial" port="0"/>
    </console>
    <channel type="spicevmc">
      <target type="virtio" name="com.redhat.spice.0"/>
      <address type="virtio-serial" controller="0" bus="0" port="1"/>
    </channel>
    <input type="tablet" bus="usb">
      <address type="usb" bus="0" port="1"/>
    </input>
    <input type="mouse" bus="ps2"/>
    <input type="keyboard" bus="ps2"/>
    <graphics type="spice" autoport="yes">
      <listen type="address"/>
      <image compression="off"/>
    </graphics>
    <sound model="ich9">
      <address type="pci" domain="0x0000" bus="0x00" slot="0x1b" function="0x0"/>
    </sound>
    <audio id="1" type="spice"/>
    <video>
      <model type="qxl" ram="65536" vram="65536" vgamem="16384" heads="1" primary="yes"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x01" function="0x0"/>
    </video>
    <redirdev bus="usb" type="spicevmc">
      <address type="usb" bus="0" port="2"/>
    </redirdev>
    <redirdev bus="usb" type="spicevmc">
      <address type="usb" bus="0" port="3"/>
    </redirdev>
    <memballoon model="virtio">
      <address type="pci" domain="0x0000" bus="0x05" slot="0x00" function="0x0"/>
    </memballoon>
  </devices>
</domain>
```
