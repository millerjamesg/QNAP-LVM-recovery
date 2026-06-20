# QNAP Thin Logical Volume Data Recovery
## Exposition
Firstly, a huge thank you to [ABLomas (QNAP-LVM-recovery)](https://github.com/ABLomas/QNAP-LVM-recovery) and [max-boehm (qnap-utils)](https://github.com/max-boehm/qnap-utils). Without their work I wouldn't have been able to recover my data. \
The following are the steps I took to recover the data from my QNAP TS-431P2 after the mainboard or CPU died about a year ago. I did have backups of my essential data, so this was a project for all of the remaining nice-to-have data. \
I consider myself an amateur at all of this, so I have no doubt that this is more complicated than it needed to be.

### The Dead NAS - QNAP TS-431P2
One day it died while I was at work and wouldn't turn on (power supply still works). It was on a surge protector and UPS. It was about 7 years old.
My QNAP setup consisted of 4x 6TB HDDs in a RAID5 array. The array contained a single 16.35TB **Thin Volume** which is why this was a problem. \
I have learned that QNAP uses modified binaries and a patched kernel for their implementation of thin volumes, so I couldn't just use the standard binaries from my Linux distribution to access the data.

### Recovery Setup
I used `dd` to make full disk backups of each of the drives in the array to a new NAS prior to attempting any of this.

#### Failures
1. I mounted the RAID5 array on a system running Linux Mint Debian Edition 6 and attempted to load the logical volume, only to have it error out. I don't remember the errors, it was a long time ago. I learned that QNAP have modified they way their logical volumes work which required their custom kernel.
2. I set up my old QNAP TS-110 with an old hard drive in the hope that even though it is a single-disk unit, it had the binaries for a multi-disk setup. It did not.
3. I attempted to compile their kernel (source available at https://sourceforge.net/projects/qosgpl), but as ABLomas found out too, there were many problems and missing references.
4. I found ABLomas' repository and Hyper-V image many months later which spurred a new round of attempts. I imaged the logical volume from within the RAID array to my new ASUSTOR NAS and tried using the Hyper-V image on a Lenovo Tiny M710q running Windows 10, but I couldn't figure out a way to get Hyper-V to access the image across the network. I couldn't get Hyper-V to read the images across an SMB share, nor with the original disk images mounted with iSCSI. The VM reported the images as significantly smaller than they should have been. Not sure if it was my implementation or limitations of some of the protocols used. Either way, I got tired and moved on.
5. I then tried using Oracle VirtualBox on Windows 10. Similar problems with getting the disk images to read across the network. But I did get ABLomas's Hyper-V system disk running in VirtualBox.

#### Success
In the end I gave up on the idea of being able to mount the images from a network location and I set up an old computer with 6 SATA ports. I mounted three of the original four RAID5 drives in a degraded state, and the fourth was erased and used to store 5TB of Virtual Disk Images used to exfiltrate the data from the virtual machine. \
I had to set it up this way because the QNAP VM only had network drivers for the Intel Gigabit Ethernet Adapter 1000E (e1000e: Intel(R) PRO/1000 Network Driver - 3.2.5-k) and VirtualBox doesn't support that adapter. It *was* detecting the virtual ethernet adapters attached to it (seen using `lspci`) and there may have been a way to get the kernel to accept additional drivers, but I didn't explore this.

##### PC
My recovery setup was an old PC with 6x SATA ports (1x OS, 1x Exfiltration, 3x RAID, 1x unused). \
I installed Linux Mint Debian Edition 7 to an old HDD, though I probably could have used the live USB image if needed. \
The PC used has an old Intel i5-750 and 6GB of DDR RAM. \
I also had an old, but functioning, QNAP TS-110 for the purpose of obtaining the file `libexpat.so.1`.

## Recovery Steps
### Creating the Virtual Machine
You can follow the steps to create the virtual machine, or download an export of it from Releases.
1. Download the HyperV virtual machine image from https://github.com/ABLomas/QNAP-LVM-recovery/releases.
2. Extract the file `QNAP_t1/Virtual Hard Disks/darbinis_vhd.vhd` from the 7z archive under the v0.1 release.
3. Install Oracle VirtualBox from https://www.virtualbox.org/wiki/Downloads. I used version 7.2.8.
4. Follow the steps below, under **Creating Virtual Disk Image from the Host System** to create your exfiltration Virtual Disk Images. Once you have formatted them, you will need to mount one and add the file `libexpat.so.1`. This file is required for mounting the Thin Logical Volume and will need to be copied to the path `/usr/lib/` each time you boot up the virtual machine.
    - Download `libexpat.so.1` from this repository. See **Extracting libexpat.so.1 yourself** if you want to extract the file from QNAP firmware yourself.
5. Create a new virtual machine. I selected the type `BSD`, and the subtype `FreeBSD`.
6. 1 CPU core and 1GB RAM was sufficient.
7. For "Hard Disk", select "Use an Existing Virtual Hard Disk File" and add the "darbinis_vhd.vhd" image. It must be added as an IDE device, not SATA. This was done automatically for me.
8. Click "Finish", then go to the settings of your new VM.
9. Enable **Expert** settings.
10. Under **System > Acceleration**, set the **Paravirtualization Interface** to **Hyper-V**.
11. Under **Serial Ports > Port 1**, enable the serial port with the following settings:
    - Port mode: Host Pipe
    - Connect to existing pipe/socket: False/unchecked
    - Path/Address: `/tmp/vbox_serial_pipe`
12. Under **Storage**, add an **AHCI (SATA)** controller, and add your logical volume .vmdk and all of your exfiltration Virtual Disk Images to it.
13. Save your changes and launch your VM.

### Running the Virtual Machine
1. If you get an error launching your virtual machine, I had to disable some kernel modules due to the error `VT-x was in use by another Hypervisor VERR_VMX_IN_VMX_ROOT_MODE`. For that error, the following worked for me. If you get a different error, try searching the web, or ask an LLM like https://lumo.proton.me.
    ``` bash
    sudo modprobe -r kvm_intel
    sudo modprobe -r kvm
    ```
2. From a terminal on the Virtual Machine's host, connect to the serial pipe. I had to install `socat` first. I never did get the escape sequence working.
    ``` bash
    sudo apt install socat
    socat -,echo=0,raw UNIX-CONNECT:/tmp/vbox_serial_pipe,escape=0x1d
    ```
3. Log in with the credentials `admin/admin`.
4. Once logged in, run the QNAP init script. Expect lots of errors, but carry on anyway.
    ``` bash
    /etc/init.d/init_check.sh
    ```
5. Determine the location of all attached virtual disks and identify them based on expected sizes.
    ``` bash
    fdisk -l
    ```
6. Make the directories and mount the exfiltration VHDs
    ``` bash
    mkdir /mnt/exfil{1..3}
    mount -t ext4 /dev/sd? /mnt/exfil1
    mount -t ext4 /dev/sd? /mnt/exfil2
    mount -t ext4 /dev/sd? /mnt/exfil3
    ```
7. Copy the missing libexpat.so.1 file to the local filesystem so that thin logical volumes can be mounted.
    ``` bash
    cp -a /mnt/exfil1/usr /
    ```
8. Symlink the logical volume so as to masquerade as a RAID array, launch the meta daemon and activate the logical volume.
    ``` bash
    ln -s /dev/sd? /dev/md0
    lvmetad
    pvscan --cache /dev/md0
    pvs
    lvchange -a y vg1/lv1
    ```
9. Mount the EXT4 partition within the logical volume and commence data extraction/exfiltration.
    ``` bash
    mkdir /mnt/data
    mount -t ext4 /dev/vg1/lv1 /mnt/data
    cd /mnt/data
    ```
10. When you are done recovering your data to the Virtual Disk Images, you will need to sync and unmount your Virtual Disk Images and logical volume before turning off the Virtual Machine.
    ``` bash
    sync
    umount /mnt/data /mnt/exfil{1..3}
    # Then hard power-off the VM.
    ```

### Extracting recovered data
1. Once the virtual machine is powered off, re-mount your Virtual Disk Images locally and move the data to its final destination.
    ``` bash
    sudo modprobe nbd # Only required if you've rebooted your system since you last ran it.
    sudo qemu-nbd -c /dev/nbd0 /mnt/exfil/exfil1.vdi
    sudo qemu-nbd -c /dev/nbd1 /mnt/exfil/exfil2.vdi
    sudo qemu-nbd -c /dev/nbd2 /mnt/exfil/exfil3.vdi
    sudo mkdir /mnt/exfil{1..3}
    sudo mount /dev/nbd0p1 /mnt/exfil1
    sudo mount /dev/nbd1p1 /mnt/exfil2
    sudo mount /dev/nbd2p1 /mnt/exfil3
    ```
2. Rinse-and-repeat as many times as needed with the amount of available storage on hand.

## Extra information
### Extracting libexpat.so.1 yourself
If you have any functioning QNAP hardware, you can obtain this file yourself by downloading the firmware **TS-X51_20230416-4.5.4.2374** from QNAP's download page for the TS-451 – the hardware version the source of the VM's disk image was seemingly created from. \
I used https://github.com/max-boehm/qnap-utils on original QNAP hardware (a TS-110).
I did need to use Entware (https://www.myqnap.org/product/entware-std/) to install some additional packages such as:
- cpio
- liblzma
- xz-utils \
and maybe some others that I cannot remember. If you encounter errors, read the failure messages and install the missing packages. \
If the package does not exist by name, try prepending `lib` to it and/or search the packages list (this was the version for my TS-110: https://bin.entware.net/armv7sf-k3.2/Packages.html).
Once the firmware package is extracted, the file will be located at `TS-X51_20230416-4.5.4.2374/sysroot/usr/lib/libexpat.so.1.6.0`. Rename the file to `libexpat.so.1` and add it to one of your exfiltration disk images. You will need to copy it to `/usr/lib` on the VM. While it is a different version than the one lvchange sought, it worked.

### Creating Virtual Disk Image from the Host System
Add the current user to the disk group and reboot the system. This is required for the Virtual Machine to be able to read the physical devices containing the RAID array.
``` bash
sudo usermod -aG disk $USER
sudo reboot
```

Determine the device location of the logical volume in the RAID array. I did so by size.
``` bash
lsblk -ndp /dev/md* -o NAME,SIZE,TYPE,STATE,MOUNTPOINT 2>/dev/null | grep md
```

Because the array was in a degraded state, I needed to force the array to be mounted.
``` bash
sudo mdadm --assemble --force /dev/md127 /dev/sdc3 /dev/sdd3 /dev/sde3
```

Create the virtual hard disk referencing md127.
``` bash
sudo VBoxManage internalcommands createrawvmdk -filename "/home/${USER}/Documents/ldvm.vmdk" -rawdisk /dev/md127
```

For subsequent reboots of the host computer, the device name would change and I would move it back to it's original location so that I didn't need to recreate the .vmdk file each time. \
I would stop whatever was on md127 and the new location of the array, then reassemble it where it used to be.
``` bash
sudo mdadm --stop /dev/md128 # New location of the array.
sudo mdadm --stop /dev/md127 # Another parition mounted here.
sudo mdadm --assemble --force /dev/md127 /dev/sdc3 /dev/sdd3 /dev/sde3 # Reassembling the array in the original location.
```

Mount storage device that will contain the exfiltration VHDs. Mine was a 6TB HDD connected via SATA. I used lsblk to find its location.
``` bash
sudo mkdir /mnt/exfil
lsblk
sudo mount /dev/sda1 /mnt/exfil
```

From VirtualBox's Media Manager, make as many 2TB Virtual Disk Images as you have space for. The maximum size I could create in the user interface was 2TB. Mount them to the host using qemu-utils.
``` bash
sudo apt install qemu-utils
sudo modprobe nbd # Required again after every reboot.
sudo qemu-nbd -c /dev/nbd0 /mnt/exfil/exfil1.vdi
sudo qemu-nbd -c /dev/nbd1 /mnt/exfil/exfil2.vdi
sudo qemu-nbd -c /dev/nbd2 /mnt/exfil/exfil3.vdi
```

Create the new partitions. Repeat for the remaining Virtual Disk Images.
``` bash
sudo fdisk /dev/nbd0
n (New partition)
p (Primary partition)
1 (Partition number)
# Accept defaults for first and last sector to use the whole Virtual Disk Image.
w (Write changes to disk)
```

Format as EXT4, while turning off metadata checksums, which seemingly isn't supported by the VM's kernel.
``` bash
sudo mkfs.ext4 -O ^metadata_csum /dev/nbd0p1
sudo mkfs.ext4 -O ^metadata_csum /dev/nbd1p1
sudo mkfs.ext4 -O ^metadata_csum /dev/nbd2p1
```

Mount one of the Virtual Disk Images and copy libexpat.so.1 to it.
``` bash
sudo mkdir /mnt/tmpexfil
sudo mount /dev/nbd0p1 /mnt/tmpexfil
sudo mkdir -p /mnt/tmpexfil/usr/lib
sudo cp ~/Downloads/libexpat.so.1 /mnt/tmpexfil/usr/lib/
sudo umount /mnt/tmpexfil
sudo rmdir /mnt/tmpexfil
```

Disconnect the Virtual Disk Images from the host system, then connect them to your Virtual Machine.
``` bash
sudo qemu-nbd -d /dev/nbd0
sudo qemu-nbd -d /dev/nbd1
sudo qemu-nbd -d /dev/nbd2
```