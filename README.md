# QNAP Thin Logical Volume Data Recovery
## Exposition
Firstly, a huge thank you to [ABLomas (QNAP-LVM-recovery)](https://github.com/ABLomas/QNAP-LVM-recovery) and [max-boehm (qnap-utils)](https://github.com/max-boehm/qnap-utils). Without their work I wouldn't have been able to recover my data. \
The following are the steps I took to recover the data from my QNAP TS-431P2 after the mainboard or CPU died about a year ago. I did have backups of my data, but only the essentials. This was a project for all of the remaining nice-to-have data. \
I consider myself an amateur at all of this, so I have no doubt that this is likely more complicated than it needed to be.

### The Dead NAS - QNAP TS-431P2
One day it died while I was at work and wouldn't turn on (power supply stil works). It was on a surge protector and UPS. It was about 7 years old.
My QNAP setup consisted of 4x 6TB HDDs in a RAID5 array. The array contaned a single 16.35TB **Thin Volume** which is where the problems began. \
I have learned that QNAP uses modified binaries and a patched kernel for their implementation of thin volumes, so I couldn't just use the standard binaries from my Linux distribution to access the data.

### Recovery Setup
I used `dd` to make full disk backups of each of the drives in the array to a new NAS prior to attempting any of this.

#### Failures
1. I mounted the RAID5 array on a system running Linux Mint Debian Edition 6 and attempted to load the logical volume, only to have it error out. I don't remember the errors, it was a long time ago. I learned that QNAP have modified they way their logical volumes work which required their custom kernel.
2. I attempted to compile their kernel (source available at https://sourceforge.net/projects/qosgpl), but as ABLomas found out too, there were many problems and missing references. 
3. I found ABLomas' repository and Hyper-V image many months later which spurred a new round of attempts. I Imaged the logical volume from within the RAID array to my new ASUSTOR NAS and tried using the Hyper-V image on a Lenovo Tiny M710q running Windows 10, but I couldn't figure out a way to get Hyper-V to access the image across the network. I couldn't get Hyper-V to read the images across an SMB share, nor with the original disk images mounted with iSCSI. The VM reported the images as significantly smaller than they should have been. Not sure if it was my implementation or limitations of some of the protocols used. Either way, I got tired and moved on.
2. I then tried using Oracle VirtualBox on Windows 10. Similar problems with getting the disk images to read across the network. But I did get ABLomas's Hyper-V system disk running in VirtualBox.

#### Success
In the end I gave up on the idea of being able to mount the images from a network location and I set up an old computer with 6 SATA ports, and mounted three of the original four RAID5 drives in a degraded state, and the fourth was erased and used to store 5TB of Virtual Disk Images used to exfiltrate the data from the virtual machine. \
I had to set it up this way because the VM only has network drivers for the Intel Gigabit Ethernet Adapter 1000E (e1000e: Intel(R) PRO/1000 Network Driver - 3.2.5-k) and VirtualBox doesn't support that adapter. It *was* detecting the adapters (seen using `lspci`) and there may have been a way to get the kernel to accept additional drivers, but I didn't explore this.

##### PC
My recovery setup was an old PC with at least 5x SATA ports (1x OS, 1x Exfiltration, 3x RAID). \
I installed Linux Mint Debian Edition 7 to an old HDD, though I probably could have used the live USB image if needed. \
The PC used has an old Intel i5-750 and 6GB of DDR RAM. \
I also had an old, but functioning, QNAP TS-110 for the purpose of obtaining the file `libexpat.so.1`.

## Recovery Steps
### Creating the Virtual Machine
1. Download the HyperV virtual machine image from https://github.com/ABLomas/QNAP-LVM-recovery/releases.
2. Extract the file `QNAP_t1/Virtual Hard Disks/darbinis_vhd.vhd` from the 7z archive under the v0.1 release.
3. Install Oracle VirtualBox from https://www.virtualbox.org/wiki/Downloads. I used version 7.2.8.
4. Follow the steps under **Creating Virtual Disk Image from the Host System** to create your exfiltration Virtual Disk Images. Once you have formatted them, you will need to mount one and add the file `libexpat.so.1`. This file is required for mounting the Thin Logical Volume and will need to be copied to the path `/usr/lib/` each time you boot up the virtual machine.
    - Download `libexpat.so.1` from this repository. See **Extracting libexpat.so.1 yourself** if you want to extract the file from QNAP firmware yourself.
5. Create a new virtual machine. I selected the type `BSD`, and the subtype `FreeBSD`.
6. 1 CPU core and 1GB RAM is sufficient.
7. For "Hard Disk", select "Use and Existing Virtual Hard Disk File" and add the "darbinis_vhd.vhd" image. It must be added as an IDE device, not SATA. This was done automatically for me.
8. Click "Finish", then go to the settings of your new VM.
9. Enable **Expert** settings.
10. Under **System > Accelleration**, set the **Paravirtualization Interface** to **Hyper-V**.
11. Under **Serial Ports > Port 1**, enable the serial port with the following settings:
    - Port mode: Host Pipe
    - Connect to existing pipe/socket: False/unchecked
    - Path/Address: `/tmp/vbox_serial_pipe`
12. Under **Storage**, add an **AHCI (SATA)** controller, and add your logical volume .vmdk and all of your exfiltration virtual disk images to it.
13. Save your changes and launch your VM.

### Running the Virtual Machine
1. If you get an error launching your, I had to disable some kernel modules that were impeding VirtualBox's access to the Hyperviser. Try searching the web for your error, or ask an LLM like https://lumo.proton.me. The below worked for me.
    ``` bash
    sudo modprobe -r kvm_intel
    sudo modprobe -r kvm
    ```
2. From a terminal on the Virtual Machine's host, connect to the serial pipe. I had to install it first.
    ``` bash
    sudo apt install socat
    socat -,echo=0,raw UNIX-CONNECT:/tmp/vbox_serial_pipe,escape=0x1d
    ```
3. Log in with the credentials admin/admin.
4. Once logged in, run the QNAP init script.
    ``` bash
    /etc/init.d/init_check.sh
    ```
5. Determine the location of all attached virtual disks and identify them based on expected sizes.
    ``` bash
    fdisk -l
    ```
6. Make the directories and mount the exfiltration VHDs
    ``` bash
    mkdir /mnt/exfil{1..3} /mnt/data
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
    mount -t ext4 /dev/vg1/lv1 /mnt/data
    cd /mnt/data
    ```
10. When you are done recovering your data to the Virtual Disk Images, you will need to turn off the Virtual Machine. Run the `sync` command before poweroff to make sure all pending writes are finalised.

### Extracting recovered data
1. Once the virtual machine is powered off, re-mount your Virtual Disk Images locally and move the data to its final destination.
    ``` bash
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
If you have any functioning QNAP hardware, you can obtain this file yourself by downloading the firmware **TS-X51_20230416-4.5.4.2374** from QNAP's download page for the TS-451, which is the hardware the VM's disk image was created from. I used https://github.com/max-boehm/qnap-utils on original QNAP hardware (a TS-110). You will need to install some packages like XZ and other things I cannot remember. Read the error messages when it fails and install the missing programs using Entware (https://www.myqnap.org/product/entware-std/). If the package does not exist by name, try prepending `lib` to it and/or search the packages list (this was the version for my TS-110: https://bin.entware.net/armv7sf-k3.2/Packages.html).
Once the firmware package is extracted, the file will be located at `TS-X51_20230416-4.5.4.2374/sysroot/usr/lib/libexpat.so.1.6.0`. Rename the file to `libexpat.so.1` and add it to one of your exfiltration disk images. You will need to copy it to `/usr/lib` on the VM. I know it is a different version of the file than the one lvchange required, but it worked.

### Creating Virtual Disk Image from the Host System
Add the current user to the disk group and reboot the system. This is required for the Virtual Machine to be able to read the logical volume in the RAID array.
``` bash
sudo usermod -aG disk $USER
sudo reboot
```

Mount the RAID array in a degraded state. \
Start by stopping md127 and whatever the array is currently mounted as. \
This is because I've already created the VHD pointing at md127.
``` bash
sudo mdadm --stop /dev/md126
sudo mdadm --stop /dev/md127
sudo mdadm --assemble --force /dev/md127 /dev/sdc3 /dev/sdd3 /dev/sde3
```

Create the virtual hard disk referencing md127.
``` bash
sudo VBoxManage internalcommands createrawvmdk -filename "/home/${USER}/Documents/ldvm.vmdk" -rawdisk /dev/md127
```

Mount external storage containing the exfiltration VHDs.
``` bash
sudo mkdir /mnt/exfil
sudo mount /dev/sda1 /mnt/exfil
```

From VirtualBox's media manager, make as many 2TB disk images as you have space for. The maximum size I could create in the user interface was 2TB. Mount them using qemu-utils.
``` bash
sudo apt install qemu-utils
sudo qemu-nbd -c /dev/nbd0 /mnt/exfil/exfil1.vdi
sudo qemu-nbd -c /dev/nbd1 /mnt/exfil/exfil2.vdi
sudo qemu-nbd -c /dev/nbd2 /mnt/exfil/exfil3.vdi
```

Create the new partitions. Repeat for nbd0, nbd1, and nbd2.
``` bash
sudo fdisk /dev/nbd0
n (New partition)
p (Primary partition)
1 (Partition number)
Accept defaults for first and last sector.
w (Write changes to disk)
```

Format as EXT4
``` bash
sudo mkfs.ext4 /dev/nbd0p1
sudo mkfs.ext4 /dev/nbd1p1
sudo mkfs.ext4 /dev/nbd2p1
```

Mount one of the Virtual Disk Images and extract libexpat.so.1 to it.
``` bash
sudo mkdir /mnt/tmpexfil
sudo mount /dev/nbd0p1 /mnt/tmpexfil
sudo mkdir -p /mnt/tmpexfil/usr/lib
sudo 7z e ~/Downloads/libexpat.so.1.7z -o/mnt/tmpexfil/usr/lib/
sudo umount /mnt/tmpexfil
sudo rmdir /mnt/tmpexfil
```

Disconnect the Virtual Disk Images and connect them to your Virtual Machine.
``` bash
sudo qemu-nbd -d /dev/nbd0
sudo qemu-nbd -d /dev/nbd1
sudo qemu-nbd -d /dev/nbd2
```