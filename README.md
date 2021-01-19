# bmc_discovery_vagrant

In this tutorial you will find an answer how to run `BMC Discovery` appliance on `Hyper-V` and how to create `Vagrant` box with `Packer`.

## Introduction - What is BMC Discovery (ADDM)?
BMC Discovery, formerly ADDM, is a digital enterprise management solution that automates asset discovery and application dependency mapping to build a holistic view of all your data center assets and the relationships between them.

Product details:

    Agentless
    Lightweight
    Scalable
    Secure
    Seamless CMDB Integration
    Reference Library
    Discovery For Storage (optional add-on)
    Available on-premise, private, or public cloud
    Powerful security operations
    Multi-cloud, software, hardware, network, and storage mapping
    Reduce storage outages
    Verify changes
    Prioritize tickets based on impact on business
    Reduce MTTR with correct data
    Identify unused infrastructure to save licenses and maintenance costs

Source: https://www.flycastpartners.com/bmc-discovery/

## Prerequisites

I assume that you have the following:

* Windows 10 (x64), with partition D:\ with 15GB of free space
* Hyper-V
* PSVersion >= `5.1.18362.752`
* HashiCorp Packer >= `1.6.2` (https://www.packer.io/downloads)
* HashiCorp Vagrant >= `2.2.10` (https://www.vagrantup.com/downloads)
* qemu-img >= `2.3.0` (https://cloudbase.it/qemu-img-windows/)
* BMC Discovery image (in this tutorial I will use `BMC Discovery Community Edition` but you can use Full Edition downloaded from `Electronic Product Distribution` (EDP) as well)

## PowerShell Script
# Stage 1: prepare vhd image
```powershell
# set variables
# If you go to https://www.bmc.com/forms/bmc-helix-discovery-trial.html it has a "START YOUR TRIAL" button.
$user = 'PASS'
$pass = 'USER'

$bmcDirectory = 'bmc_discovery'
$vhdDirectory = 'vhd'
$vmDirectory = 'vmdata'

$bmcFileName = 'BMC_Discovery_VA_ga_Community.zip'
$qemuFileName = 'qemu-img-win-x64-2_3_0.zip'

$bmcSource = "ftp://${user}:${pass}@epddownload.bmc.com/$bmcFileName"
$qemuSource = 'https://cloudbase.it/downloads'

# part 1: download BMC Discovery Community Edition
# change path
Set-Location -Path D:\

# make mandatory path
if (!(Test-Path $bmcDirectory -PathType Container)) {
    New-Item -Name $bmcDirectory -ItemType Directory
}

# change path
Set-Location -Path D:\$bmcDirectory

# download BMC Discovery Community Edition
(New-Object System.Net.WebClient).DownloadFile($bmcSource, "D:\$bmcDirectory\$bmcFileName")

# expand archive
Expand-Archive -Path .\$bmcFileName -DestinationPath .\BMC_Discovery_VA_ga_Community

# list files
Get-ChildItem .\BMC_Discovery_VA_ga_Community\ADDM*\*

<#
PS D:\bmc_discovery> Get-ChildItem .\BMC_Discovery_VA_ga_Community\ADDM*\*


    Directory: D:\bmc_discovery\BMC_Discovery_VA_ga_Community\ADDM_VA_64_12.0.0.1_809805_ga_Community


Mode                LastWriteTime         Length Name                                                                                                                                         
----                -------------         ------ ----                                                                                                                                         
-a----       13.05.2020     22:15     1945465856 ADDM_VA_64_12.0.0.1_809805_ga_Community-disk-0.vmdk                                                                                          
-a----       13.05.2020     22:15           8155 ADDM_VA_64_12.0.0.1_809805_ga_Community.ovf                                                                                                  
-a----       13.05.2020     22:16            228 ADDM_VA_64_12.0.0.1_809805_ga_Community.sha256           
#>

# part 2: download qemu-img-win-x64-2_3_0.zip
# change path
Set-Location -Path D:\$bmcDirectory

# download qemu
(New-Object System.Net.WebClient).DownloadFile("$qemuSource/$qemuFileName", "D:\$bmcDirectory\$qemuFileName")

# expand archive
Expand-Archive -Path .\$qemuFileName -DestinationPath .\qemu-img-win-x64-2_3_0

# list files
Get-ChildItem .\qemu*\*

<#
PS D:\bmc_discovery> Get-ChildItem .\qemu*\*


    Directory: D:\bmc_discovery\qemu-img-win-x64-2_3_0


Mode                LastWriteTime         Length Name                                                                                                                                         
----                -------------         ------ ----                                                                                                                                         
-a----       18.09.2013     13:35          86528 libgcc_s_sjlj-1.dll                                                                                                                          
-a----       30.03.2015     15:09        2584872 libglib-2.0-0.dll                                                                                                                            
-a----       30.03.2015     15:10          79707 libgthread-2.0-0.dll                                                                                                                         
-a----       30.03.2015     03:38        1475928 libiconv-2.dll                                                                                                                               
-a----       30.03.2015     13:59         464017 libintl-8.dll                                                                                                                                
-a----       18.09.2013     13:35          18944 libssp-0.dll                                                                                                                                 
-a----       16.06.2015     21:37        5615492 qemu-img.exe       
#>

# part 3: convert vmdk image to vhd image
# change path
Set-Location -Path D:\$bmcDirectory

# make mandatory path
if (!(Test-Path $vhdDirectory -PathType Container)) {
    New-Item -Name $vhdDirectory -ItemType Directory
}

# run qemu and convert vmdk to vhd
.\qemu-*\qemu-img.exe convert (Get-ChildItem -Path *.vmdk -Recurse | select *).FullName -O vpc -o subformat=dynamic .\vhd\bmc_discovery.vhd

# list files
Get-ChildItem $vhdDirectory

<#
PS D:\bmc_discovery> Get-ChildItem $vhdDirectory


    Directory: D:\bmc_discovery\vhd


Mode                LastWriteTime         Length Name                                                                                                                                         
----                -------------         ------ ----                                                                                                                                         
-a----       20.09.2020     22:39     4275113472 bmc_discovery.vhd          
#>

# part 4: create new vm then start it
# change path
Set-Location -Path D:\$bmcDirectory

# make mandatory path
if (!(Test-Path $vmDirectory -PathType Container)) {
    New-Item -Name $vmDirectory -ItemType Directory
}

# create new vm
New-VM -Name bmc_discovery -MemoryStartupBytes 1GB -BootDevice VHD -VHDPath .\vhd\bmc_discovery.vhd -Path $vmDirectory -Generation 1 -Switch 'Default Switch'

# start vm
Start-VM -Name bmc_discovery
```

# Stage 2: Change default password for `tideway` and `root` users
Login to VM via SSH and change default passwords
```
Username: tideway
Initial Password: tidewayuser

Username: root
Initial Password: tidewayroot

Username: system
Initial Password: system
```

In that case I will set them like this:
```
user: tideway
pass: V4gr4nt!t1dew4y

user: root
pass: V4gr4nt!r00t

user: system
pass: V4gr4nt!system
```

You can find more information here (https://communities.bmc.com/message/492774#492774) or here (https://docs.bmc.com/docs/discovery/110/logging-in-to-the-appliance-command-line-625695411.html).

# Stage 3: Download and install `Hyper-V VM Integration Services`
This is an optional step and if you don't want to use `Packer` you can go directly to `Stage 4: Option 2`
```bash
cd /tmp
wget https://download.microsoft.com/download/6/8/F/68FE11B8-FAA4-4F8D-8C7D-74DA7F2CFC8C/lis-rpms-4.3.5.x86_64.tar.gz
tar -xzf lis-rpms-4.3.5.x86_64.tar.gz
cd LISISO/RPMS77
rpm -ivh --nodeps kmod-microsoft-hyper-v-4.3.5-20200304.x86_64.rpm microsoft-hyper-v-4.3.5-20200304.x86_64.rpm
```

# Stage 4: Create Vagrant bmc_discovery box
## Option 1: Packer
Create `build.json` with following content:
```json
{
    "variables": {
        "hyperv_source_vm": "bmc_discovery",
        "hyperv_switch_name": "Default Switch",
        "hyperv_target_name": "bmc_packer_{{ timestamp }}",
        "ssh_password": "V4gr4nt!t1dew4y"
    },
    "builders": [
        {
            "clone_from_vm_name": "{{ user `hyperv_source_vm` }}",
            "ssh_password": "{{ user `ssh_password` }}",
            "ssh_username": "tideway",
            "switch_name": "{{ user `hyperv_switch_name` }}",
            "type": "hyperv-vmcx",
            "vm_name": "{{ user `hyperv_target_name` }}"
        }
    ],
    "provisioners": [
        {
            "inline": [
                "sudo -S shutdown -P now || true"
            ],
            "type": "shell"
        }
    ],
    "post-processors": [
        {
            "type": "vagrant"
        }
    ]
}
```

then run following commands:
```powershell
.\packer.exe build -force .\build.json
```

you should see this:
```
Warning: Warning when preparing build: "hyperv-vmcx"                                                                                                                                                                            
Hyper-V might fail to create a VM if there is not enough free memory in the
system.

Warning: Warning when preparing build: "hyperv-vmcx"

A shutdown_command was not specified. Without a shutdown command, Packer
will forcibly halt the virtual machine, which may result in data loss.


hyperv-vmcx: output will be in this color.

==> hyperv-vmcx: Creating build directory...
==> hyperv-vmcx: Creating switch 'Default Switch' if required...
==> hyperv-vmcx:     switch 'Default Switch' already exists. Will not delete on cleanup...
==> hyperv-vmcx: Cloning virtual machine...
==> hyperv-vmcx: Enabling Integration Service...
==> hyperv-vmcx: Skipping mounting Integration Services Setup Disk...
==> hyperv-vmcx: Mounting secondary DVD images...
==> hyperv-vmcx: Configuring vlan...
==> hyperv-vmcx: Determine Host IP for HyperV machine...
==> hyperv-vmcx: Host IP for the HyperV machine: 192.168.96.129
==> hyperv-vmcx: Attempting to connect with vmconnect...
==> hyperv-vmcx: Starting the virtual machine...
==> hyperv-vmcx: Waiting 10s for boot...
==> hyperv-vmcx: Typing the boot command...
==> hyperv-vmcx: Waiting for SSH to become available...
==> hyperv-vmcx: Connected to SSH!
==> hyperv-vmcx: Provisioning with shell script: C:\Users\mjaromi\AppData\Local\Temp\packer-shell427449495
==> hyperv-vmcx:
==> hyperv-vmcx: We trust you have received the usual lecture from the local System
==> hyperv-vmcx: Administrator. It usually boils down to these three things:
==> hyperv-vmcx:
==> hyperv-vmcx:     #2) Think before you type.
==> hyperv-vmcx:     #3) With great power comes great responsibility.
==> hyperv-vmcx:
==> hyperv-vmcx: [sudo] password for tideway:
==> hyperv-vmcx: sudo: no password was provided
==> hyperv-vmcx: Forcibly halting virtual machine...
==> hyperv-vmcx: Waiting for vm to be powered down...
==> hyperv-vmcx: Unmount/delete secondary dvd drives...
==> hyperv-vmcx: Unmount/delete Integration Services dvd drive...
==> hyperv-vmcx: Unmount/delete os dvd drive...
==> hyperv-vmcx: Unmount/delete floppy drive (Run)...
==> hyperv-vmcx: Compacting disks...
    hyperv-vmcx: Compacting disk: bmc_discovery.vhd
    hyperv-vmcx: Disk size is unchanged
==> hyperv-vmcx: Disconnecting from vmconnect...
==> hyperv-vmcx: Unregistering and deleting virtual machine...
==> hyperv-vmcx: Deleting build directory...
==> hyperv-vmcx: Running post-processor: vagrant
==> hyperv-vmcx (vagrant): Creating Vagrant box for 'hyperv' provider
    hyperv-vmcx (vagrant): Copying: output-hyperv-vmcx\Virtual Hard Disks\bmc_discovery.vhd
    hyperv-vmcx (vagrant): Copied output-hyperv-vmcx\Virtual Hard Disks\bmc_discovery.vhd to C:\Users\mjaromi\AppData\Local\Temp\packer206464701\Virtual Hard Disks\bmc_discovery.vhd
    hyperv-vmcx (vagrant): Copying: output-hyperv-vmcx\Virtual Machines\E680DA54-C5E9-4F4D-862D-C5F7C927D75F.VMRS
    hyperv-vmcx (vagrant): Copied output-hyperv-vmcx\Virtual Machines\E680DA54-C5E9-4F4D-862D-C5F7C927D75F.VMRS to C:\Users\mjaromi\AppData\Local\Temp\packer206464701\Virtual Machines\E680DA54-C5E9-4F4D-862D-C5F7C927D75F.VMRS
    hyperv-vmcx (vagrant): Copying: output-hyperv-vmcx\Virtual Machines\E680DA54-C5E9-4F4D-862D-C5F7C927D75F.vmcx
    hyperv-vmcx (vagrant): Copied output-hyperv-vmcx\Virtual Machines\E680DA54-C5E9-4F4D-862D-C5F7C927D75F.vmcx to C:\Users\mjaromi\AppData\Local\Temp\packer206464701\Virtual Machines\E680DA54-C5E9-4F4D-862D-C5F7C927D75F.vmcx
    hyperv-vmcx (vagrant): Copying: output-hyperv-vmcx\Virtual Machines\E680DA54-C5E9-4F4D-862D-C5F7C927D75F.vmgs
    hyperv-vmcx (vagrant): Copied output-hyperv-vmcx\Virtual Machines\E680DA54-C5E9-4F4D-862D-C5F7C927D75F.vmgs to C:\Users\mjaromi\AppData\Local\Temp\packer206464701\Virtual Machines\E680DA54-C5E9-4F4D-862D-C5F7C927D75F.vmgs
    hyperv-vmcx (vagrant): Copying: output-hyperv-vmcx\Virtual Machines\box.xml
    hyperv-vmcx (vagrant): Copied output-hyperv-vmcx\Virtual Machines\box.xml to C:\Users\mjaromi\AppData\Local\Temp\packer206464701\Virtual Machines\box.xml
    hyperv-vmcx (vagrant): Compressing: Vagrantfile
    hyperv-vmcx (vagrant): Compressing: Virtual Hard Disks\bmc_discovery.vhd
    hyperv-vmcx (vagrant): Compressing: Virtual Machines\E680DA54-C5E9-4F4D-862D-C5F7C927D75F.VMRS
    hyperv-vmcx (vagrant): Compressing: Virtual Machines\E680DA54-C5E9-4F4D-862D-C5F7C927D75F.vmcx
    hyperv-vmcx (vagrant): Compressing: Virtual Machines\E680DA54-C5E9-4F4D-862D-C5F7C927D75F.vmgs
    hyperv-vmcx (vagrant): Compressing: Virtual Machines\box.xml
    hyperv-vmcx (vagrant): Compressing: metadata.json
Build 'hyperv-vmcx' finished after 5 minutes 42 seconds.

==> Wait completed after 5 minutes 42 seconds

==> Builds finished. The artifacts of successful builds are:
--> hyperv-vmcx: 'hyperv' provider box: packer_hyperv-vmcx_hyperv.box
```

After that you can add `packer_hyperv-vmcx_hyperv.box` to the Vagrant:
```
vagrant box add bmc_discovery .\packer_hyperv-vmcx_hyperv.box
```

## Option 2: PowerShell + tar
```powershell
$bmcDirectory = 'bmc_discovery'
$boxDirectory = 'bmc_discovery_box'

# change path
Set-Location -Path D:\$bmcDirectory

# make mandatory path
if (!(Test-Path $boxDirectory -PathType Container)) {
    New-Item -Name $boxDirectory -ItemType Directory
}

# remove VM snapshot
Remove-VMSnapshot -VMName bmc_discovery

# stop VM
Stop-VM -Name bmc_discovery -Force

# export VM
Export-VM -Name bmc_discovery -Path $boxDirectory

# change path
Set-Location -Path $boxDirectory/bmc_discovery

# remove Shapshots directory
Remove-Item .\Snapshots -Force

# create metadata.json file
[IO.File]::WriteAllLines("D:\$bmcDirectory\bmc_discovery_box\bmc_discovery\metadata.json", '{"provider": "hyperv"}')

# create Vagrant box
tar czf ../bmc_discovery.box ./*

# go one directory up
cd ..

# add Vagrant box
vagrant box add bmc_discovery .\bmc_discovery.box

<#
PS D:\bmc_discovery\bmc_discovery_box> vagrant box add bmc_discovery .\bmc_discovery.box
==> box: Box file was not detected as metadata. Adding it directly...
==> box: Adding box 'bmc_discovery' (v0) for provider:
    box: Unpacking necessary files from: file://D:/bmc_discovery/bmc_discovery_box/bmc_discovery.box
    box:
==> box: Successfully added box 'bmc_discovery' (v0) for 'hyperv'!
#>

Remove-VM -Name bmc_discovery -Force
```

# Stage 5: use bmc_discovery Vagrant box to run BMC Discovery
Create `Vagrantfile` with following content:
```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

VAGRANT_BOX = 'bmc_discovery'
VAGRANTFILE_API_VERSION = '2'

BMC_DISCOVERY_INSTANCES = 2
BMC_DISCOVERY_USERNAME = 'tideway'
BMC_DISCOVERY_PASSWORD = 'V4gr4nt!t1dew4y'

VM_NAME_PREFIX = 'bmc_discovery'
VM_MIN_MEMORY = 4096
VM_MAX_MEMORY = 4096

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.box = VAGRANT_BOX
  config.vm.provider 'hyperv'
  config.vm.network 'private_network', bridge: 'Default Switch'
  config.vm.synced_folder '.', '/vagrant', disabled: true
  
  config.ssh.shell = 'sh'
  config.ssh.username = BMC_DISCOVERY_USERNAME
  config.ssh.password = BMC_DISCOVERY_PASSWORD

  config.vm.provider :hyperv do |h|
    h.vmname = "#{VM_NAME_PREFIX}-1"
    h.memory = VM_MIN_MEMORY
    h.maxmemory = VM_MAX_MEMORY
  end

# you can comment above and uncomment below part and provision multiple appliances
#  (1..BMC_DISCOVERY_INSTANCES).each do |i|
#    config.vm.define "#{VM_NAME_PREFIX}-#{i}" do end
#  end
end
```

then run following commands:
```powershell
vagrant up
vagrant ssh
```

If you see this:
```
The following SSH command responded with a non-zero exit status.
Vagrant assumes that this means the command failed!

sed -i '/#VAGRANT-BEGIN/,/#VAGRANT-END/d' /etc/fstab
```
it does mean that Vagrant was not able to modify `/etc/fstab` which is not writable when you are logged as a `tideway` user. You can ignore it. If you don't want to see this message again you can simply edit `plugins/guests/linux/cap/persist_mount_shared_folder.rb` Vagrant file and add an additional check for `/etc/fstab` (see: https://github.com/mjaromi/vagrant/commit/80f85a19002b8be99ecfb552fa688a06c961d83f), but as I said it is not necessary.


output:
```
PS D:\bmc_discovery\bmc_discovery_box> vagrant up
Bringing machine 'default' up with 'hyperv' provider...
==> default: Verifying Hyper-V is enabled...
==> default: Verifying Hyper-V is accessible...
==> default: Importing a Hyper-V instance
    default: Creating and registering the VM...
    default: Successfully imported VM
    default: Configuring the VM...
==> default: Starting the machine...
==> default: Waiting for the machine to report its IP address...
    default: Timeout: 120 seconds
    default: IP: 192.168.96.140
==> default: Waiting for machine to boot. This may take a few minutes...
    default: SSH address: 192.168.96.140:22
    default: SSH username: tideway
    default: SSH auth method: password
    default:
    default: Inserting generated public key within guest...
    default: Removing insecure key from the guest if it's present...
    default: Key inserted! Disconnecting and reconnecting using new SSH key...
==> default: Machine booted and ready!
PS D:\bmc_discovery\bmc_discovery_box> vagrant ssh
Last login: Mon Sep 21 22:28:01 2020

The BMC Discovery appliance is an integrated software stack consisting of
everything found on the appliance, including the operating system,
BMC Discovery application software, and system configuration. It is not a
general purpose system and should not be treated as such. If the operating
system is customized, it is technically out of support. We reserve the right
to withdraw support, and make no guarantees that future upgrades will
be compatible or maintain any of those customizations. We may ask for
customizations to be removed for testing purposes. In practice, we will try
and provide support for issues that are unrelated to the OS layer.

Documentation regarding supported configuration changes and additional software
that can be installed on the appliance can be found at:
  https://docs.bmc.com/docs/display/ADDM/Documentation+Home

For additional information, please contact BMC Customer Support.
[tideway@localhost ~]$ su
Password:
[root@localhost tideway]#
```
