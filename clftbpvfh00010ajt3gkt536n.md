---
title: "TurtleNet 2.5: GPU Passthrough w      ith Proxmox"
datePublished: Wed Mar 29 2023 06:46:23 GMT+0000 (Coordinated Universal Time)
cuid: clftbpvfh00010ajt3gkt536n
slug: 25-gpu-passthrough
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1680071777270/981ec289-be46-476b-9d75-8db880ee6db3.png
tags: self-hosted, gpu, homelab, proxmox

---

## Introduction

This section is optional, and provides additional context in the case that you would like to attach a GPU to your VM. This can be useful for a variety of tasks, such as setting up a cloud gaming server (which we'll do now) or performing machine learning- or graphics- related tasks.

As a disclosure, you can find lots of guides on how to do this online: [here's one](https://3os.org/infrastructure/proxmox/gpu-passthrough/gpu-passthrough-to-vm/), and [here's another](https://www.reddit.com/r/homelab/comments/b5xpua/the_ultimate_beginners_guide_to_gpu_passthrough/). The information here summarizes the process I took to get my setup working, but yours may be different so just be patient and try more than one guide if it's not doing what you expect it to!

### Disclaimer: Multiple GPUs make life easier!

Essentially, what GPU passthrough does is that it donates the entire graphics card over to the VM. That means that Proxmox, and by extension every other VM, will no longer be able to access the card you pass through!

For most cases, this should be perfectly fine (as long as you're not running anything that requires a GUI, like Windows, on other VMs). However, you'll probably find it much easier to have two GPUs: one cheap, low-powered primary GPU to drive Proxmox, and a higher-powered GPU to pass through to the VM that needs it.

## Preliminary Checks

1. **Make sure that the GPU is plugged in and detected by Proxmox:** If the hardware is faulty, then none of this will work of course! You can ensure that the GPU is detected by entering the command `lspci`. Your GPU should show up in the output, along with an ID (such as `06:00`).
    
2. **Make sure that the VM you want to pass the GPU through to is working:** Before doing the passthrough, the VM should be accessible via the Proxmox console. You should also set up SSH, Remote Desktop, Parsec, or another service that will allow you to access it if it's working, since the Proxmox console will be disabled after passthrough!
    
3. **Make sure that no other VM's are using the GPU at the same time:** Only one VM can use a passed-through GPU at a time. If another VM needs access to the GPU, either ensure that only one of them is running at a time, or get a second GPU to pass both through.
    

## Follow the guide

In the interest of conciseness, I will defer to the large collection of pre-existing guides. Try your favorite one, and come back when you're done, even if things aren't working!

### If things are working

If GPU passthrough is working, you'll notice the following:

* Attempting to connect via console will give an error.
    
* Plugging in a monitor via HDMI/DP directly to the passed through GPU will give the VM output, not the proxmox console.
    
* Even if ballooning is enabled, the VM should use 100% of its allocated RAM.
    
* If you set up SSH or another form of access, you should be able to access the VM as usual from other devices.
    

Should the above be true, congratulations! You have successfully implemented GPU passthrough.

### When things are not working

Chances are that passthrough didn't work the first time around. (It didn't for me either!) Here are a few debugging steps I followed that worked for me:

1. If you're passing through to Windows, your PCI device should look like this:
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1680071798824/6cd72187-4e04-484e-9b92-d1fee8d960ac.png align="center")
    
    If you're passing through to Linux, your PCI device should look the same as above, except "Primary GPU" should be unchecked.
    
2. Make sure that the "Machine Type" of the VM is q35.
    
3. If you're using an NVIDIA 10-series GPU (like a GTX 1070), [this ROM patcher](https://github.com/Matoking/NVIDIA-vBIOS-VFIO-Patcher) might be necessary.
    
4. It's possible that the GPU is being used by another process (Simple Framebuffer seems to be a common culprit). You can use these steps to fix this:
    
    1. Run `cat /proc/iomem` in the Proxmox shell to view a list of PCI devices. You should be able to identify your GPU in this list, and it should not be used by any processes (assuming the VM is shut off). For example, my GPU is device `06:00` and my output looks like this:
        
        ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1680072109066/cd5ea87e-17bd-4295-94ec-4670cf312f71.png align="center")
        
    2. If your output does not look like the above (might have more things under the "PCI Bus 0000:06"), run the following code block, replacing `XX` with the desired ID:
        
        ```bash
             	echo 1 > /sys/bus/pci/devices/0000\:XX\:00.0/remove \
        		echo 1 > /sys/bus/pci/rescan
        		echo simple-framebuffer.0 > /sys/bus/platform/drivers/simple-framebuffer/unbind
        ```
        
        You may need to run this every time the server reboots.
        
5. If none of the above works, view the error log using `tail -100 /var/log/syslog` . If the system is having trouble processing the GPU, you'll probably get a huge amount of error spam in this output that you can paste into Google and get more useful results.