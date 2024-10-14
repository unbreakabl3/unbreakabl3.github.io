---
layout: post
title: VM Disk Management with vRO
date: "2024-09-19"
media_subpath: /assets/img/vro-disk-management/
categories: [VMware, Build Tools, Aria Orchestrator, How To]
tags: [vmware, building_tools, disk_management]
---

Disk management in vCenter is a straightforward process. However, what if we want to create a new disk using a code? What if we want to specify a storage controller to which we want to attach the disk? What if we want to ensure that the storage controller has a free slot to attach a new disk? Today, we will attempt to accomplish these tasks. While the relatively simple task may seem straightforward, it will turn out to be slightly more challenging than it initially appears.

## General goals

- Show the user all necessary details.
- Support for all available storage controllers.
- Validate that there are available slots in the storage controller to attach a new disk and warn the user.
- Limit the maximum disk size.
- Automatically find the next available slot in a selected storage controller.

vRBT goals:

- Explore how to create multiple scriptable tasks with vRBT

To achieve our objectives, we’ll need to gather some specific details based on the provided user information and collect data directly from the virtual machine.

A use case:

We aim to create and attach a new disk with a custom size to one of the available storage controllers within the virtual machine:

1. Ask the user to select a VM.
2. Ask the user to select one of the available storage controllers.
3. Ask the user to provide a disk size.
4. Run the workflow.
![Image](image.png){: .shadow }{: .normal }
_Logic diagram_

## The solution

### Get all available storage controller names

To display the list of all available storage controllers in the selected virtual machine on a custom form, we need to create a function called `getDeviceControllerNames`. This function is straightforward and retrieves an array of storage controller names by invoking the `getDeviceControllers` function.

```javascript
/**
 * Get the device controllers names
 *
 * @param {VC:VirtualMachine} vm - The name of the virtual machine to check.
 * @returns {Array/string} - Array of used device controllers names
 */
(function (vm) {
  if (!vm) return [];
  var deviceControllers = System.getModule(
    "com.clouddepth.disk_management.actions"
  ).getDeviceControllers(vm);
  return deviceControllers.map(function (device) {
    return device.deviceInfo.label;
  });
});
```

The `getDeviceControllers` function queries all available types of controllers in the virtual machine and returns only those that are instances of a specific type.

```javascript
/**
 * Get the device controllers associated
 *
 * @param {VC:VirtualMachine} vm - The name of the virtual machine to check.
 * @returns {Array/object} - Array of used device controllers associated
 */
(function (vm) {
  if (!vm) return [];
  var devices = vm.config.hardware.device;
  return devices.filter(function (device) {
    return (
      device instanceof VcParaVirtualSCSIController ||
      device instanceof VcVirtualIDEController ||
      device instanceof VcVirtualAHCIController ||
      device instanceof VcVirtualLsiLogicSASController ||
      device instanceof VcVirtualNVMEController
    );
  });
});
```

As a result, we have this list below.
![Image](image 2.png){: .shadow }{: .normal }

### Get the disks - external validation

Once the appropriate storage controller is chosen, we want to determine the number of disks already attached to that controller. This is a crucial concept behind this feature: fail early. If there aren’t any available slots, we want to alert the user and prevent the workflow from proceeding (using an external validation).

To achieve this, we need to gather several pieces of information to decide whether we can create a new disk and attach it to the controller:

1. Identify the required storage controller.
2. Determine the number of disks already attached to that controller.
3. Obtain the maximum number of disks that can be attached to that type of controller.
4. Calculate the available slots.

This is precisely what the external validation function `validateFreeDeviceUnits` accomplishes.

```typescript
/**
 * Validate if there are any free storage controller slots available
 * @param {string} diskControllerName - Disk controller name
 * @param {VC:VirtualMachine} vm - Virtual machine
 * @returns {string}
 */
(function validateFreeDeviceUnits(
  vm: VcVirtualMachine,
  diskControllerName: string
) {
  const deviceControllers = System.getModule(
    "com.clouddepth.disk_management.actions"
  ).getDeviceControllers(vm);
  if (!deviceControllers) return "No controllers found";
  const attachedDisks: Array<number> = System.getModule(
    "com.clouddepth.disk_management.actions"
  ).getDeviceControllerAttachedDisks(deviceControllers, diskControllerName);
  const usedDeviceUnits: Array<number> = System.getModule(
    "com.clouddepth.disk_management.actions"
  ).getDeviceUsedUnitsNumber(vm, attachedDisks);
  const maxDeviceUnits: number = System.getModule(
    "com.clouddepth.disk_management.actions"
  ).setDeviceUnusedUnitsNumber(deviceControllers, diskControllerName);
  if (!maxDeviceUnits) return "No free device units available";
  const freeDeviceUnit: number = System.getModule(
    "com.clouddepth.disk_management.actions"
  ).getDeviceUnusedUnitsNumber(vm, maxDeviceUnits, usedDeviceUnits);
  if (freeDeviceUnit === 0)
    return "No free device unit number is available to attach a new disk. Select another controller.";
});
```

> Example: `IDE` controller support only up to two disks. In case we want to create a third disk and attach it to that controller, as a result, we’ll display the following warning and will not start the workflow.
>![Image](image 3.png){: .shadow }{: .normal }
{: .prompt-info}

### A storage controller unit number

Each disk, connected to the storage controller, is assigned a specific slot. In the illustration provided, we observe that the first four slots are already occupied. Consequently, if we intend to create a new disk and connect it to that controller, it should be attached to the next available slot (`unitNumber` in API).
![Image](image 4.png){: .shadow }{: .normal }

We can ask the user to select one, but it would be more interesting and intelligent to do it automatically and avoid boring the user with irrelevant questions. To achieve this, we have a function called `getDeviceUsedUnitsNumber`. This function filters all the devices for a specific type, `VcVirtualDisk`, finds its unique ID (key), and then retrieves the unitNumber, which represents the slot number where the disk is connected.

```typescript
/**
 * Get the device unit number not in use
 *
 * @param {VC:VirtualMachine} vm - The name of the virtual machine to check.
 * @param {Array/number} deviceControllerAttachedDisks - Array of disks attached to the controller
 * @returns {Array/number} - Array of used device unit numbers
 */
(function (
  vm: VcVirtualMachine,
  deviceControllerAttachedDisks: Array<number>
): Array<number> {
  const devices: Array<VcVirtualDevice> = vm.config.hardware.device;
  const usedUnitNumbers: Array<number> = [];
  if (
    !deviceControllerAttachedDisks ||
    deviceControllerAttachedDisks.length === 0
  ) {
    return usedUnitNumbers;
  }
  const attachedDiskSet = new Set(deviceControllerAttachedDisks);
  return devices
    .filter(
      (device) =>
        device instanceof VcVirtualDisk && attachedDiskSet.has(device.key)
    )
    .map((device) => device.unitNumber);
});
```

### Maximum number of disks can be attached to the controller

To obtain those numbers, we utilize a function called `setDeviceUnusedUnitsNumber`. This function is remarkably straightforward and merely returns a predetermined value, which is determined by the type of controller.

```typescript
/**
 * Get the device unit number not in use
 *
 * @param {Array/object} deviceControllers - Virtual Machine device controller type
 * @param {string} diskControllerLabel - Storage controller label
 * @returns {number} - Maximum number of devices supported by the device controller
 */
(function (
  deviceControllers: Array<VcVirtualDevice>,
  diskControllerLabel: string
) {
  if (!diskControllerLabel || !deviceControllers)
    throw new Error("Provide parameters are missing");
  const controller = deviceControllers.find((device) => {
    return device.deviceInfo.label === diskControllerLabel;
  });

  if (!controller) {
    throw new Error(`No controller found for label: ${diskControllerLabel}`);
  }
  switch (true) {
    case controller instanceof VcParaVirtualSCSIController:
      return 64;
    case controller instanceof VcVirtualIDEController:
      return 2;
    case controller instanceof VcVirtualAHCIController:
      return 29;
    case controller instanceof VcVirtualLsiLogicSASController:
      return 16;
    case controller instanceof VcVirtualNVMEController:
      return 15;
    default:
      throw new Error("Unknown controller type");
  }
});
```

### Automatically get a next available slot

Once we have the maximum number of slots supported by the storage controller and the number of used slots, we can calculate **how many** and **which** slots are available to use.

The variable `predefinedArray` creates an array of numbers from 0 to the maximum number provided in the previous step.

The variable `unusedDeviceUnits` holds an array of all numbers except those that were removed from it, which represent the already used slots. For instance, a SCSI controller supports up to 64 disks. Two disks are already connected to the controller at slot numbers 2 and 5. Providing this information to the function, we’ll receive an array of numbers from 0 to 63, excluding numbers 2 and 5: [0, 1, 3, 4, 6, 7, 8, …, 63]. Now, we can always take the first available number from this array. This means a new disk will be attached to slot number 0, followed by slot number 1, and so on.

```typescript
/**
 * Get the device unit number not in use
 *
 * @param {VC:VirtualMachine} vm - Virtual Machine to get the unit number
 * @param {number} maximumDeviceUnitsNumber - Number of maximum device units
 * @param {Array/number} deviceUnitToRemove - Array of device units to remove from the list of all available devices
 * @returns {number} - Array of unused device unit numbers
 */
(function (
  vm: VcVirtualMachine,
  maximumDeviceUnitsNumber: number,
  deviceUnitToRemove: Array<number>
): number | Array<number> {
  if (!vm || !maximumDeviceUnitsNumber)
    throw new Error("Provide parameters are missing");
  const predefinedArray: Array<number> = Array.from(
    { length: maximumDeviceUnitsNumber },
    (_, i) => i
  );

  if (deviceUnitToRemove && deviceUnitToRemove.length > 0) {
    const unusedDeviceUnits = predefinedArray.filter(
      (num) => deviceUnitToRemove.indexOf(num) === -1
    );
    return unusedDeviceUnits.length > 0 ? unusedDeviceUnits[0] : 0;
  }

  return predefinedArray;
});
```

### Create a new disk

The last missing piece in our logic is the creation of a disk. To create a new disk and attach it to the storage controller, we need to provide the following details:

1. Virtual Machine (VM).
2. A slot number where the new disk should be attached.
3. Disk size.
4. A storage controller’s unique number, represented by the `key` attribute.

We have all the necessary details except for the `key`. Let’s obtain it. For this, we have a function called `getDeviceControllerKey`. This function is quite straightforward. It retrieves the key value of the controller based on the controller name provided by the user in a custom form.

```typescript
/**
 * Get storage controller device key
 *
 * @param {string} diskControllerLabel - The name of the device controller
 * @param {Array/VcVirtualDevice} deviceControllers - Array of deviceControllers objects
 * @returns {number} - The key number of the device controller
 */
(function (
  diskControllerLabel: string,
  deviceControllers: Array<VcVirtualDevice>
): number | Array<never> | undefined {
  if (!diskControllerLabel || !deviceControllers)
    throw new Error(
      "Both 'diskControllerLabel' and 'deviceControllers' parameters are required."
    );
  const device = deviceControllers.find(
    (device) => device.deviceInfo.label === diskControllerLabel
  );
  return device?.key;
});
```

Now, we’re ready to create our `DiskManagement` class. In this class, we’ll have a method called `createDisk` (which, by the way, will be expanded upon in later classes). This method will accept all the values we’ve collected before. To create a new disk, a few initializations are required:

1. Disk BackingInfo.
2. The actual disk itself.
3. A VM updated configuration specification.

Once these are set, we’ll reconfigure the VM with all the provided details by running the `reconfigureVM` method.

```typescript
export class DiskManagement {
  public createDisk(
    vm: VcVirtualMachine,
    deviceUnitNumber: number,
    diskSize: number,
    deviceControllerKey: number
  ): VcVirtualMachineConfigSpec {
    // Create Disk BackingInfo
    const diskBackingInfo = new VcVirtualDiskFlatVer2BackingInfo();
    diskBackingInfo.diskMode = "persistent";
    diskBackingInfo.fileName = "[" + vm.datastore[0].name + "]";
    diskBackingInfo.thinProvisioned = true;

    // Create VirtualDisk
    const disk = new VcVirtualDisk();
    //@ts-ignore
    disk.backing = diskBackingInfo;
    disk.controllerKey = deviceControllerKey;
    disk.unitNumber = deviceUnitNumber;
    disk.capacityInKB = diskSize * 1024 * 1024;

    // Create Disk ConfigSpec
    const deviceConfigSpec = new VcVirtualDeviceConfigSpec();
    deviceConfigSpec.device = disk;
    deviceConfigSpec.fileOperation =
      VcVirtualDeviceConfigSpecFileOperation.create;
    deviceConfigSpec.operation = VcVirtualDeviceConfigSpecOperation.add;

    const deviceConfigSpecs = [];
    deviceConfigSpecs.push(deviceConfigSpec);

    // List of devices
    const configSpec = new VcVirtualMachineConfigSpec();
    configSpec.deviceChange = deviceConfigSpecs;
    return configSpec;
  }

  public reconfigureVM(
    vm: VcVirtualMachine,
    configSpec: VcVirtualMachineConfigSpec
  ) {
    try {
      const task = vm.reconfigVM_Task(configSpec);
      System.getModule("com.vmware.library.vc.basic").vim3WaitTaskEnd(
        task,
        true,
        3
      );
    } catch (e) {
      throw `Failed to create and attach disk to VM  ${vm.name}. ${e}`;
    }
  }
}
```

### Workflow

Now that we have almost all the necessary details, we can create a workflow and implement this logic step by step. One of our goals is to use multiple scriptable tasks when writing code with vRBT. To achieve this, a new functionality is coming up. Refer to the [documentation](https://github.com/vmware/build-tools-for-vmware-aria/blob/main/docs/versions/latest/Components/Archetypes/typescript/Components/Workflows.md) for more information. To create a new or multiple scriptable tasks, we need to use the `@Item`, `@In`, and `@Out` decorators. The `@Item` decorator creates a new scriptable task element in the canvas, while `@In` and `@Out` map the inputs and outputs of that scriptable task. This is similar to how we do it with the GUI.

Another important aspect is that we need to use `attributes` to create our variables. This is exactly the same as we would do with the GUI.

Here’s an example of a part of our workflow using this method:

```typescript
import { Workflow, In, Item, Out } from "vrotsc-annotations";
import { DiskManagement } from "../actions/diskManagement";

@Workflow({
  name: "Disk Management",
  path: "MyOrg/MyProject",
  id: "",
  description: "This workflow is used to create and attached a new disk to the VM",
  input: {
    vm: { type: "VC:VirtualMachine" },
    diskSize: { type: "number" },
    diskControllerName: { type: "string" }
  },
  attributes: {
    result: { type: "any" },
    deviceControllers: { type: "Array/Object" },
    attachedDisks: { type: "Array/number" },
    usedDeviceUnits: { type: "Array/number" },
    maxDeviceUnits: { type: "number" },
    deviceControllerKey: { type: "number" },
    freeDeviceUnit: { type: "number" }
  }
})
export class CreateAndAttachNewDiskToVM {
  @Item({ target: "getAttachedDisks" })
  public getDeviceControllers(vm: VcVirtualMachine, @Out deviceControllers: Array<VcVirtualDevice>) {
    deviceControllers = System.getModule("com.clouddepth.disk_management.actions").getDeviceControllers(vm);
    if (!deviceControllers) throw new Error("No controllers found");
  }

  @Item({ target: "getUsedDeviceUnits" })
  public getAttachedDisks(@In deviceControllers: Array<VcVirtualDevice>, @In diskControllerName: string, @Out attachedDisks: Array<number>) {
    attachedDisks = System.getModule("com.clouddepth.disk_management.actions").getDeviceControllerAttachedDisks(deviceControllers, diskControllerName);
    if (!attachedDisks) System.log("No attached disks found");
  }

  @Item({ target: "getMaxDeviceUnits" })
  public getUsedDeviceUnits(@In vm: VcVirtualMachine, @In attachedDisks: Array<number>, @Out usedDeviceUnits: Array<number>) {
    usedDeviceUnits = System.getModule("com.clouddepth.disk_management.actions").getDeviceUsedUnitsNumber(vm, attachedDisks);
  }

...


```

As a result, we have a workflow.

![Image](image 8.png){: .shadow }{: .normal }

We have an `attributes` are converted to the `variables`.

![Image](image 5.png){: .shadow }{: .normal }

We have all our `@Item` are converted to the scriptable tasks.

![Image](image 6.png){: .shadow }{: .normal }

And our VM is reconfigured with many new disks, each one of them connected properly.

![Image](image 7.png){: .shadow }{: .normal }

## Summary

Surprisingly, attaching a disk to a virtual machine can be a challenging task. Fortunately, we discovered today how to accomplish this. In the future, we’ll attempt to add a `removeDisk` method to our `DiskManagement` class.

## Source Code

The source code with the unit tests can be found [here](https://github.com/unbreakabl3/vmware_aria_orchestrator_examples/tree/main/disk_management/src)

The vRO packages are also available [here](https://github.com/unbreakabl3/vmware_aria_orchestrator_examples/blob/main/disk_management/com.clouddepth.disk_management-1.0.97.package)  and the external ECMASCRIPT package [here](https://github.com/unbreakabl3/vmware_aria_orchestrator_examples/blob/main/disk_management/com.vmware.pscoe.library.ecmascript-2.41.0.package).

> Both packages should be imported
{: .prompt-info}
