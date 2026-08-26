---
title: "Evaluating Harvester in vSphere"
date: 2021-12-20
slug: "evaluating-harvester-in-vsphere"
categories: 
  - "cloud"
  - "kubernetes"
  - "microservices"
  - "virtualisation"
---

**Disclaimer - The use of nested virtualisation is not a supported topology**

[Harvester](https://docs.harvesterhci.io) is an open-source HCI solution aimed at managing Virtual Machines, similar to vSphere and Nutanix, with key differences including (but not limited to):

- Fully Open Source
- Leveraging Kubernetes-native technologies
- Integration with Rancher

Testing/evaluating any hyperconverged solution can be difficult - It usually requires having dedicated hardware as these solutions are designed to work directly on bare metal. However, we can circumvent this by leveraging **_nested virtualisation_** \- something which may be familiar with a lot of homelabbers (myself included) - which involves using an existing virtualisation solution provision workloads that also leverage virtualisation technology.

## Step 1 - Planning

To mimic what a production-like system may look like, two NICs will be leveraged - one that facilitates management traffic, and the other for Virtual Machine traffic, as depicted below

![](images/harvester.drawio.png)

`MGMT network` and `VM Network` will manifest as VDS Port groups.

Also, [download and make available the latest ISO for harvester](https://github.com/harvester/harvester/releases)

## Step 2 - Create vDS Port Groups

It is highly recommended to create new Distributed Port groups for this exercise, mainly because of the configuration we will be applying in the next step.

Create a new vDS Port Group:

![](images/image-61c095c1f3245.png)

Give the port group a name, such as `harvester-mgmt`

![](images/image-61c096108470e.png)

Adjust any configuration (ie VLAN ID) to match your environment (if required). Or accept the defaults:

![](images/image-61c09688e5f8e.png)

Repeat this process to create the `harvester-vm` Port group. We should now have two port groups:

- harvester-mgmt
- harvester-vm

## Step 3 - Enable MAC learning on Port groups \[Critical\]

[William Lam has an excellent post on how to accomplish this.](https://williamlam.com/2018/04/native-mac-learning-in-vsphere-6-7-removes-the-need-for-promiscuous-mode-for-nested-esxi.html) This is required for Harvester (or any hypervisor) to function correctly when operating in a nested environment.

```
Set-MacLearn -DVPortgroupName @("harvester-mgmt") -EnableMacLearn $true -EnablePromiscuous $false -EnableForgedTransmit $true -EnableMacChange $false

Set-MacLearn -DVPortgroupName @("harvester-vm") -EnableMacLearn $true -EnablePromiscuous $false -EnableForgedTransmit $true -EnableMacChange $false
```

## Step 4 - Creating a Harvester VM

Our Harvester VM will operate like any other VM, with some important differences. In vSphere, go through the standard VM creation wizard to specify the Host/Datastore options. When presented with the OS type, select `Other Linux (64 bit)`.

![](images/image-61c09911ac6eb.png)

When customising the hardware, select `Expose hardware assisted virtualization to the guest OS` - This is crucial, as without this selected Harvester will not install.

![](images/image-61c09970f32d9.png)

Add an additional network card so that our VM leverages both previously created port groups:

![](images/image-61c09a4c8a1be.png)

And finally, mount the Harvester ISO image.

## Step 4 - Install Harvester

Power on the VM and providing the ISO is mounted and connected, you should be presented with the install screen. As this is the first node, select `create a new Harvester Cluster`

![](images/image-61c09cf662462.png)

Select the Install target and optional MBR partitioning

![](images/image-61c09d40be993.png)

Configure the hostname, management nic and IP assignment options.

![](images/image-61c09daceb78d.png)

Configure the DNS config:

![](images/image-61c09e5dca092.png)

Configure the Harvester VIP. This is what we will use to access the Web UI. This can also be obtained via DHCP if desired.

![](images/image-61c09e9c403c8.png)

Configure the cluster token, this is required if you want to add more nodes later on.

![](images/image-61c09ec566774.png)

Configure the local Password:

![](images/image-61c09eeee9411.png)

Configure the NTP server Address:

![](images/image-61c09f0d8cf10.png)

If desired, the subsequent options facilitate importing SSH keys, reading a remote config, etc which are optional. A summary will be presented before the install begins:

![](images/image-61c09f5d06998.png)

Proceed with the install.

Note : After a reboot, it may take a few minutes before harvester reports as being in a `ready` state - Once it does, navigate to the reported management URL.

![](images/image-61c0a1b1a0953.png)

At which point you will be prompted to reset the `admin` password

## Step 5 - Configure VM Network

Once logged in to Harvester navigate to Hosts > Edit Config

![](images/image-61c0a7c146a13.png)

Configure the secondary NIC to the VLAN network (our VM network)

![](images/image-61c0a81a8230b.png)

Navigate to Settings > VLAN > Edit

![](images/image-61c0a875d8c24.png)

Click "Enable" and select the default interface to the secondary interface. This will be the default for any new nodes that join the cluster.

![](images/image-61c0a8bdba377.png)

To create a network for our VM's to reside in, select Network > Create:

![](images/image-61c0a97242e1c.png)

Give this network a name and a VLAN ID. Note - you can supply VLAN ID 1 if you're using the native/default VLAN.

![](images/image-61c0a9d7720ee.png)

## Step 6 - Test VM Network

Firstly, create a new `image`:

![](images/image-61c0b69360951.png)

For this example, we can use an ISO image. After supplying the URL Harvester will download and store the image:

![](images/image-61c0b72283e55.png)

After downloading, we can create a VM from it:

![](images/image-61c0b7785681f.png)

Specify the VM specs (CPU and Mem)

![](images/image-61c0b813bec18.png)

Under Volumes, add an additional volume to act as the installation target for the OS (Or leave if purely wanting to use a live ISO):

![](images/image-61c0b89747719-1024x625.png)

Under Networks, change the selection to the VM network that was previously created and click "Create":

![](images/image-61c0b8f099707-1024x623.png)

Once the VM is in `running` state, we can take a VNC console to it:

![](images/image-61c0bbc66d7e9.png)

At which point we can interact with it as we would expect with any HCI solution:

![](images/image-61c0bc3dd27c2-1024x863.png)
