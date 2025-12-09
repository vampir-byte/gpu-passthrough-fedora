# GPU Pass-through Guide for Fedora 43.
Short tutorial/guide on how I personally got GPU pass-through to work on my system, since I had to go through various different places to figure this all out. I will link the sources I used below, and also please note this may or may not work for you depending on your hardware and there is numerous different ways to do this, this is just what worked for me.

For context my systems specs are as follows:

- AMD Ryzen 9 7950X
- 64GB DDR5 RAM @ 5600MHz
- AMD Radeon RX 7800 XT
- AMD Radeon RX 6600 XT
- USB Controller
- Fedora 43 (GNOME)

My machine is super overkill for 98% of what I use it for. At a minimum so long as you have a modern Intel or AMD CPU released since the late-2010s, 16GB RAM, plus multiple GPUs (including integrated) you should have all you realistically need for this. 

This doesn't cover both single GPU pass-through nor pass-through with two identical GPUs. I have no intent on covering that as I've no personal need, however I believe the Arch Linux wiki [1] makes mention of what to do so maybe head there. 

Also certain aspects of this guide may be different for NVIDIA cards which I didn't account for having exclusively AMD GPUs, so if something doesn't work it might be because the commands are slightly different for that (consult [1] as it likely will have the corrosponding command, thanks Arch wiki people).

---

### 1. Enabling features, installing software.

Please enable IOMMU and whatever virtualization features your motherboard has for you in the BIOS. They are called different things depending on the vendor, so I suggest looking it up. Also make sure you have whatever virtualization stuff you need. e.g. Virt-Manager, QEMU/KVM, whatever you intend to use. On Fedora (read [2]) the easiest way for me to do this (as I'm lazy) I found is with

`$ sudo dnf install @virtualization`

and after doing that enabling and starting the Systemd related services it installed

`$ sudo systemctl start libvirtd & sudo systemctl enable libvirtd`.

---

### 2. Getting IOMMU ID's.

Get the IOMMU ID's for your GPU you are passing through via lspci. 

run `$ lspci -nnk`.

The ID should be in brackets like `[0000:ffff]`. When doing this make sure you also get your audio output for the GPU too. You can simplfy searching also by adding `| grep '*keyword here*'`.

There's a few other things you may want to figure here too, such as if you want to pass through a USB card so you can use a KVM switch, you should get the ID for that now.

I ran into an issue with certain things being grouped together when I did not want them to be which caused issues with VMs working. This is something which may occur to you and the way to resolve this is using ACS Override patch with your kernel. Me, being lazy, did not want to go through the effort of compiling a kernel with the patch, so I installed the CachyOS kernel from [5] to get around that as it is included. To install that kernel just follow the instructions on the page to add the Copr repo, then install the kernel using dnf for the package `kernel-cachyos`. I will mention the other step related to it in the following section.

---

### 3. Configuring GRUB.

Add this + your ID to your `/etc/default/grub` at `GRUB_CMDLINE_LINUX`.

`GRUB_CMDLINE_LINUX="rhgb quiet loglevel=3 amd_iommu=on iommu=pt vfio-pci.ids=0000:ffff"`

if again you have multiple ID's, add a `,` so its like `vfio-pci.ids=0000:ffff,1111:eeee`. If you are using an Intel CPU, replace the amd_iommu with intel_iommu. There are other commands you can add to `GRUB_CMDLINE_LINUX` which may be beneficial if you'd like, however I prefer this.

Also obviously replace `0000:ffff`...

Run `$ sudo grub2-mkconfig -o /etc/grub2-efi.cfg` to update it officially.

For those who needed the ACS patch, the other step is to add `pcie_acs_override=downstream,multifunction` to `GRUB_CMDLINE_LINUX`. It will ungroup everything which should fix the issue if you again had one.

---

### 4. Configuring Dracut.

Add this to Dracut, making the file `/etc/dracut.conf.d/10-vfio.conf`

`force_drivers+=" vfio vfio_iommu_type1 vfio_pci "`

MIND YOU, it will complain if there's no spaces after and before the quotes. You also may need a different command using an NVIDIA GPU, read [1].

then run `$ sudo dracut -f` to update it officially, overwriting the initramfs.

---

### 5. Checking if everything worked.

Reboot and check if it worked. Do so using `$ sudo dmesg | grep -i vfio` and `$ lspci -nnk` to see. If command one doesn't really make it clear, look at your GPU in the results of command two to see if vfio-pci took control of it e.g. `Kernel driver in use: vfio-pci`

If you installed the CachyOS kernel, do make sure you selected that when you booted your system, else you will be using your kernel without ACS Override. I heavily suggest changing your default kernel to it using grubby (which is pre-installed) via the following:

`sudo grubby --info=ALL | grep -E "index=|kernel="` Will show every kernel installed and their index. Take note of the index of the kernel w/ CachyOS, e.g. `/boot/vmlinuz-6.17.9-cachyos1.fc43.x86_64`. 

Next run `sudo grubby --set-default-index=`, adding whatever index the CachyOS kernel was located at.

Lastly check if it worked with `sudo grubby --default-index`, in my case my CachyOS kernel was located at 0, and the output was `0`, so it worked. Now everytime you boot it should boot to that index/kernel without having to select it in settings.

---
	
### 6. Completion.

Everything should just be set at this point to wire up your card to a monitor and create your virtual machines. From my awareness all virtual machines should be run with Q35 firmware and UEFI BIOS, else you will likely run into issues [3]. You also obviously must include you PCI Host Device in the VM configuration else your VM will not being using it.

If you do run into any issues, e.g. machines not booting or black screens, here are some possible short fixes:
- Disable ROM Bar [4]
- Add this <hyperv> bit to your VM's XML configuration:
  <features>
    <hyperv mode="custom">
      <relaxed state="on"/>
      <vapic state="on"/>
      <spinlocks state="on" retries="8191"/>
      <vendor_id state="on" value="randomid"/>
    </hyperv>
  </features>
- You may be encountering a reset issue which plagues certain AMD GPUs (Vega & RX 7000 Series). I'd suggest searching for more info if thats the case.

Generally in my experience, beyond adding your PCI to Linux and Windows 10 VMs nothing else really needs to be done (at least with my RX 6600 XT) to now have a functioning GPU within the VM. Windows 11 I've no clue on as I try to avoid it like the plague even if Windows 10 is now EOL.

---

### 7. Other notes.

If you are passing through to a macOS VM, outside of the certain steps you must do to properly setup the VM (I suggest https://oneclick-macos-simple-kvm.notaperson535.is-a.dev/) I had to pass the following to get it to boot with my RX 6600 XT. 

<qemu:arg value="-global"/>
<qemu:arg value="ICH9-LPC.acpi-pci-hotplug-with-bridge-support=off"/>`

If you are thinking about passing through a GPU to macOS, it must be both supported and if you have an NVIDIA card it likely needs patched vbios (although I don't see the point of using NVIDIA as it lost support circa macOS Mojave, which has been EOL since 2021). 

If you attempt to run online games, anti-cheats will likely immediately try to ban you. There are ways to get around this, however as I do not play or endorse ever touching software like that, I've zero clue what you would need to do regarding that. Else-wise I hope this has proven some utility, if not in its total length than random snipits to troubleshoot or figure out a problem.

| Sources |                                                                                                          |
|--------|----------------------------------------------------------------------------------------------------------|
| 1      | https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF                                                |
| 2      | https://docs.fedoraproject.org/en-US/quick-docs/virtualization-getting-started/                          |
| 3      | https://gist.github.com/paul-vd/5328d8eb2c626dff36ee143da2e85179                                         |
| 4      | https://forum.level1techs.com/t/attempting-single-gpu-passthrough-but-only-getting-a-black-screen/167867 |
| 5      | https://copr.fedorainfracloud.org/coprs/bieszczaders/kernel-cachyos/		                  								|
