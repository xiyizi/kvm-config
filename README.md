# Note to wayfarers of the internet

This is more of an aide-memoire than a tutorial anyone should try to follow. Lots of the tutorials out there for getting GPU-accelerated virtual machines running are wildly inconsistent and may break your set-up. Ultimately I stumbled on a combination that works for my specific hardware via lots of trial and effort. The following bits of information are for me to use in case I ever break or blow up my computer and have to do it all again.

# kvm-config
XML config and startup settings for GPU-passthrough of a second nVidia card

  - Operating System: Kubuntu 19.10
  - Kernel Version: 5.3.0-18-generic
  - OS Type: 64-bit
  - Processors: 8 × AMD FX(tm)-8350 Eight-Core Processor
  - Memory: 15.6 GiB of RAM

  - Host GPU: Nvidia 750GT 2GB
  - Guest GPU: Nvidia 1050TI 4GB

  - Guest operating system: Windows 10 64bit

  - Virtual Machine Manager version 2.2.1
  - qemu version 4.0

# Key points: 
  - svm must be enabled in BIOS
  - iommu group manipulation must be enabled in BIOS
  - GPU must be isolated and passed through to VFIO via script during boot process, *before* nvidia driver loads
  - the GPU you want to pass through should be in the *second* PCI-e slot
  - sound not working yet - this is a work in progress
  
# How I did it:

install the required packages:

`````shell
sudo apt-get install libvirt0 bridge-utils virt-manager qemu-kvm ovmf
`````
  
make changes to _/etc/default/grub_:

  GRUB_CMDLINE_LINUX_DEFAULT="amd_iommu=on iommu=pt kvm_amd.npt=1 rd.modules-load=vfio-pci"
  
Update grub:

```shell
sudo update-grub
```

Reboot and check everything is all good:

```shell 
virt-host-validate
```

Identify PCI device:

```shell
lspci -nnv
```
  
Find the numbers of your nvidia GPU you want to pass through. A sound device will be part of your graphics card so pass that through too:

```shell
05:00.0 VGA compatible controller [0300]: NVIDIA Corporation GP107 [GeForce GTX 1050 Ti] [10de:1c82] (rev a1) (prog-if 00 [VGA controller])
        Subsystem: Gigabyte Technology Co., Ltd GP107 [GeForce GTX 1050 Ti] [1458:3746]

05:00.1 Audio device [0403]: NVIDIA Corporation GP107GL High Definition Audio Controller [10de:0fb9] (rev a1)
        Subsystem: Gigabyte Technology Co., Ltd GP107GL High Definition Audio Controller [1458:3746]
```
 
The numbers I want to pass through to the VFIO driver are therefore __10de:1c82__ and **10de:0fb9**. Do this by editing _/etc/modprobe.d/vfio.conf_:

```shell
softdep nvidia pre: vfio vfio_pci
softdep nvidiafb pre: vfio vfio_pci
softdep nvidia_drm pre: vfio vfio_pci
softdep nouveau pre: vfio vfio_pci
options vfio-pci ids=10de:1c82,10de:0fb9 disable_vga=1
```
  
And by editing _/etc/initramfs-tools/modules_:

```shell
softdep nvidia pre: vfio vfio_pci
softdep nvidiafb pre: vfio vfio_pci
softdep nvidia_drm pre: vfio vfio_pci
softdep nouveau pre: vfio vfio_pci

vfio
vfio_iommu_type1
vfio_virqfd
options vfio_pci ids=10de:1c82,10de:0fb9
vfio_pci ids=10de:1c82,10de:0fb9
vfio_pci
nvidiafb
nvidia
```
  
And by editing _/etc/modprobe.d/kvm.conf_:

```shell
options kvm ignore_msrs=1
```
  
Update init:
```shell
  sudo update-initramfs -u
  ```
  
Reboot and your second card should be completely isolated from your main operating system. nvidia control panel shouldn't even be able to see it even exists. You can check by running 

```shell
lspci -nnv
```
  
And checking the output:

```shell
  05:00.0 VGA compatible controller [0300]: NVIDIA Corporation GP107 [GeForce GTX 1050 Ti] [10de:1c82] (rev a1) (prog-if 00 [VGA controller])
        Subsystem: Gigabyte Technology Co., Ltd GP107 [GeForce GTX 1050 Ti] [1458:3746]
        Flags: fast devsel, IRQ 16, NUMA node 0
        Memory at f4000000 (32-bit, non-prefetchable) [size=16M]
        Memory at a0000000 (64-bit, prefetchable) [size=256M]
        Memory at b0000000 (64-bit, prefetchable) [size=32M]
        I/O ports at c000 [size=128]
        Expansion ROM at f5000000 [disabled] [size=512K]
        Capabilities: [60] Power Management version 3
        Capabilities: [68] MSI: Enable- Count=1/1 Maskable- 64bit+
        Capabilities: [78] Express Legacy Endpoint, MSI 00
        Capabilities: [100] Virtual Channel
        Capabilities: [250] Latency Tolerance Reporting
        Capabilities: [128] Power Budgeting <?>
        Capabilities: [420] Advanced Error Reporting
        Capabilities: [600] Vendor Specific Information: ID=0001 Rev=1 Len=024 <?>
        Capabilities: [900] Secondary PCI Express <?>
        Kernel driver in use: vfio-pci
```

The nvidia card can now be pass through to a virtual-machine.

Now on to actually creating the virtual machine. The version I used included in the standard Ubuntu repositories as of version 19.04 (version 2.2.1) makes it all pretty simple. First, create a raw image to use as the hard disk for the virtual machine:

```shell
fallocate -l 300G /media/extra/win/win10.img
```

You can enlarge it afterwards if you make it too small but beware it can be a hassle if you need to embiggen it _after_ you've installed Windows to it. Beware of creating an image on a removable drive that your window manager auto-mounts for you. It does something weird to the permissions and doesn't let the machine boot up. I specifically added my second hard drive to _/etc/stab/_ to avoid this problem.

Now fire up Virtual Manager and create a new machine:

![alt text](https://i.imgur.com/ms7ei05.png "Post a comment on this webzone if you want a pizza roll")

![part 2](https://i.imgur.com/6bujvLU.png "fuck movies")

I used the ISO you can download from [Microsoft's website](https://www.microsoft.com/en-gb/software-download/windows10ISO)

![part 3](https://i.imgur.com/OC5mIf3.png "a gangsta ride blades if you ain't gon ride fly then you might as well hate")

I didn't want my virtual machine to gobble up *all* my RAM so I only gave it 12gb.

![part 4](https://i.imgur.com/2uB2QaU.png "no way")

Using the image I created earlier.

![part 5](https://i.imgur.com/WvUPDMO.png "my shit is custom")

Customise installation before continuing is a must so make sure it's checked.

![part 6](https://i.imgur.com/0O53LVQ.png "Q is the best character in Star Trek")

I used Q35 as my machine type and selected UEFI code for my firmware.

![part 7](https://i.imgur.com/3JO8N1E.png "hardware")

Click `add hardware` and select your nvidia card in the PCI selection. Click add hardware and also add your nvidia card's sound controller.

Begin the installation! I was brought t the UEFI bootloader screen so type `exit` to reach the boot selection screen and choose the ISO cd rom device. The monitor you have attached to your pass-through GPU may or may not work at this point. I had to take some extra steps after installation to get it to work.

Specifically, I had to run

```shell
sudo virsh edit win10
```

And add the following xml in between the features brackets:


```xml
  <features>
    <acpi/>
    <apic/>
    <hyperv>
      <relaxed state="on"/>
      <vapic state="on"/>
      <spinlocks state="on" retries="8191"/>
      <vendor_id state="on" value="1234567890ab"/>
    </hyperv>
    <kvm>
      <hidden state="on"/>
    </kvm>
    <vmport state="off"/>
    <ioapic driver="kvm"/>
  </features>
```
That allowed my monitor to work on the next boot of the virtual machine. Then to get sound working via pulseaudio, I had to add to the bottom of my xml file this:

```xml
  <qemu:commandline>
    <qemu:env name="QEMU_AUDIO_DRV" value="pa"/>
    <qemu:env name="QEMU_PA_SERVER" value="/run/user/1000/pulse/native"/>
  </qemu:commandline>
```

Finally, I bought a KVM switching tool. There are several good ones to buy on various electronic websites. I have one that switches monitors and my mouse and keyboard at the press of a button on my desk. Very handy!
