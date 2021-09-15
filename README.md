# Linux Mint - Single GPU passthrout setup

- [Linux Mint - Single GPU passthrout setup](#linux-mint---single-gpu-passthrout-setup)
  - [Basics](#basics)
  - [Edit Grub](#edit-grub)
  - [GPU Isolation](#gpu-isolation)
  - [Install software](#install-software)
  - [Configure Libvirt](#configure-libvirt)
  - [Configure Qemu](#configure-qemu)
  - [Create VM](#create-vm)
  - [Setup hooks](#setup-hooks)
  - [GPU Bios](#gpu-bios)

## Basics

This setup works on my PC, only exception was Rode AI-1. I can't promise it will work for you.


```
RTX 3070
AMD Ryzen 5 5600X
16GB RAM
```


Enable in BIOS / UEFI

```
IOMMU Enabled
SVM Enabled
```

Update OS

```
sudo apt update -y 
sudo apt upgrade -y 
```

## Edit Grub

```bash
sudo nano /etc/default/grub
```


Find 
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
```
Change to
```
GRUB_CMDLINE_LINUX_DEFAULT="amd_iommu=on iommu=pt iommu=1 video=efifb:off quiet splash"
```

Run

```bash
sudo update-grub
sudo reboot
```


After reboot
```
sudo cat /proc/cmdline
```
Output should look like this
```
BOOT_IMAGE=/boot/vmlinuz-5.4.0-60-generic root=UUID=0587b30a-06cf-4df2-82fe-fb8db547e1c5 ro amd_iommu=on iommu=pt iommu=1 video=efifb:off quiet splash vt.handoff=1
```

## GPU Isolation

```bash
#!/bin/bash
shopt -s nullglob
for g in `find /sys/kernel/iommu_groups/* -maxdepth 0 -type d | sort -V`; do
    echo "IOMMU Group ${g##*/}:"
    for d in $g/devices/*; do
        echo -e "\t$(lspci -nns ${d##*/})"
    done;
done;
```

Make sure that GPU is in it's own group, with nothing else being there. 

Example output: 
```
IOMMU Group 1:
	00:01.0 PCI bridge: Intel Corporation Xeon E3-1200 v2/3rd Gen Core processor PCI Express Root Port [8086:0151] (rev 09)
IOMMU Group 2:
	00:14.0 USB controller: Intel Corporation 7 Series/C210 Series Chipset Family USB xHCI Host Controller [8086:0e31] (rev 04)
IOMMU Group 4:
	00:1a.0 USB controller: Intel Corporation 7 Series/C210 Series Chipset Family USB Enhanced Host Controller #2 [8086:0e2d] (rev 04)
IOMMU Group 10:
	00:1d.0 USB controller: Intel Corporation 7 Series/C210 Series Chipset Family USB Enhanced Host Controller #1 [8086:0e26] (rev 04)
IOMMU Group 13:
	06:00.0 VGA compatible controller: NVIDIA Corporation GM204 [GeForce GTX 970] [10de:13c2] (rev a1)
	06:00.1 Audio device: NVIDIA Corporation GM204 High Definition Audio Controller [10de:0fbb] (rev a1)
```

Write down the GPU PCI address, both for video and audio. It was `2b:00.0` and `2b:00.1` in my case.


## Install software
```bash
sudo apt install qemu-kvm libvirt-clients libvirt-daemon-system bridge-utils virt-manager ovmf git
```

## Configure Libvirt
```bash
sudo nano /etc/libvirt/libvirtd.conf
```

Find and uncomment
```
unix_sock_group = "libvirt"
unix_sock_rw_perms = "0770"
```

Add
```
log_filters="1:qemu"
log_outputs="1:file:/var/log/libvirt/libvirtd.log"
```

Run
```bash
sudo usermod -a -G libvirt $(whoami)
sudo systemctl start libvirtd
sudo systemctl enable libvirtd
```

## Configure Qemu
```bash
sudo nano /etc/libvirt/qemu.conf
```

Find
```
#user = "root"
#group = "root"
```

Change to
```
user = "YOUR USERNAME"
group = "YOUR USERNAME"
```

Run
```
sudo systemctl restart libvirtd

sudo usermod -a -G kvm "YOUR USERNAME" 
sudo usermod -a -G libvirt "YOUR USERNAME"

sudo reboot
```

## Create VM

1. Open Virtual-Manager.
2. Create new VM
3. Edit the VM before installation
   * Chipset to Q35
   * Bios to EUFI
   * Enable boot manager
   * Install
4. Add GPU, mouse, keyboard and all devices you need as passthroug.

## Setup hooks
```bash
git clone https://gitlab.com/risingprismtv/single-gpu-passthrough.git
sudo ./single-gpu-passthrough/install_hooks.sh
```

## GPU Bios
Required for RTX 3070. Download your bios from https://www.techpowerup.com/vgabios/. [94.04.3A.40.63](https://www.techpowerup.com/vgabios/228182/msi-rtx3070-8192-201125) in my case. Open in HEX editor, search for text VIDEO.
There should be something like `U...K7000.L.w.VIDEO`, delete EVERYTHING before the `U`. Save as `patch.rom`.

```bash
sudo chmod -R 775 patch.rom
sudo chown yourusername:yourusername patch.rom
sudo mkdir /usr/share/vgabios
sudo mv patch.rom /usr/share/vgabios/
sudo virsh edit win10
```

Find PCI bus for GPU, in my case `0x2b`. There should be `</Source>` under it. Add line under it.
```
<rom file='/usr/share/vgabios/patch.rom'/>
```
Find `</features>` and `</hyperv>` inside of it. Add this before `</hyperv>`.
```
<vendor_id state='on' value='fuckNvidia'>
```




