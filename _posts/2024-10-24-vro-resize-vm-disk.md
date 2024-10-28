---
layout: post
title: Smart VM disk size management with vRO
date: "2024-10-24"
media_subpath: /assets/img/vro-resize-vm-disk/
categories: [VMware, Build Tools, Aria Orchestrator, How To]
tags: [vmware, building_tools, disk_management, tricks]
---

Disk management in vRO was always a challenging task, because it may require some not trivial steps, which are usually, not visible in GUI or hidden by Powershell modules. Once of such tasks we're going to talk about today - how to increase a disk size.

General goals:

* Support dynamic disk resize.
* Create informative custom form.
* Check a snapshot existence before resize.
* Validate maximum disk size before resize.

The use case:

We aim to design a workflow that enhances disk size. Possible volitions should be checked before the resize itself.

The diagram below explains the logic.
![Image](image.png){: .shadow }{: .normal }
_Logic diagram_

## The solution

### Get disks details

First, we want to display a list of available disks for the selected virtual machine. To achieve this, let's create a function (action element) called `getListOfDisksProperties` that will be responsible for this task.
This function performs two primary functions:

1. Check, if there are snapshots exists for that VM. If so, **throw an error**.
2. Get all the disks and their details:
   1. Disk label
   2. Disk size (returned in KB)

#### Get VM snapshots

We will create a simple function called `getSnapshot` to obtain snapshots. This function retrieves all the snapshots of a VM and returns them as an array of `VcVirtualMachineSnapshot` objects.

>I want to pause for a moment to explain the logic behind getting a snapshot tree - this is an interesting part.
>If  `vm.snapshot.rootSnapshotList` exists, the function will iterate over each root snapshot and initiate the tree traversal using the `traverseSnapshotTree` function. A helper function, `traverseSnapshotTree`, is defined within `getSnapshot` .
>This function takes a `VcVirtualMachineSnapshotTree`, representing a node in the snapshot tree. It adds the snapshot property to the snapshots array and then recursively visits all child snapshot trees by calling itself on each child.
>
>This approach ensures that the function collects the root snapshot and any nested child snapshots by traversing the entire tree.
{: .prompt-info}

```typescript
/**
 * getVmSnapshot.ts
 *
 * Get an array of snapshots
 *
 * @param {VC:VirtualMachine} vm - Virtual Machine
 * @returns {Array/VC:VirtualMachineSnapshot} - Array of snapshots
 */
(function getVmSnapshot(vm: VcVirtualMachine): Array<VcVirtualMachineSnapshot> {
  const snapshots: Array<VcVirtualMachineSnapshot> = [];
  if (vm?.snapshot?.rootSnapshotList) {
    const snapshotsTree = function traverseSnapshotTree(tree: VcVirtualMachineSnapshotTree) {
      snapshots.push(tree.snapshot);
      const childTrees: Array<VcVirtualMachineSnapshotTree> = tree.childSnapshotList?.filter(Boolean);
      if (childTrees?.length) {
        childTrees.forEach(traverseSnapshotTree);
      }
    };
    vm.snapshot.rootSnapshotList.forEach(snapshotsTree);
  }
  return snapshots;
});


```

>I want to show you how to write this function in vRO native Javascript ES5
>```javascript
>/**
> * getVmSnapshot.js
> *
> * Get an array of snapshots
> *
> * @param {VC:VirtualMachine} vm - Virtual Machine
> * @returns {Array/VC:VirtualMachineSnapshot} - Array of snapshots
> */
>(function getVmSnapshot(vm) {
>  var snapshots = [];
>  
>  if (vm && vm.snapshot && vm.snapshot.rootSnapshotList) {
>    var traverseSnapshotTree = function(tree) {
>      snapshots.push(tree.snapshot);
>      var childTrees = (tree.childSnapshotList || []).filter(function(child) {
>        return !!child;
>      });
>      
>      if (childTrees.length) {
>        childTrees.forEach(traverseSnapshotTree);
>      }
>    };
>    
>    vm.snapshot.rootSnapshotList.forEach(traverseSnapshotTree);
>  }
>  
>  return snapshots;
>})
>```
{: .prompt-tip}

#### Get disks details

```typescript
/**
 * getListOfDisksProperties.ts
 *
 * Get disks properties
 *
 * @param {VC:VirtualMachine} vm - Virtual Machine
 * @returns {Array/string} - Array of available disks labels
 */

(function getListOfDisksProperties(vm: VcVirtualMachine): Array<string> {
  const getVmSnapshots = System.getModule("com.clouddepth.disk_management.actions").getVmSnapshot(vm);
  if (getVmSnapshots.length > 0) throw new Error("There are one or more snapshots found. Cannot increase the disk size.");

  const devices: Array<VcVirtualDevice> = vm.config.hardware.device;
  const convertBytes = System.getModule("com.examples.vmware_aria_orchestrator_examples.actions").generalFunctions();
  const diskInfo: Array<string> = devices
    .filter((device) => device instanceof VcVirtualDisk)
    .map((device) => {
      //@ts-ignore
      const convertedBytes = convertBytes.GeneralFunctions.prototype.convertBytes(device.capacityInBytes);
      return `${device.deviceInfo.label} | ${convertedBytes}`;
    });

  return diskInfo;
});


```

When we select a (VM) to work on in a custom form, we can treat it as an object. Consequently, we can access all its devices by retrieving them from the `vm.config.hardware.device`. If a device is an instance of `VcVirtualDisk`, we will add it to the array of devices we want to work with.

After filtering all the disks, we need to obtain their labels and sizes to display them later in a custom form.
The issue we face is that `device.capacityInBytes` returns the size in bytes, but we want to show the information to the user in gigabytes (GB).
To address this, we can create a small function called `convertBytes`. This function will automatically convert bytes to megabytes (MB) or gigabytes (GB) based on the size provided in bytes.

```typescript
public convertBytes(bytes: number) {
  if (bytes === null || bytes === undefined) {
    throw new Error("Mandatory input 'bytes' is null or undefined");
  }

  if (typeof bytes !== "number" || bytes < 0) {
    throw new Error("'bytes' must be a non-negative number");
  }

  const sizes = ["Bytes", "KB", "MB", "GB", "TB"];
  if (bytes === 0) return "0 Bytes";

  const i = Math.floor(Math.log(bytes) / Math.log(1024));
  const result = (bytes / Math.pow(1024, i)).toFixed(2);

  return `${result} ${sizes[i]}`;
}
```

Now, we can create a visually appealing string using the pipe symbol and display it in the dropdown box as “Hard Disk 1 | 10 GB,” for instance.
I wanted to include the disk size next to its name as well. This provides more information than just the name alone, as names are often less relevant.

### Set disk size

Now that we know which disk we want to extend, we need to know by how much. For this purpose, we have a dedicated input called `diskSize`It is a simple input of type `number`, which is limited up to 62TB in a custom form.

### Workflow

Once we have all the necessary details, we can initiate our workflow. The process is quite simple:

1. Locate the disk object using its name (selected by the user in the custom form).
2. Increase the disk size by creating a new disk specification with the updated size provided by the user in the custom form.

>We’re employing the @Item decorator in this example to demonstrate how to create multiple scriptable tasks. Each @Item will generate a distinct scriptable task in the workflow.
{: .prompt-info}

```typescript
export class IncreaseDiskSize {
  @Item({ target: "increaseDisk" })
  public getDiskObjectByLabel(vm: VcVirtualMachine, disks: string, @Out disk: VcVirtualDisk) {
    disk = System.getModule("com.clouddepth.disk_management.actions").getDiskObjectByLabel(vm, disks);
    if (!disk) throw new Error("No disk found");
  }
  @Item({ target: "end" })
  public increaseDisk(vm: VcVirtualMachine, disk: VcVirtualDisk, diskSize: number, @Out diskSpec: VcVirtualMachineConfigSpec) {
    const diskManagement = new DiskManagement();
    diskSpec = diskManagement.increaseDiskSize(vm, disk, diskSize);
    if (!diskSpec) throw new Error("No diskSpec found");
    try {
      diskManagement.reconfigureVM(vm, diskSpec);
    } catch (error) {
      throw new Error(`Failed to reconfigure VM. ${error}`);
    }
  }
}
```

Lets see how to do that.

#### Get disk object by label

To increase the disk size, we need to treat the disk as an object. Let's create a simple function called `getDiskObjectByLabel` that returns a disk object based on its label.

```typescript
/**
 * getDiskObjectByLabel.ts
 *
 * Get list of available disks
 *
 * @param {VC:VirtualMachine} vm - Virtual Machine
 * @returns {VC:ManagedObject} - Disk object
 */

(function (vm: VcVirtualMachine, labelToMatch: string): VcVirtualDevice | null {
  const devices: Array<VcVirtualDevice> = vm.config.hardware.device;

  // Extract the part of the label before the "|" character
  const labelPart = labelToMatch.split("|")[0].trim();
  const matchedDevice: VcVirtualDevice | undefined = devices.find((device) => device?.deviceInfo?.label === labelPart);
  return matchedDevice ? matchedDevice : null;
});
```

Since we're displaying a dropdown list of disks in the format “Hard Disk Name | Hard Disk Size,” we need to eliminate the size component and exclusively operate with the name.
Consequently, we'll split our string and utilize its initial segment to identify a suitable disk based on its label.

#### Increase disk size

If you recall, we already have a written class called [`DiskManagement`](https://github.com/unbreakabl3/vmware_aria_orchestrator_examples/blob/80f33521d189d14df7dbfe9d8db90a00209a5bad/disk_management/src/actions/diskManagement.ts). Therefore, let's add a new method into called `increaseDiskSize`.

```typescript
public increaseDiskSize(vm: VcVirtualMachine, diskToIncrease: VcVirtualDisk, diskSizeGB: number): VcVirtualMachineConfigSpec {
  const kilobytes = 1024 * 1024;
  const newSizeKB = diskToIncrease.capacityInKB + diskSizeGB * kilobytes; // Convert to Kilobytes
  const spec = new VcVirtualMachineConfigSpec();
  spec.deviceChange = [new VcVirtualDeviceConfigSpec()];
  spec.deviceChange[0].operation = VcVirtualDeviceConfigSpecOperation.edit;
  spec.deviceChange[0].device = new VcVirtualDisk();
  spec.deviceChange[0].device.controllerKey = diskToIncrease.controllerKey;
  spec.deviceChange[0].device.unitNumber = diskToIncrease.unitNumber;
  spec.deviceChange[0].device.key = diskToIncrease.key;
  //@ts-ignore
  spec.deviceChange[0].device.backing = new VcVirtualDiskFlatVer2BackingInfo();
  //@ts-ignore
  spec.deviceChange[0].device.backing.fileName = diskToIncrease.backing.fileName;
  //@ts-ignore
  spec.deviceChange[0].device.backing.diskMode = diskToIncrease.backing.diskMode;
  //@ts-ignore
  spec.deviceChange[0].device.capacityInKB = newSizeKB;
  return spec;
  }
```

This method accepts a VM, a disk that we want to extend, and the size for the extension.

>The size must be specified in kilobytes (KB), but the provided size is in gigabytes (GB), so we need to convert it to KB.
{: .prompt-info}

Let's examine the steps to increase the disk size. First, we need to create a new `VcVirtualMachineConfigSpec`.
Within this, we will create a new `VcVirtualDeviceConfigSpec`, as we're updating both the VM specifications and the virtual devices. Next, we must create a new `VcVirtualDisk`, which we will use to update the existing disk.

Each disk has a unique key identifier generated automatically and randomly. In our example, if the disk we want to extend has a key of 2000, we must include this key in the new spec to indicate to vRO which disk we want to update.

```javascript
...
dynamicType = null, 
dynamicProperty = null, 
key = 2000, 
deviceInfo = (vim.Description) { dynamicType = null, dynamicProperty = null, label = Hard disk 1, summary = 4,194,305 KB }
...
```

>The value of the `key` above  and more can be fetched by running simple code below.
```javascript
var devices = vm.config.hardware.device;
for each (device in devices) {
    if (device instanceof VcVirtualDisk) {
        System.log(device)
    }
}
```
{: .prompt-tip}

`fileName`, `diskMode` ,`controllerKey` and `unitNumber` are mandatory inputs, therefore we’ll pass them as is.
And of course, the reason we're here - `newSizeKB`. This value is calculated by adding a new size to the existing size of the disk and converted to KB.
Now, we're ready to return this spec and reconfigure the VM by running `diskManagement.reconfigureVM(vm, diskSpec)` .
>To read more about storage controllers and unit numbers in my [previous article](https://www.clouddepth.com/posts/vro-disk-management/).
{: .prompt-info}
>You can fairly ask, “How do I know to use the `VcVirtualDiskFlatVer2BackingInfo` class and its properties when building a `VcVirtualDeviceConfigSpec`?” To answer this, we need to examine the API response. If we use the code sample above and print `vm.config.hardware.device`, we can see many details about the `VcVirtualDisk` object we want to work with.
```shell
2024-10-24 14:29:55.839 +02:00infoDynamicWrapper (Instance) : [VcVirtualDisk]-[class com.vmware.o11n.plugin.vsphere_gen.VirtualDisk_Wrapper] -- VALUE : (vim.vm.device.VirtualDisk) {
   dynamicType = null,
   dynamicProperty = null,
   key = 2000,
   deviceInfo = (vim.Description) {
      dynamicType = null,
      dynamicProperty = null,
      label = Hard disk 1,
      summary = 10,485,761 KB
   },
   backing = (vim.vm.device.FlatVer2BackingInfo) {
      dynamicType = null,
      dynamicProperty = null,
      fileName = [NACL-01T] win01/win01_2.vmdk,
      datastore = Stub: moRef = (ManagedObjectReference: type = Datastore, value = datastore-74, serverGuid = null), binding = https://ops-vc01t.embl.de:443/sdk,
      backingObjectId = ,
      diskMode = persistent,
      split = false,
      writeThrough = false,
      thinProvisioned = false,
      eagerlyScrub = false,
      uuid = 6000C295-f307-a18f-3a85-d467caf8f0b5,
      contentId = 456a6fc013e2b6349baafe6efffffffe,
      changeId = null,
      parent = null,
      deltaDiskFormat = null,
      digestEnabled = false,
      deltaGrainSize = null,
      deltaDiskFormatVariant = null,
      sharing = sharingNone,
      keyId = null
   },
   ```
>Our backing device has a type of `FlatVer2BackingInfo`. We can see it here on line 11: `backing = (vim.vm.device.FlatVer2BackingInfo)`. Let’s take this type and go to the vRO API. Here we are. This is the name of the class we want to use in our code.
![Image](image 9.png){: .shadow }{: .normal }
{: .prompt-tip}

## Tests

Let's begin our workflow. First, select a virtual machine. In the Disk Properties window, we can easily view all the disks and their sizes, which is very convenient.
![Image](image 2.png){: .shadow }{: .normal }
Choose the disk you want to extend and specify the desired size.
![Image](image 6.png){: .shadow }{: .normal }
Run the workflow.
![Image](image 7.png){: .shadow }{: .normal }
If everything went well, we can confirm in the vCenter that the disk was increased.
![Image](image 8.png){: .shadow }{: .normal }
Let’s explore the consequences of a sticky “0” key on our keyboard. Our philosophy is to create bulletproof solutions whenever possible. To that end, we have implemented an external validation that ensures the total disk size cannot exceed 62TB.
![Image](image 3.png){: .shadow }{: .normal }

For that, lets create an action element called `validateMaximumDiskSize`. It is pretty simple and self explanatory function.

```typescript
/**
 * validateMaximumDiskSize.ts
 *
 * Validate disk size doesn't exceed 62TB
 * @param {string} currentDiskSize - Current Disk Size
 * @param {number} sizeToAdd - GB to add
 * @returns {string}
 */
(function validateFreeDeviceUnits(currentDiskSize: string, sizeToAdd: number): string {
  const maxDiskSizeInGb = 57741; // 62 TB in GB

  // Use a regular expression to extract the number from the currentDiskSize string
  const match: RegExpMatchArray | null = currentDiskSize.match(/\d+(?=\s*GB)/);
  if (!match) {
    return "Invalid disk size format";
  }

  // Convert the extracted string to a number
  const currentDiskSizeInGb = parseInt(match[0], 10);

  // Perform the size validation
  return currentDiskSizeInGb + sizeToAdd > maxDiskSizeInGb ? "Disk size will be bigger than 62TB" : "";
});
```

Using an example below, we can create a validation and bind the inputs.
![Image](image 4.png){: .shadow }{: .normal }
Now, let's select another virtual machine that has a snapshot. Upon doing so, we immediately encounter an error stating that this virtual machine has a snapshot and cannot be extended. This serves as another cool safety check.
![Image](image 5.png){: .shadow }{: .normal }

>This type of error differs from external validation. Here, the error is triggered immediately when the VM is selected. External validation only works when the Run button is hit.
{: .prompt-tip}

## Summary

Disk management in vRO can be challenging, but it’s definitely doable. The most important thing is to always strive for bulletproof solutions. Sometimes, it may take more time to create a solution than to create the workflow itself. However, if the workflow fails repeatedly, no one will want to use it. :)

## Source Code

The source code can be found [here](https://github.com/unbreakabl3/vmware_aria_orchestrator_examples/tree/main/disk_management/src).

The vRO packages are also available [here](https://github.com/unbreakabl3/vmware_aria_orchestrator_examples/raw/refs/heads/main/disk_management/com.clouddepth.disk_management-1.0.136.package), [here](https://github.com/unbreakabl3/vmware_aria_orchestrator_examples/raw/refs/heads/main/general_examples/com.examples.vmware_aria_orchestrator_examples-1.0.25.package) and the external ECMASCRIPT package [here](https://github.com/unbreakabl3/vmware_aria_orchestrator_examples/raw/refs/heads/main/disk_management/com.vmware.pscoe.library.ecmascript-2.43.0.package).

> All packages should be imported
{: .prompt-info}
