# Proxmox-GPU-Passthrough

Install latest version of PVE, use this program to make the iso bootable https://rufus.ie/en/

Download iso here https://www.proxmox.com/en/

Install graphics card in machine.

Having an SSD boot disk is HIGHLY recommended for this, I can't stress this enough.

Install PVE and set aside, the rest should be done from another system on the same subnet, if you don't know what a subnet is, move on please, cause this is not for you :-)

Foot notes-I had an extra switch and another laptop to do this which made this so much easier. The laptop had Ubuntu Studio installed, no specific reason for this, but you do need a system with Remote Viewer installed to use this method, I used a switch to put both systems on the same "network" this is equivalent to plugging your new server directly into your router and using another laptop on wifi. There are other ways to do this, but this is my way.

open your web gui -> https://ip:8006 Open shell on PVE

Follow the instruction for GRUB if you did the GRUB2 install or for Systemd for the Ventoy normal install

Booting Proxmox using Ventoy Grub2 can give Ramdisk problems, it's better to use normal boot mode

Initial GRUB -----------------------

nano /etc/default/grub Modify this line GRUB_CMDLINE_LINUX_DEFAULT=....

for INTEL => GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on"

for AMD => GRUB_CMDLINE_LINUX_DEFAULT="quiet amd_iommu=on"

Save file

Run update-grub to save boot configuration.

Systemd ------------------------

for systemd-boot you need to use /etc/kernel/cmdline - however all parameters need to go into one single line! 

nano /etc/kernel/cmdline 

add the following to the end of the line: 
* for INTEL => intel_iommu=on
* for AMD => amd_iommu=on

save file

run 'proxmox-boot-tool refresh' to save the configuration

VFIO Modules ---------------------------------------------------------

nano /etc/modules

Just add this on separate lines:
* vfio
* vfio_iommu_type1
* vfio_pci
* vfio_virqfd

Save file and close

Commands for pipe just copy and past these in the shell and it will create the file where you need it.

echo "options vfio_iommu_type1 allow_unsafe_interrupts=1" > /etc/modprobe.d/iommu_unsafe_interrupts.conf

echo "options kvm ignore_msrs=1" > /etc/modprobe.d/kvm.conf

echo "blacklist nouveau" >> /etc/modprobe.d/blacklist.conf

echo "blacklist nvidia" >> /etc/modprobe.d/blacklist.conf

PCIe Passthrough----------------------------------------------------

List Devices => lspci -v

Locate your cards hardware id

Identify => lspci -n -s XX:XX

you should get 2 put em in this command in the shell

echo "options vfio-pci ids=XXXX:XXXX,XXXX:XXXX disable_vga=1"> /etc/modprobe.d/vfio.conf

update-initramfs -u ---->REBOOT<-----

Windows VM setup -----------------------------------------------------

Use latest distro https://www.microsoft.com/en-us/software-download/windows10 Setup machine with OVMF BIOS and EFI Disk, set machine type to q35

Assign 4 cores and 8GB to machine ! Minimum !

Set the CD/DVD to use ISO

Add the PCI device, Choose All functions, ROM-Bar and PCI-E, leave PRIMARY unchecked for now.

Connect the GPU to a monitor

Go back into the shell and nano /etc/pve/qemu-server/XXX.conf
XXX stands for the VM id

add CPU: host,hidden=1 to the top, PVE will move this when you boot up the machine, save, exit

Note - MANY functions will not correctly in 64-bit applications if you do not set the flag.

Change "Display" to Spice, this will force the VM to download the script when you activate the console and you can control the VM through Remote Viewer. WAY Easier then setting up RDP. to use RDP you have to enable it in System settings and turn off the firewall on the VM. If you are setting this VM up from another windows machine and not a linux, use RDP method.

Start your VM

Have the remote-viewer open, sometimes you have to be quick to catch the "Press any key to boot from" in order to get into windows.

!!! If you VM doesn't see the hard drive you have to install VirtIO-Guest tools, download here linux-kvm.org/page/WindowsGuestDrivers/Download_Drivers you need the iso. Shut down the VM, if you can't shut it down, rm the lock file for that VM. Add a SATA drive to the VM and add virtio ISO, when you get prompted to add drivers your looking for virtioscsi=>AMD64, should say Red Hat, once it is installed you should see your drive!!!

Once your VM is up you need to install the virtio guest tools from the iso you downloaded.

I like to have the driver for my card downloaded to a USB and passed through so I can install the one i want, if you just let it download the driver, this WILL work, I've done it both ways, but this means you need a bridged connection to the internet.

Your secondary monitor SHOULD come up, once it does you can shut the VM down and pass through a keyboard and mouse and change "Display" to None. Sound is automatically passed through with the card through HDMI since you should have added the IDs vfio.conf

Don't forget to turn on "boot on startup" in options on the PVE GUI, otherwise you will have to ssh to it OR you can use a cellphone on the same network and turn it on that way. Auto boot is WAY easier.
