---
title: "vDS to vSS and back again"
date: 2017-07-02
slug: "vds-to-vss-and-back-again"
categories: 
  - "virtualisation"
---

# Overview

I was recently tasked with migrating a selection of ESXi 5.5 hosts into a new vSphere 6.5 environment. These hosts leveraged Fibre Channel HBA's for block storage and 2x10Gbe interfaces for all other traffic types. I assumed that doing a vDS detach and resync was not the correct approach to do this, even though some people reported success doing it this way.  The /r/vmware Reddit community agreed and later I found a [VMware KB article](https://kb.vmware.com/selfservice/microsites/search.do?language=en_US&cmd=displayKC&externalId=1029498) that backs the more widely accepted solution involving moving everything to a vSphere Standard Switch first.

#  Automating the process

There are already several resources on how to do vDS -> vSS migrations but I fancied trying it myself. I used [Virtually Ghetto's script as a foundation for my own](http://www.virtuallyghetto.com/2013/11/automate-reverse-migrating-from-vsphere.html) but wanted to add a few changes that were applicable to my specific environment. These included:

- Populating a vSS dynamically by probing the vDS the host was attached to, including VLAN ID tags
    - Additionally, add a prefix to differentiate between the vSS and vDS portgroups
- Automating the migration of VM port groups from the vDS to a vSS in a way that would result in no downtime.

# Script process

This script performs the migration on a specific host, defined in $vmhost.

1. Connect to vCenter Server
2. Create a vSS on the host called "vSwitch\_Migration"
3. Iterate through the vDS portgroups, recreate on the vSS like-for-like, including VLANID tagging (where appropriate).
4. Acquire list of VMKernel adaptors
5. Move vmnic0 from the vDS to the vSS. At the same time migrate the VMKernel interfaces
6. Iterate through all the VM's on the host, reconfigure port group so it resides in the vSS
7. Once all the VM's have migrated, add the second (and final, in my environment) vmnic to the vSS
8. At this point nothing specific to this host resides on the vDS, therefore remove the vDS from this host

If you plan to run these scripts in your environment, test first in a non-production environment.

\[code language="powershell"\]

Write-Host "Connecting to vCenter Server" -foregroundcolor Green Connect-VIServer -Server "vCenterServer" -User administrator@vsphere.local -Pass "somepassword" | Out-Null

\# Individual ESXi host to migrate from vDS to VSS $vmhost = "192.168.1.20" Write-Host "Host selected: " $vmhost -foregroundcolor Green

\# Create a new vSS on the host $vss\_name = New-VirtualSwitch -VMHost $vmhost -Name vSwitch\_Migration Write-Host "Created new vSS on host" $vmhost "named" "vSwitch\_Migration" -foregroundcolor Green

#VDS to migrate from $vds\_name = "MyvDS" $vds = Get-VDSwitch -Name $vds\_name

#Probe the VDS, get port groups and re-create on VSS $vds\_portgroups = Get-VDPortGroup -VDSwitch $vds\_name foreach ($vds\_portgroup in $vds\_portgroups) { if(\[string\]::IsNullOrEmpty($vds\_portgroup.vlanconfiguration.vlanid)) { Write-Host "No VLAN Config for" $vds\_portgroup.name "found" -foregroundcolor Green $PortgroupName = $vds\_portgroup.Name New-VirtualPortGroup -virtualSwitch $vss\_name -name "VSS\_$PortgroupName" | Out-Null }

else

{ Write-Host "VLAN config present for" $vds\_portgroup.name -foregroundcolor Green $PortgroupName = $vds\_portgroup.Name New-VirtualPortGroup -virtualSwitch $vss\_name -name "VSS\_$PortgroupName" -VLanId $vds\_portgroup.vlanconfiguration.vlanid | Out-Null } }

#Create a list of VMKernel adapters $management\_vmkernel = Get-VMHostNetworkAdapter -VMHost $vmhost -Name "vmk0" $vmotion1\_vmkernel = Get-VMHostNetworkAdapter -VMHost $vmhost -Name "vmk1" $vmotion2\_vmkernel = Get-VMHostNetworkAdapter -VMHost $vmhost -Name "vmk2" $vmkernel\_list = @($management\_vmkernel,$vmotion1\_vmkernel,$vmotion2\_vmkernel)

#Create mapping for VMKernel -> vss Port Group $management\_vmkernel\_portgroup = Get-VirtualPortGroup -name "VSS\_Mgmt" -Host $vmhost $vmotion1\_vmkernel\_portgroup = Get-VirtualPortGroup -name "VSS\_vMotion1" -Host $vmhost $vmotion2\_vmkernel\_portgroup = Get-VirtualPortGroup -name "VSS\_vMotion2" -Host $vmhost $pg\_array = @($management\_vmkernel\_portgroup,$vmotion1\_vmkernel\_portgroup,$vmotion2\_vmkernel\_portgroup)

#Move 1 uplink to the vss, also move over vmkernel interfaces Write-Host "Moving vmnic0 from the vDS to VSS including vmkernel interfaces" -foregroundcolor Green Add-VirtualSwitchPhysicalNetworkAdapter -VMHostPhysicalNic (Get-VMHostNetworkAdapter -Physical -Name "vmnic0" -VMHost $vmhost) -VirtualSwitch $vss\_name -VMHostVirtualNic $vmkernel\_list -VirtualNicPortgroup $pg\_array -Confirm:$false

#Moving VM's from vDS to VSS $vmlist = Get-VM | Where-Object {$\_.VMHost.name -eq $vmhost}

foreach ($vm in $vmlist) { #VM's may have more that one nic $vmniclist = Get-NetworkAdapter -vm $vm foreach ($vmnic in $vmniclist) { $newportgroup = "VSS\_" + $vmnic.NetworkName Write-Host "Changing port group for" $vm.name "from" $vmnic.NetworkName "to " $newportgroup -foregroundcolor Green Set-NetworkAdapter -NetworkAdapter $vmnic -NetworkName $newportgroup -Confirm:$false | Out-Null } }

#Moving additional vmnic to vss Write-Host "All VM's migrated, adding second vmnic to vss" -foregroundcolor Green Add-VirtualSwitchPhysicalNetworkAdapter -VMHostPhysicalNic (Get-VMHostNetworkAdapter -Physical -Name "vmnic1" -VMHost $vmhost) -VirtualSwitch $vss\_name -Confirm:$false

#Tidyup - Remove DVS from this host Write-Host "Removing host from vDS" -foregroundcolor Green $vds | Remove-VDSwitchVMHost -VMHost $vmhost -Confirm:$false\[/code\]

 

 

# The reverse

Although vSphere has some handy tools to migrate hosts, portgroups and networking to a vDS, scripting the reverse didn't require many changes to the original script:

\[code language="powershell"\]

Write-Host "Connecting to vCenter Server" -foregroundcolor Green Connect-VIServer -Server "vCenterServer" -User administrator@vsphere.local -Pass "somepassword" | Out-Null

\# Individual ESXi host to migrate from vDS to VSS $vmhost = "192.168.1.20" Write-Host "Host selected: " $vmhost -foregroundcolor Green

#VDS to migrate to $vds\_name = "MyvDS" $vds = Get-VDSwitch -Name $vds\_name

#Vss to migrate from $vss\_name = "vSwitch\_Migration" $vss = Get-VirtualSwitch -Name $vss\_name -VMHost $vmhost

#Add host to vDS but don't add uplinks yet Write-Host "Adding host to vDS without uplinks" -foregroundcolor Green Add-VDSwitchVMHost -VMHost $vmhost -VDSwitch $vds

#Create a list of VMKernel adaptors $management\_vmkernel = Get-VMHostNetworkAdapter -VMHost $vmhost -Name "vmk0" $vmotion1\_vmkernel = Get-VMHostNetworkAdapter -VMHost $vmhost -Name "vmk1" $vmotion2\_vmkernel = Get-VMHostNetworkAdapter -VMHost $vmhost -Name "vmk2" $vmkernel\_list = @($management\_vmkernel,$vmotion1\_vmkernel,$vmotion2\_vmkernel)

#Create mapping for VMKernel -> vds Port Group $management\_vmkernel\_portgroup = Get-VDPortgroup -name "Mgmt" -VDSwitch $vds\_name $vmotion1\_vmkernel\_portgroup = Get-VDPortgroup -name "vMotion0" -VDSwitch $vds\_name $vmotion2\_vmkernel\_portgroup = Get-VDPortgroup -name "vMotion1" -VDSwitch $vds\_name $vmkernel\_portgroup\_list = @($management\_vmkernel\_portgroup,$vmotion1\_vmkernel\_portgroup,$vmotion2\_vmkernel\_portgroup)

#Move 1 uplink to the vDS, also move over vmkernel interfaces Write-Host "Moving vmnic0 from the vSS to vDS including vmkernel interfaces" -foregroundcolor Green Add-VDSwitchPhysicalNetworkAdapter -VMHostPhysicalNic (Get-VMHostNetworkAdapter -Physical -Name "vmnic0" -VMHost $vmhost) -DistributedSwitch $vds\_name -VMHostVirtualNic $vmkernel\_list -VirtualNicPortgroup $vmkernel\_portgroup\_list -Confirm:$false

#Moving VM's from VSS to vDS $vmlist = Get-VM | Where-Object {$\_.VMHost.name -eq $vmhost}

foreach ($vm in $vmlist) { #VM's may have more that one nic $vmniclist = Get-NetworkAdapter -vm $vm foreach ($vmnic in $vmniclist) { $newportgroup = $vmnic.NetworkName.Replace("VSS\_","") Write-Host "Changing port group for" $vm.name "from" $vmnic.NetworkName "to " $newportgroup -foregroundcolor Green Set-NetworkAdapter -NetworkAdapter $vmnic -Portgroup $newportgroup -Confirm:$false | Out-Null } }

#Moving additional vmnic to vds Write-Host "All VM's migrated, adding second vmnic to vDS" -foregroundcolor Green Add-VDSwitchPhysicalNetworkAdapter -VMHostPhysicalNic (Get-VMHostNetworkAdapter -Physical -Name "vmnic1" -VMHost $vmhost) -DistributedSwitch $vds\_name -Confirm:$false

#Tidyup - Remove vSS from this host Write-Host "Removing VSS from host" -foregroundcolor Green Remove-VirtualSwitch -VirtualSwitch $vss -Confirm:$false\[/code\]
