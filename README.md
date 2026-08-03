# Virtual Director

**A work-in-progress to virtualise the CA10 Automation Controller from Control4 for use with Type 1 & Type 2 hypervisors.**

The Director application is at the core of the automation ecosystem that Control4 built for use with their hardware. Depending on the size of your automation requirements, their hardware portfolio comprises a number of  devices capable of acting as the primary controller. However, none of their devices were built with adequate resiliency to mitigate a hardware failure meaning any such incident would cause the immediate loss of access to your devices controlling the lights, HVAC or security system to your property.

Unlike other automation ecosystems currently available, Control4 do not have a development roadmap that allows for the Director to run on virtualised systems. This is an effort to implement just that.

✅ **Choice of Hypervisor** - Depending on your infrastructure, you can now run the Director on either a single machine running Oracle's VirtualBox or via a fully resilient VMware vSphere environment.

⚙️ **Customise the hardware** - Larger projects will benefit from allocating more than the default 8GB RAM available in the hardware CA10 (4GB in other lesser devices).

🛡️ **Automated Failover/Reduced Downtime** - Leverage the benefits of virtualisation by creating snapshots, maintain uptimes by using vMotion across multiple ESXi servers whilst removing the need to register a replacement Controller online before it can be used.

## Pre-requisites
1. An existing CA10 to obtain the necessary files from the recovery partition;
2. Practical knowledge of both the Linux operating system and one of either Oracle VirtualBox or VMware ESXi; and
3. Matching patch files to the version of your current Control4 OS.

## Installation
The build process is fairly straightforward:  you’ll need the included custom kernel, three patch files and my patching script to assemble the required files. I’ve included a bootable VDI image - you drop the patched files in, boot it in VirtualBox or ESXi (once converted to VMDK), and it will build the Director automatically from the factory scripts.

From testing, a fully working system should be built within 3 minutes of first boot and as long as you have the patch files (which are matched to that OS version), the script will build it. 

### ⚡ Overview
A correct implementation will follow these three stages: -
1. Downloading the factory files from your CA10;
2. Patching and assembling the boot disk; and
3. Configuring the VM

### Stage 1: Downloading the Factory Files
This is quite straightforward and involves SSH’ing into your CA10, mounting the recovery partition and downloading the compressed archive and boot packages. SSH access is crucial I’m afraid so without it, there is no way forward.
1. Download the patch files for the version of your OS from here.
2. SSH into your CA10 and run the following commands: -
```
> mkdir /mnt/media/disk
> mount /dev/md/recfs /mnt/media/disk
```
There will now be a “disk” folder under the already-shared “media” folder on your Director. The Director shares folders over the network via SMB/CIFS so this could either be accessed using a file browser (at \\director_IP\media\disk) or via an SFTP/SCP file client like WinSCP. You will need the following files: -

**```recfs.tar.xz```**
**```kernel-lb.deb```**
**```kernel-modules-lb.deb```**

Close your file client, unmount the partition and clean up. Via SSH: -
```
> umount -l /mnt/media/disk
> rmdir /mnt/media/disk
```

At this stage, I’d recommend saving these three files somewhere for safekeeping and making copies to work with. Rename those copies to **```recfs_FACTORY.tar.xz```**, **```kernel-lb_FACTORY.deb```** and **```kernel-modules-lb_FACTORY.deb```**. It’s possible the recovery partition will contain an older version of the OS  than the one running but this is rare. There are workarounds if this is the case.

### Stage 2 - Patching and assembling the boot disk
Access to a system running an updated Linux OS is now needed (***if you do not have one, you can download a Live CD/DVD and use VirtualBox to boot a temporary workspace in RAM to carry out the next steps***). You’ll also need to enable SSH access. Finally, make sure you have downloaded a copy of the unified patcher with associated patch files for your running Control4 OS from the [Releases](https://github.com/ThePopolou/Virtual-Director/releases) section. 

1. In Linux, copy across the factory files and extract the downloaded patch files all to the same folder.
2. In addition to your three factory files, you should also now have the following build files: -
   1. ```unified_patcher.sh```
   2. ```lb_boot_stick.vdi```
   3. ```kernel-lb_PATCH.vcdiff```
   4. ```kernel-modules-lb_PATCH.vcdiff```
   5. ```recfs_PATCH.vcdiff```

3. Please ensure **```xdelta3```** is installed on your Linux system.
4. Make sure that the file names to the factory files all have **```_FACTORY```** appended to their names. The script will be looking for files with this name pattern when it runs.
5. Open a terminal and from your folder start the script as root: -

```
> sudo ./unified_patcher.sh
```

6. The script will check its directory for the files and any that are missing will be flagged. The patcher will also allow you to proceed whether or not it finds all the factory files so be sure this is what you need before you continue.
7. You should then see the output from different stages of the script as it reconfigures the factory files. Depending on your system, it could take a couple of minutes to complete. You will get an OUTPUT folder with the reconstructed files inside. The patched files will be renamed to their factory names with almost similar sizes to the originals. If not then something went wrong.
8. Using the VDI boot image, mount this in VirtualBox and open the disk as root. You now want to copy the files from the OUTPUT folder to the root of the bootable image. You should then have the three files plus a boot folder.
9. Unmount the VDI, remove it from the Linux VM session and shut it down. If you want, you can also move the OUTPUT folder containing the patched binaries out of the VM to a safe place for future use otherwise it will be lost once the Linux VM is turned off.

At this point, you should now have a fully bootable image of the OS installer with the patched binaries inside. 

### Stage 3 - Configuring the VM
The next steps are broken down into two parts: Part A is the same irrespective of which hypervisor you choose whilst the steps to follow in Part B depends on which Hypervisor you're using and involves adding custom entries in the VM configuration file created for the virtual CA10 instance (**```.vbox```** for VirtualBox or **```.vmx```** for ESXi). This is necessary due to how Control4 programmed hardware detection of the CA10 in software. VirtualBox was designed to be far more liberal with some low-level system calls whilst VMware took a very different (and stringent) stance so both hypervisors require unique and custom parameters to overcome these hurdles.

#### Part A
The following steps are the same for both hypervisors: -
1. Create a new VM with 4 CPUs (two sockets, two cores) and as much RAM as you require (8GB min).
2. Attach two new SATA disks of 40GB in size each (your choice whether to make them thin or thick). Make sure the controller is SATA and add the first VM disk to the first SATA port (0:0) and the second VM disk to the second port (0:1).
3. Attach the bootable image file as a third SATA disk and (important) make sure it is on the third SATA port (0:2) or higher.
4. Configure the VM to use UEFI (but disable secure boot).
5. If asked, set the OS as an “Other Linux” 64-bit system (or Debian, say).
6. Attach a single NIC either in bridged or NAT mode. You can add a second but unnecessary if you have fallback connections (in ESXi say).
7. A critical step is to pick a MAC address for the NIC. A standard MAC is 12 hexadecimal characters (6 bytes) so choosing a new address will first require use of Control4’s prefix of **```00 0F FF```** plus 6 random hexadecimal characters of your choice. This means there are more than 16 million variations to choose from so plug in any you want to make a compliant MAC address (or better, use google to generate one so there is some randomisation to it). Make a note of it and then add this to the NIC on the VM settings screen.
8. It is often best practice not to enable other features that won’t be needed (USB, Audio, Nested Virtualisation) but the custom kernel I built does have the drivers IF you needed them. I left them off.

#### Part B
**VirtualBox**
1. Open a command prompt in the VirtualBox program folder (```C:\Program Files\Oracle\VirtualBox```) and (assuming the name of your VM is **```CA10```** and the MAC you selected is **```000FFFA1B2C3```**), run the following commands: -

```
> VBoxManage setextradata CA10 "VBoxInternal/Devices/efi/0/Config/DmiSystemVendor" "Control4"
> VBoxManage setextradata CA10 "VBoxInternal/Devices/efi/0/Config/DmiSystemProduct" "Control4 CA-10 Automation Controller"
> VBoxManage setextradata CA10 "VBoxInternal/Devices/efi/0/Config/DmiSystemSerial" "000FFFA1B2C3"
> VBoxManage setextradata CA10 "VBoxInternal/Devices/efi/0/Config/DmiSystemFamily" "Automation Controller"
> VBoxManage setextradata CA10 "VBoxInternal/Devices/efi/0/Config/DmiSystemSKU" "C4-CA10"
```
>***Optional*** - you can add a serial port to monitor the boot process, diagnostic logging and use an interactive console. This is done in the VM settings by enabling Port 1, using the first available COM port of your local machine and set the Port Mode to “Host Pipe”. The path address will then be \\.\pipe\serial1 (serial1 for COM1, serial2 for COM2, etc..). Then using a telnet client, you can connect to that port (at 115200 bps) after the VM has been started. 

**VMware ESXi/vSphere**
1. With the SATA drives added, a SATA controller will also be added so you can remove the default SCSI controller.
2. In vSphere, edit the settings for the VM and select the VM Options tab. Make sure under Boot Options “EFI” is selected for Firmware.
3. In the same tab, under Advanced, edit the Configuration Parameters and create two new entries with the following values: - 
```
SMBIOS.noOEMstrings = "TRUE"
smbios.addHostVendor = "FALSE"
```
4. The “uuid.bios” entry in the vmx file will also need to be updated to reflect the MAC address that you chose and previously configured for the VM NIC. In the case of ESXi, SSH into the server (or use WinSCP), navigate to the VM directory and edit the VMX configuration file to make sure the entry matches the MAC address: -
   1. The format is **```uuid.bios = "00 0f ff xx xx xx ee 0f-ac e2 d9 2c a3 b1 63 bb"```** where “xx” is the remaining hexadecimals to your MAC address. So based on the earlier MAC, that line will then be modified to **```uuid.bios = "00 0f ff a1 b2 c3 ee 0f-ac e2 d9 2c a3 b1 63 bb"```**.
5. Use VirtualBox to convert the VDI file to VMDK: -
```
> VBoxManage.exe clonehd lb_boot_stick.vdi lb_boot_stick_converted.vmdk --format VMDK
```
6. Upload the VMDK to the VM folder on the ESXi server, import it and then delete the original file: -
```
> vmkfstools -i lb_boot_stick_converted.vmdk -d thin lb_boot_stick.vmdk && rm lb_boot_stick_converted.vmdk
```
>***Optional*** - adding a serial port is also quite simple and helpful. The only exception is that the serial port is mapped to the IP address of the ESXi host. So, in the VM settings, select “Use Network”, make sure it is “connected” and “connected at power on”, set the direction to “Server” and enter **```telnet://5000```** for the Port URI. Click “Yield on CPU poll”. To connect to it, use a telnet client pointed at the ESXI server IP on port 5000.

7. If you find you cannot connect over Serial, you may need to enable the Firewall rule “VM serial port connected over network”. 

## Wrap-up
When either VM system boots, you will see the kernel logging to the VM console screen. Not all of this is shown because Control4 preferred to send this over the serial and it's the same here. If you configured the VM with a serial port then you should see the entire boot plus the factory scripts (i.e.: creating the raid array, partitions, the filesystem and OS packages installing etc.). It will reboot twice automatically before then arriving at a login prompt.

>***Note***: after first boot, the kernel no longer uses the bootable image file so it can be disconnected and removed. All reboots after this first restart will also have zero output going to the VM console (it will be blank) and instead all logging will go over serial. Again, Control4 did this by design and is fine for most people to be fair but if you needed to see the actual VM console screen, you would instead need to boot off the bootable image file again so that it bypasses the installed kernel that forces a silent console. It’s quite trivial to just leave the serial port configured if you ever needed to reconnect directly (say, if you lose SSH access) and ignore the VM console screen. This is my preference.

A final word of warning. If you later update the OS through the factory tools (Device Image Updater, C4 Toolbox or via Composer) it will rewrite the boot partition and you will lose the VM capability. T 

At this point you can log in using **```c4admin/c4admin```** or look up the IP your DHCP server gave the Director and log in via SSH. You should now have a clean installation of the Director ready to either build up or restore a project on to. Also, depending on which OS you chose to patch and run, you may see some activity being logged as it downloads updates to its own scripts and installed drivers. Going forward, you should be able to use the Director just as if it was running off dedicated hardware.

## Disclaimer
This project has no affiliation with Control4, Snap AV or ADI and was driven by a necessity to ensure my own Control4 environments were running in a perfectly resilient and redundant manner since there is currently no equivalent solution available by the manufacturer. You must have an existing CA10 unit and be an existing customer of Control4 to use the software here. **Please use at your own risk.**
