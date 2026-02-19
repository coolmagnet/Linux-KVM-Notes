# Linux-KVM-Notes

<pre>
How to share directories between host and guests on kvm

The most efficient and modern way to share directories between a KVM host and a guest is
using Virtio-FS. This method provides better performance and local-like file semantics 
compared to older methods like 9p or network shares. 

Step 1: Configure the Host (Virt-Manager) 
Before the guest can access a folder, you must enable shared memory and add the filesystem hardware. 

    Enable Shared Memory:
        Open your VM settings in Virt-Manager (the "i" icon).
        Select Memory on the left and check the box Enable shared memory.
    Add Filesystem Hardware:
        Click Add Hardware > Filesystem.
        Driver: Select virtiofs.
        Source Path: Browse to the directory on your host machine you want to share.
        Target Path: Enter a "mount tag" (this is just a label, e.g., my_shared_folder). 
	Note: This is not a path on the guest yet, just a label.

        Click Finish and start the VM. 

Step 2: Access the Share in the Guest
The steps depend on whether your guest OS is Linux or Windows.
For Linux Guests (e.g., Ubuntu)

    Create a Mount Point: Open a terminal in the guest and create a directory where the share will appear:

    sudo mkdir /mnt/shared

Mount the Directory: Use the mount command with the tag you created in Step 1:

sudo mount -t virtiofs my_shared_folder /mnt/shared

Permanent Mount (Optional): To make the share persist after rebooting, add this line to your guest's 
/etc/fstab:

my_shared_folder /mnt/shared virtiofs rw,noatime,_netdev 0 2 

For Windows Guests

    Install Drivers: You must install WinFSP and the VirtIO-Win drivers inside the Windows guest.
    Enable Service: Open Services (services.msc), find VirtIO-FS Service, and set its Startup type to 
    Automatic and click Start.
    Access: The shared folder should automatically appear as a network drive (usually Z:) in File Explorer. 


================


Perform drag and drop like virtualbox

KVM (specifically when using virt-manager) does not support "drag and drop" file transfers in the same 
seamless, built-in way that VirtualBox does. While VirtualBox uses its proprietary Guest Additions to 
enable this, KVM is a lower-level hypervisor that relies on external tools and different protocols for
guest-host interaction. 

However, you can achieve similar or better functionality using these methods:

    Gnome Boxes: This is a user-friendly frontend for KVM/QEMU that does support drag and drop for
    files by default. If you want a VirtualBox-like experience on Linux without manual configuration,
    this is the best alternative.

    SPICE & spice-vdagent: For standard KVM/virt-manager setups, you must install the SPICE agent inside
    the guest OS. This enables shared clipboards (copy/paste text) and, in some cases, limited file
    transfers depending on the client used.

    Virtio-FS / Shared Folders: This is the "Linux way" of handling large or frequent file transfers.
    It is much faster and more reliable than drag and drop. You can set up a shared directory on your
    host that appears as a drive in the guest.

    Samba/NFS: You can also treat the VM like a separate machine on your network and use standard
    network file sharing. 


================


Installing the SPICE agent

To enable drag-and-drop and shared clipboard in KVM (via
virt-manager), you must install the SPICE agent inside your guest operating system and ensure your
VM hardware is configured correctly. 

1. Hardware Configuration (Host)
Before booting the VM, check these settings in virt-manager (click the lightbulb icon for details): 

    Display Spice: Ensure the "Type" is set to Spice server.
    Video: Ensure the "Model" is set to QXL for best performance.
    Add Channel: Click Add Hardware > Channel. Set "Name" to com.redhat.spice.0 and "Device Type" to
    Spice agent (spicevmc). This creates the communication bridge. 

2. Guest Installation
Follow the steps for your specific guest operating system:
For Linux Guests
Open a terminal inside your VM and run the command for your distribution: 

    Ubuntu / Debian / Kali: sudo apt install spice-vdagent.
    Fedora / CentOS / RHEL: sudo dnf install spice-vdagent.
    Arch Linux: sudo pacman -S spice-vdagent. 

Note: You must reboot the guest or start the service manually using sudo systemctl start spice-vdagentd. 
For Windows Guests

    Boot your Windows VM.
    Download the SPICE Guest Tools Installer from the official Spice-space.org website.
    Run the .exe file. This package includes the QXL video driver, VirtIO drivers, and the SPICE agent.
    Reboot the Windows guest. 

3. Usage
Once installed and rebooted:

    Clipboard: Copying and pasting text between host and guest should now work automatically.
    Drag and Drop: You can drag files from your host file manager and drop them onto the guest desktop.
    Automatic Resolution: Resizing the virt-manager window should now automatically adjust the guest's
    screen resolution. 

Important: If drag-and-drop doesn't work immediately on Linux, ensure you are using X11 instead of Wayland,
as Wayland support for this feature can be inconsistent depending on the distribution. 


================


Snapshots

KVM supports snapshots, but they are handled differently than in VirtualBox. While VirtualBox focuses 
on a user-friendly "tree" of snapshots in its GUI, KVM offers more powerful, professional-grade snapshot
types through virt-manager and the command-line tool virsh. 

1. Internal vs. External Snapshots
KVM distinguishes between where the snapshot data is stored: 

    Internal Snapshots (Standard): These are stored directly inside your .qcow2 disk image file. This is
    the most similar to VirtualBox—you have one file containing both the current state and all your
    historical snapshots. These are easily managed in the virt-manager GUI.

    External Snapshots (Advanced): These create a new, separate "overlay" file. The original disk becomes
    read-only, and all new changes go into the new file. This is safer for live backups and allows you to
    snapshot disks that aren't in qcow2 format (like raw partitions). 

2. Snapshots of Running VMs ("Live" Snapshots)
KVM can capture the full state of a running VM, including its RAM (memory). 

    When you restore such a snapshot, the VM doesn't just "boot up"—it instantly resumes exactly where it
    was, with all applications still open.
    Note: Taking a memory snapshot in KVM may briefly "pause" the VM while the RAM is written to disk. 

3. Key Differences from VirtualBox

    Storage Requirements: To use snapshots in the virt-manager GUI, your VM must use the qcow2 disk format.
    If you chose "raw" during setup, the snapshot button will be greyed out.

    Performance: KVM's snapshot system is designed for enterprise reliability. Unlike VirtualBox, which can
    sometimes slow down as you add more snapshots, KVM's performance remains extremely stable.

    Management: While VirtualBox has a very visual "Snapshot Tree," virt-manager lists them more simply.
    For complex branching or "live" external snapshots, users often use the virsh snapshot-create-as command. 

Warning: Avoid "mixing" internal and external snapshots on the same VM, as it can occasionally lead to data
management confusion or loss. 


================


In virtualbox the host key combo is Right CTRL, what is it for KVM?

virt-manager (the standard KVM GUI), the default "host key" or grab-release combo is:
Left Ctrl + Left Alt
If you are using Gnome Boxes, the combo is:
Ctrl + Alt
Key Differences from VirtualBox:

    Release Behavior: Unlike VirtualBox, which usually requires the host key to let go of your mouse,
    modern KVM setups with the SPICE agent (which you just installed!) often feature "seamless mouse
    integration." This means you shouldn't even need the host key; your cursor will simply move freely
    between the VM and your desktop.

    Entering/Exiting Fullscreen: If you are stuck in a fullscreen view, F11 is the standard toggle for
    virt-manager, but the Ctrl + Alt combo will still release the input focus.

How to Change it (virt-manager)
If you hate that combo and want to change it:

    Open virt-manager.
    Go to Edit > Preferences.
    Click the Console tab.
    Look for Grab keys and define your own shortcut (like Right Ctrl if you're used to the VirtualBox way).


================


Pre-allocate space

    ls -lh: Shows the Logical size. This is the size the file claims to be. If you told KVM to make a 
    40GB disk, ls will show 40GB.

    du -h: Shows the Physical size. This is the actual amount of data currently written to your SSD/HDD.

    df -h (Host): Shows the Remaining Physical Space. If you have a 40GB sparse file that is only using
    5GB of actual data, df will show that you still have the other 35GB available for other things on
    your host.


================


Back Date

Set Date
virt-xml winxp --connect qemu:///system --edit --confirm --qemu-commandline="-rtc base=2006-09-01T00:00:00,clock=vm"
virsh -c qemu:///system dumpxml winxp | grep -A 5 -B 1 qemu:

Rollback
virt-xml winxp --connect qemu:///system --edit --confirm --xml xpath.delete=./qemu:commandline
virsh -c qemu:///system dumpxml winxp | grep -i qemu


================

Disk Clone

When you use virt-clone, it automatically generates a new, unique MAC address for the clone by default
virt-clone --connect qemu:///system -o debian13 -n debian13-clone -f /path/to/debian13-clone.qcow2 
</pre>
