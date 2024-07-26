---
layout: post
title: Dependency Injection pattern in vRO or how to add vNIC to VM in vRA as a custom day 2 action
date: "2024-07-25"
media_subpath: /assets/img/vra-how-to-add-vnic-to-vm-as-day2-action/
categories: [VMware, Build Tools, Aria Orchestrator, Aria Automation, How To]
tags: [vmware, building_tools, day_2, custom_resources]
---

Today, we're going to implement a day 2  action for vRA deployment. Custom day 2 actions are a great feature of vRA. They allow us to add almost anything. We will use this feature to add and connect a new vNIC to the deployed VM. We will support both Standard and Distributed switches and all available types of vNICs. Additionally, we'll use <u>native JavaScript</u>, but we'll employ more advanced techniques to make our code more professional. To do this, we'll create a networking class that covers all the network parts.
The idea is to create a workflow that will be triggered by the deployment and will call the action element, which will take care of creating and configuring a new network card.

To the VSCode.

## Action element

### Classes in ES5

In vRO, action elements are JavaScript functions wrapped by vRO. However, these are not regular functions but Immediately-Invoked Function Expressions ([IIFE](https://developer.mozilla.org/en-US/docs/Glossary/IIFE)), which run as soon as they are defined. We define an IIFE function inside parentheses and add () to execute that function.

Several preparation steps are needed to add a new network interface card (NIC) to the VM, all related to networking. Therefore, it would be reasonable to create a networking class. It's important to note that vRO's JavaScript engine is Rhino, and its current version supports JavaScript version 5.1, which doesn't support classes natively. Nonetheless, some techniques can help bridge this gap.

> More details about classes in Javascript ES5 can be found [here](https://medium.com/@apalshah/javascript-class-difference-between-es5-and-es6-classes-a37b6c90c7f8).
{: .prompt-tip}

### Network Class

Let’s create a new action element called `virtualNetworkManagement` and a new class called `VirtualNetworkManagement`. This class is a function that utilizes the prototype feature of the JS. To create a NIC, we need three preparation steps (methods):

1. **createVirtualDeviceConfigSpec** - generate a new config spec for a new device (NIC). The operation will be ADD, so we want to add a NIC.
2. **createVirtualDeviceConnectInfo** - defines the connectivity properties of the NIC. All arguments in that method are booleans.
3. **createVirtualEthernetAdapter** - defines the type of the NIC. `adapterType` is a string that will be passed from the custom form selected by the user.

```javascript
var VirtualNetworkManagement = /** @class */ (function () {
  VirtualNetworkManagement.prototype.createVirtualDeviceConfigSpec = function (
    operation
  ) {
    var vmConfigSpec = new VcVirtualDeviceConfigSpec();
    vmConfigSpec.operation = operation;
    return vmConfigSpec;
  };
  VirtualNetworkManagement.prototype.createVirtualDeviceConnectInfo = function (
    allowGuestControl,
    connected,
    startConnected
  ) {
    var connectInfo = new VcVirtualDeviceConnectInfo();
    connectInfo.allowGuestControl = allowGuestControl;
    connectInfo.connected = connected;
    connectInfo.startConnected = startConnected;
    return connectInfo;
  };
  VirtualNetworkManagement.prototype.createVirtualEthernetAdapter = function (
    adapterType
  ) {
    switch (adapterType) {
      case "E1000":
        return new VcVirtualE1000();
      case "E1000e":
        return new VcVirtualE1000e();
      case "Vmxnet":
        return new VcVirtualVmxnet();
      case "Vmxnet2":
        return new VcVirtualVmxnet2();
      case "Vmxnet3":
        return new VcVirtualVmxnet3();
      default:
        throw new Error("Unknown adapter type: ".concat(adapterType));
    }
  };
  return VirtualNetworkManagement;
})();
```

All these are required to create a new vNIC.
Now, we're ready to add a vNIC to the VM. We'll need another method in our class called **addVnicToSwitch** to do so.

```javascript
VirtualNetworkManagement.prototype.addVnicToSwitch = function (
  vm,
  switchType,
  adapterType
) {
  var configSpec = new VcVirtualMachineConfigSpec();
  var vmConfigSpecs = [];
  // Create virtual device config spec for adding a new device
  var vmDeviceConfigSpec = this.createVirtualDeviceConfigSpec(
    VcVirtualDeviceConfigSpecOperation.add
  );
  // Create connection info for port group
  var connectInfo = this.createVirtualDeviceConnectInfo(true, true, true);
  // Create virtual ethernet adapter based on adapter type
  var vNetwork = this.createVirtualEthernetAdapter(adapterType);
  if (!vNetwork) throw new Error("Failed to create VirtualEthernetAdapter");
  vNetwork.backing = switchType;
  vNetwork.unitNumber = 0;
  vNetwork.addressType = "Generated";
  vNetwork.wakeOnLanEnabled = true;
  vNetwork.connectable = connectInfo;
  // Add the configured virtual ethernet adapter to device specs
  vmDeviceConfigSpec.device = vNetwork;
  vmConfigSpecs.push(vmDeviceConfigSpec);
  configSpec.deviceChange = vmConfigSpecs;
  System.log("Reconfiguring the virtual machine to add new vNIC");
  try {
    var task = vm.reconfigVM_Task(configSpec);
    System.getModule("com.vmware.library.vc.basic").vim3WaitTaskEnd(
      task,
      true,
      2
    );
  } catch (error) {
    throw new Error("Failed to create vNIC");
  }
};
```

#### Dependency Injection Pattern

The most interesting aspect of this method is a variable called `switchType` (line 17 in `addVnicToSwitch` method above). Both Standard and Distributed switches share almost identical configurations for all the provided code except for the backing part. For the Standard vSwitch, the `vNetwork.backing` must be `VcVirtualEthernetCardLegacyNetworkBackingInfo`, while for the Distributed vSwitch, it must be `VcVirtualEthernetCardDistributedVirtualPortBackingInfo`.

Since we want to support connections for both Standard and Distributed switches, we must find an elegant way to replace only the `switchType` variable. Otherwise, we would need to duplicate our code just because of one line of code. This is where the Dependency Injection pattern in JavaScript comes into play. A well-explained example can be found [here](https://stackoverflow.com/a/6085922).

We will implement a light version of this pattern, focusing on its core functionality, which can be extended as needed. The idea is to create external classes that prepare the Distributed and Standard switch objects and inject them into the main `addVnicToSwitch` method as a `switchType` variable. By doing this, we decouple the object's construction (`SwitchPortConnection` in our case for both types of switches) from the function it is called.
Therefore, we are going to write another three classes:

- The `StandardVirtualSwitchPortConnection` class is preparing the backing of the standard switch.

```javascript
var StandardVirtualSwitchPortConnection = /** @class */ (function () {
  function StandardVirtualSwitchPortConnection() {}
  StandardVirtualSwitchPortConnection.prototype.createStandardVirtualSwitchPortConnection =
    function (standardNetworkGroup) {
      if (!standardNetworkGroup)
        throw new Error("'standardNetworkGroup' argument is required");
      var backingInfo = new VcVirtualEthernetCardLegacyNetworkBackingInfo();
      backingInfo.useAutoDetect = true;
      backingInfo.deviceName = standardNetworkGroup.name;
      return backingInfo;
    };
  return StandardVirtualSwitchPortConnection;
})();
```

- The `DistributedVirtualPortBackingInfo` and `DistributedVirtualSwitchPortConnection`, which are preparing the baking of the distributed switch.

```javascript
var DistributedVirtualPortBackingInfo = /** @class */ (function () {
  function DistributedVirtualPortBackingInfo() {}
  DistributedVirtualPortBackingInfo.prototype.createVirtualEthernetCardDistributedVirtualPortBackingInfo =
    function (port) {
      if (!port) throw new Error("'Port' argument is required");
      var backingInfoDvs =
        new VcVirtualEthernetCardDistributedVirtualPortBackingInfo();
      backingInfoDvs.port = port;
      return backingInfoDvs;
    };
  return DistributedVirtualPortBackingInfo;
})();
exports.DistributedVirtualPortBackingInfo = DistributedVirtualPortBackingInfo;

//########################################################################

var DistributedVirtualSwitchPortConnection = /** @class */ (function () {
  function DistributedVirtualSwitchPortConnection() {}
  DistributedVirtualSwitchPortConnection.prototype.createDistributedVirtualSwitchPortConnection =
    function (portGroup) {
      if (!portGroup) throw new Error("'portGroup' argument is required");
      var distributedPortConnection =
        new VcDistributedVirtualSwitchPortConnection();
      var distributedVirtualSwitch = VcPlugin.convertToVimManagedObject(
        portGroup,
        portGroup.config.distributedVirtualSwitch
      );
      distributedPortConnection.switchUuid = distributedVirtualSwitch.uuid;
      distributedPortConnection.portgroupKey = portGroup.key;
      return distributedPortConnection;
    };
  return DistributedVirtualSwitchPortConnection;
})();
```

>Usually, the class' constructor is used to utilize the DI pattern. The constructor is receiving the dependency, which wants to eject. This function is used as a constructor for `VirtualNetworkManagement` class, because ES5 does not support constructors.
>```javascript
>function VirtualNetworkManagement(switchType) {
>this.switchType = switchType;
>}
>```
{: .prompt-tip}

## Workflow

The workflow is quite simple. We call the `virtualNetworkManagement()` class and execute its methods based on the given inputs. In this process, we apply dependency injection (DI) depending on the type of vSwitch provided. After selecting the switch, we prepare it and pass it to the `addVnicToSwitch` method, which handles the task. Cool, isn't it?

![Image](image.png){: .shadow }{: .normal }

Let's start the workflow and see what we have.

There are two ways to provide a predefined list of adapter types: hardcode it in the custom form or use an action element as an external action. The second one if a best practice. Therefore, we create a new, simple action element that returns an array of strings.

![Image](image%2011.png){: .shadow }{: .normal }

As a result, we can see this array in a dropdown list.

![Image](image%202.png){: .shadow }{: .normal }

Switch type input has two predefined values as well.

![Image](image%203.png){: .shadow }{: .normal }

Based on what switch type is selected, a different port group appears. Of course, both standard and distributed portgroups are searchable fields.

![Image](image%204.png){: .shadow }{: .normal }

![Image](image%205.png){: .shadow }{: .normal }

## vRA Day 2 action

Lets add new Resource Action. The resource type will be a `Cloud.vSphere.Machine`, because we want to add a vNIC to the VM. The workflow will be our 'Add NIC to VM' workflow. 
>Don't forget to enable 'Activate' option. Otherwise, our day 2 action will no be visible in the UI.
{: .prompt-info}

All the workflow's inputs will be automatically added to the Property Binding section below.

![Image](image%206.png){: .shadow }{: .normal }

One of the important things to do when we have an input of type `VC:VirtualMachine` is properly binding it. When this day 2 action will run, the vRA should know to which VM this should be applied. To do so, the auto-discovery process will run. This is represented by a built-in action element called `findVcVmByVcAndVmUuid`, which will find the VM and return it as an object. Therefore, <u>changing the default binding "in request" to "with binding action" is essential</u>.

![Image](image%207.png){: .shadow }{: .normal }

Now, we can open one of the deployments in vRA, select the cloud vsphere machine in the canvas, click on the ACTIONS button and see a new "Add vNIC" action.

![Image](image%208.png){: .shadow }{: .normal }

Select the adapter type.

![Image](image%209.png){: .shadow }{: .normal }

And select the portgroup based on the virtual switch type.

![Image](image%2010.png){: .shadow }{: .normal }

## Summary

VMware Aria Automation and VMware Aria Orchestrator provide an excellent tandem for automating almost anything (XaaS? 😉). And now, we can extend it even further and bring different design patterns into it.

Any feedback is highly appreciated.

## Source Code

The source code with the unit tests can be found [here](https://github.com/unbreakabl3/vmware_aria_orchestrator_examples/tree/main/add_new_nic_to_vm/src) and [here](https://github.com/unbreakabl3/vmware_aria_orchestrator_examples/blob/main/general_examples/src/actions/virtualNetworkManagement.ts).

The vRO packages are also available [here](https://github.com/unbreakabl3/vmware_aria_orchestrator_examples/blob/main/add_new_nic_to_vm/com.clouddepth.add_new_nic_to_vm-1.0.14.package) and [here](https://github.com/unbreakabl3/vmware_aria_orchestrator_examples/blob/main/general_examples/com.examples.vmware_aria_orchestrator_examples-1.0.20.package).
