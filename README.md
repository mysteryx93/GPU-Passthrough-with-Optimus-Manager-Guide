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

Some people get their GPU passthrough setup in 30 minutes. You won't be one of those people. You're about to embark on a challenging journey.

Start by reading this [pinned post](https://www.reddit.com/r/VFIO/comments/m9xa6o/help_people_help_you_put_some_effort_in/) on the VFIO reddit group, which is a great place for support.

> Youtube tutorial videos inevitably skip some steps because the person making the video hasn't hit a certain problem, has different hardware, whatever. Written resources are the thing you're going to need. This shouldn't be hard to accept; after all, you're asking for help on a text-based medium. If you cannot accept this, you probably should give up on running Windows with GPU passthrough in a VM.

The [VFIO Discord channel](https://discord.gg/QgrqzYqQ) is another great place for support, but first do a lot of reading on your own.

I recommend you to configure your VM in various steps:

1. Setup a VM called `win10` without GPU passthrough, using the [Arch page](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF) as main reference (and perhaps simpler guides to start). This Arch page has the most accurate information, but doesn't solve every problems. Install Windows on that VM.
2. Setup a VM called 'win10-static` with static vfio-pci binding, meaning Linux won't have access to the GPU at all. For an Optimus Muxless laptop, [Lan Tian](https://lantian.pub/en/article/modify-computer/laptop-intel-nvidia-optimus-passthrough.lantian/) and [Misairu-G](https://gist.github.com/Misairu-G/616f7b2756c488148b7309addc940b28) guides are your only options. There will be several issues to work-around. You'll have succeeded when Windows appears on the TV.
3. Setup a VM called `win10-gpu` with dynamic binding and unbinding. Using this guide.

