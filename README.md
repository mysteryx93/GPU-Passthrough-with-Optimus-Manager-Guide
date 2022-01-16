# Guide for GPU Passthrough on laptop with Optimus Manager

After spending a full week trying to get GPU passthrough to work on my laptop, I wanted to share the solution. There are plenty of guides out there, and I'll focus mainly on the information that is missing from those guides.

This guide applies if you're running Linux and use Optimus Manager to manage your dual-graphics.

If you get a freeze issue on `virsh nodedev-detach` with a different setup, you can probably easily adapt the solution to your needs.

My laptop: `Acer Helios 300, Intel Core i7-10750H, NVidia RTX 2060`    
OS: `Garuda Linux` (which comes with Optimus Manager by default)    
VM: `Virtual Machine Manager, QEMU, KVM`

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
Get your VendorId and DeviceId by typing this. Look at Subsystem. 1025 is my vendor-id and 1442 is my device-id.

    lspci -nnks 01:00.

    01:00.0 VGA compatible controller [0300]: NVIDIA Corporation TU106M [GeForce RTX 2060 Mobile] [10de:1f15] (rev a1)
    Subsystem: Acer Incorporated [ALI] Device [1025:1442]


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

    lspci -nnks 01:00.

Undo.

    sudo nohup ./etc/libvirt/hooks/qemu.d/win10-gpu/release/end/unbind_vfio.sh

Try switching back and forth a few times.

It will now automatically switch the GPU when starting your `win10-gpu` VM! Great work.

One last problem. If you don't log back into Linux before Windows 10 boots up, you will have no internet connection. The WiFi connection is tied to the Linux user session. Logging in after won't restore the Windows connection.

[TODO: FIX]


## Performance Tuning

Hugepages: Transparent hugepages work by default. The experts who helped me said that there's no benefit in setting up static hugepages. Even though most guides tell you to do so.

CPU pinning: the pinning strategy depends on your CPU. I made some tests and was surprised by the results.
- No pinning (1 socket, 6 cores, 2 threats): severe stuttering in games!
- Pinning all 12 threads: only slight stuttering, much less than I'd expect
- Pinning 5 cores out of 6: very fluid games, but lower Geekbench score

I decided to pin 5 of 6 cores in `win10-gpu`, and to pin all 12 threads in the `win10` non-GPU VM. It's really a choice between max performance and max responsiveness. You can optimize both VMs differently.

## Final Configuration

Here's my final QEMU configuration for [win10-gpu](https://pastebin.com/n1eRP51b) and for [win10](https://pastebin.com/JFjjPH3F).
