---
layout: post
title: How to set default and maximum VM's hardware version on the cluster level
date: "2024-06-28 12:57:08 +0200"
media_subpath: /assets/img/vro-how-to-set-cluster-vm-hw-version/
categories: [VMware, Build Tools, Aria Orchestrator, How To, vRO]
tags: [vmware, building_tools, vm_hw_version, resource_element, configuration_element, external_validation]
---

A while back, I came across an intriguing blog post by William Lam [blog post](https://williamlam.com/2022/07/how-to-limit-the-maximum-supported-vm-virtual-hardware-compatibility-in-vsphere.html) that got me thinking about how we could achieve a similar goal in the Aria world and even take it a step further. So, I dedicated some time to it, and here we are. Therefore, today, we will implement setting the default VM hardware version when creating a new VM and specify the maximum allowed time in your organization.

Fasten your seat belts. We're diving in.

## The Problem

In a large-scale environment, it's essential to adhere to specific standards. This includes ensuring we use the default and maximum hardware versions for VMs. So, how do we accomplish this?
The concept configures the cluster we intend to work on with the default and maximum VM hardware versions.
![Image](7f9aa6e9-9aa7-4773-831d-ac5c5653dd03.png){: .shadow }{: .normal }
_Workflow form example_

We are going to implement a new workflow that will accommodate the following use cases:

1. Configuration at the cluster level
2. Incorporation of all supported versions for the resource element
3. Capability to reset the configured settings to their default values
4. External validation to ensure that the default version does not exceed a maximum of one

The logic is illustrated in the diagram below.
![Image](b3cd0a71-ed57-46b7-a3df-d463fa5fe3f1.png){: .shadow }{: .normal }
_Logic diagram_

## The solution

### Pre-requisites

[vRBT](https://github.com/vmware/build-tools-for-vmware-aria) supports resource elements and configuration elements. We're going to use both.
Create two files in the `resource` folder.

![Image](aae5b1e7-0113-4079-bed0-a6865b7defbb.png){: .shadow }{: .normal }

The `vm_hw_versions.json` will include the list of all the supported hardware versions.

```json
{
  "vmx-21": "ESXi 8.0 Update 2 and later",
  "vmx-20": "ESXi 8.0 and later",
  "vmx-19": "ESXi 7.0 Update 2 or later",
  "vmx-18": "ESXi 7.0 Update 1 or later",
  ...
  "": "Reset to default cluster/datacenter configuration"
}
```

The second one, `vm_hw_versions.json.element_info.json,` will tell vRO the location where this element should be created and its type: "application/json".

```json
{
 "path": "MyOrg/SetVMHardwareVersion",
 "mimeType": "application/json"
}
```

Create a configuration element in the `config` folder that points to the resource element.

```typescript
import { Configuration } from "vrotsc-annotations";

@Configuration({
  name: "ConfigurationEl",
  path: "MyOrg/MyProject",
  attributes: {
    rsPath: {
      type: "string",
      value: "/MyOrg/SetVMHardwareVersion",
      description: "Resource Element Path"
    },
    rsName: {
      type: "string",
      value: "vm_hw_versions.json",
      description: "Resource Element Name"
    }
  }
})
export class TestConfiguration {}
```

 You may wonder, why would I do that? It might seem redundant. However, the configuration element can be linked to the workflow variables and utilized later in the workflow itself.
>This is a best practice because the versioning of the configuration element can be managed separately from the workflow, for example, for tests.
{: .prompt-tip}
In addition, I wanted to demonstrate how to work with config and resource elements in vRBT using pure code.

### Main

#### Workflow variables

Let's talk about the workflow. To add the variables to the workflow, we need to use a new section called `attributes` (line 19).
>Make sure you're using vRBT v.0.39.0 at least
{: .prompt-info}

```typescript
@Workflow({
  name: "Set VM hardware version",
  path: "MyOrg/MyProject",
  id: "",
  description: "The workflow will configure the VM hardware default and maximum version on selected cluster",
  input: {
    defaultHardwareVersion: {
      type: "string"
    },
    maxHardwareVersion: {
      type: "string"
    },
    cluster: {
      type: "VC:ClusterComputeResource",
      description: "VC Cluster to configure the VM hardware"
    }
  },
  attributes: {
    rsName: { type: "string", bind: true, value: "MyOrg/MyProject/ConfigurationEl/rsName" },
    rsPath: { type: "string", bind: true, value: "MyOrg/MyProject/ConfigurationEl/rsPath" }
  },
  output: {
    result: { type: "Any" }
  },
  presentation: ""
})
```

- `bind: true` - means that we have to bind the value of the attribute to Configuration Element variable.
- `value: "Some/Path/To/ConfigurationElement/variableName"` - points to the Configuration Element and variable inside the Configuration Element to bind to.

When we're working with configuration element in our workflow, it's important to understand how values are referenced. A value's reference path usually follows the format `value: "Some/Path/To/ConfigurationElement/variableName"`. This means that we can directly link a variable in our workflow to a specific attribute within our configuration setup.

In the scenario described, we're specifically looking at two attributes: `rsName` and `rsPath`. Both of these serve as attributes for a configuration element we've already set up. Consequently, we end up with two variables within our workflow. Each of these variables is directly linked to its corresponding attribute in the configuration element—essentially, we have a one-to-one correspondence between each workflow variable and its respective configuration attribute.

![Image](2ccc54b5-7408-493a-baad-bff31d9f2922.png){: .shadow }{: .normal }{: width="300" height="200" }

Now, those two variables can be used inside the workflow's scriptable task.

#### External functions

To display the available hardware (HW) versions to the user, we will first retrieve the values from our resource element and present them in a custom form dropdown box. To accomplish this, we'll utilize a function called `getVmHwVersionsResourceElement`. This function will accept the path and name of the resource element, locate it, and then return an array of strings representing the HW versions. The operation of this function relies on leveraging an external class named `resourceElementManagement`, which is equipped with all the essential methods for handling resource elements efficiently. The class can be found [here](https://github.com/unbreakabl3/vmware_aria_orchestrator_examples/blob/main/general_examples/src/actions/resourceElementManagement.ts).

```typescript
/**
 * @param {string} rsName - Resource element name
 * @param {string} rsPath - Resource element path
 * @returns {Properties} - Array of strings
 */
(function getVmHwVersionsResourceElement(rsName: string, rsPath: string): Array<string> {
  if (!rsName || !rsPath) return [];
  const resourceElement = System.getModule("com.examples.vmware_aria_orchestrator_examples.actions").resourceElementManagement();
  const vars = {
    rsName: rsName,
    rsPath: rsPath
  };
  try {
    const content = resourceElement.ResourceElementManagement.prototype.getResourceElement(vars);
    const jsonObject = content.getContentAsMimeAttachment().content;
    if (typeof jsonObject !== "string") {
      throw new Error("Invalid content format received");
    }
    const jsonData = JSON.parse(jsonObject);
    return jsonData;
  } catch (error) {
    throw new Error(`Error getting VM hardware versions: ${error}`);
  }
});
```

First, we want to show the user the list of available HW versions. To do so, we must use the `getVmHwVersionValue` function to extract keys and values from our `vm_hw_versions.json` file. This function returns an array of strings containing the HW versions, which we can display to the user. The process involves utilizing another function we previously developed, `getVmHwVersionsResourceElement`, to iterate through the keys and fetch their corresponding values.

```typescript
/**
 * @param {string} rsName - Resource element name
 * @param {string} rsPath - Resource element path
 * @returns {Array/string} - Array of values
 */
(function getVmHwVersionValue(rsName: string, rsPath: string): Array<string> {
  if (!rsName || !rsPath) return [];
  try {
    const values: Array<string> = [];
    const jsonData = System.getModule("com.clouddepth.set_vm_hw_version.actions").getVmHwVersionsResourceElement(rsName, rsPath);
    for (var key in jsonData) {
      values.push(jsonData[key]);
    }
    return values;
  } catch (error) {
    System.error(`Error fetching VM hardware versions from resource element: ${error}`);
    return [];
  }
});
```

The out put will be:
![Image](2b4eb77e-7638-4521-821c-7087991c6ece.png){: .shadow }{: .normal }

To make this dropdown work, we need to bind <u>both</u> `defaultHardwareVersion` and `maxHardwareVersion` input to `getVmHwVersionValue` function as shown below.

![Image](f22c9c72-111a-44ca-8b76-0b2bf0448366.png){: .shadow }{: .normal }
_Map the dropdown input to the external action_

And the Display type should be "DropDown"

![Image](264b5ebb-fb39-4f98-b8e5-c7ae51def325.png){: .shadow }{: .normal }

>There's a clever method for integrating the workflow's variables into a custom form, which served as another motivation for establishing those variables. The primary advantage lies in our reliance on a single configuration file linked across various locations, eliminating the need for updates or reconfigurations.
{: .prompt-tip}

![Image](0c84e775-3ae2-4fe7-b28e-8799b0656ef4.png){: .shadow }{: .normal }{: width="300" height="200" }

There's an interesting challenge I encountered while working on improving the user interface for selecting hardware versions. Essentially, it was simpler to present a dropdown menu of keys like "vmx-19" to the user rather than the more descriptive "ESXi 7.0 Update 2 or later". This choice was grounded in the fact that the underlying cluster configuration method requires the argument in the "vmx-?" format. However, from a user experience standpoint, it's significantly more helpful to display the full description ("ESXi 7.0 Update 2 or later"), as few of us remember what "vmx-19" corresponds to.

Acknowledging this, it became clear that we couldn't directly pass the user-friendly value ("ESXi 7.0 Update 2 or later") to the function. We first needed to determine which "vmx-?" key matches the descriptive value. This approach ensures a more intuitive and favorable experience for the end-user, despite the additional complexity it introduces on the back end. This is a classic example of putting the user's needs first, even if it means creating extra work for the developers.

To do so, we need another function. Please welcome on the stage the `getKeyByValue`. That is a straightforward function, which receives our JSON and value to look for as an argument, looping through and returning the key if it's found.

```typescript
/**
 * @param {string} json - JSON
 * @param {string} value - Value to look for
 * @returns {Array/string} - Key of the value
 */
(function getKeyByValue(json: { [key: string]: string }, value: string): string | undefined {
  for (const key in json) {
    if (Object.prototype.hasOwnProperty.call(json, key)) {
      if (json[key] === value) {
        return key;
      }
    }
  }
  return undefined;
});
```

#### Main code

Now, we can use this function to get the keys and values from our `vm_hw_versions.json`. The next step involves leveraging the primary code to identify the "vmx-?" key corresponding to the chosen option in the custom form. As an illustration, if the "Set Default Hardware Version" field is selected as "ESXi 7.0 Update 2 or later", our task is to locate and secure the "vmx-19" key in a variable named `defaultHardwareVersionKey`. With all requisite arguments at our disposal, we can employ the `setVmHardwareVersion` method to reconfigure the cluster effectively.

Our workflow receives a few inputs:

- `rsPath` and `rsName` from the config element
- `cluster`, `defaultHardwareVersion` and  `maxHardwareVersion` - from the custom form input

After completing all the preliminary steps, we now have all the required values to reconfigure our cluster.

```typescript
public install(rsPath: string, rsName: string, cluster: VcClusterComputeResource, defaultHardwareVersion: string, maxHardwareVersion: string, @Out result: any): void {
  const clusterFunctions = System.getModule("com.examples.vmware_aria_orchestrator_examples.actions").clusterComputeResourceManagement();
  const jsonData = System.getModule("com.clouddepth.set_vm_hw_version.actions").getVmHwVersionsResourceElement(rsName, rsPath);
  const maxHardwareVersionKey = System.getModule("com.clouddepth.set_vm_hw_version.actions").getKeyByValue(jsonData, maxHardwareVersion);
  const defaultHardwareVersionKey = System.getModule("com.clouddepth.set_vm_hw_version.actions").getKeyByValue(jsonData, defaultHardwareVersion);
  if (!cluster || !defaultHardwareVersion || !maxHardwareVersion) throw new Error(`Cannot reconfigure the cluster. Required arguments are missing`)
  clusterFunctions.ClusterComputeResourceManagement.prototype.setVmHardwareVersion(cluster, defaultHardwareVersionKey, maxHardwareVersionKey);
}
```

The implementation of `clusterComputeResourceManagement` can be found [here](https://github.com/unbreakabl3/vmware_aria_orchestrator_examples/blob/main/general_examples/src/actions/clusterComputeResourceManagement.ts).

## Final result

### External validation

Let's test our workflow by setting some values and running the workflow. Oh, what do we see? There appears to be an error indicating that the maximum version cannot be lower than the default one. Good catch. This underscores the value of the external validation we implemented for such scenarios—bullet-proof solutions, remember?

![Image](6cb7b925-888d-42a0-a922-6f2eb1e9c2b9.png){: .shadow }{: .normal }

The validation is straightforward and straightforward. We need to provide both default and maximum version keys and compare between them. The validation code can be found [here](https://github.com/unbreakabl3/vmware_aria_orchestrator_examples/blob/main/set_vm_hw_version/src/actions/external_validation/validateVmHwVersion.ts) folder.

![Image](ef4c1ab0-91fb-4c58-bffe-5d4226b3808a.png){: .shadow }{: .normal }
_External validation_

Let's fix it and try again.

![Image](5a4ddeb2-9490-423e-a503-313f5e3aede9.png){: .shadow }{: .normal }

We can see our cluster default setting is updated

![Image](d96712c3-7fff-4853-8d40-3e065fe2c373.png){: .shadow }{: .normal }

>If there is a need to reset all the settings to their defaults
>![Image](7dbebdd8-3ba7-4f5b-be39-8929fede22ba.png){: .shadow }{: .normal }
{: .prompt-tip}

## Summary

To enhance our method further, we could incorporate logic to selectively display only newer versions than a specified default version. This approach introduces a level of refinement by filtering out versions that do not meet the criteria, ensuring users only see relevant updates. While the possibilities for enhancement are boundless, adopting this strategy would establish a strong foundation.

Any feedback is highly appreciated.

## Source Code

The source code with the unit tests can be found [here](https://github.com/unbreakabl3/vmware_aria_orchestrator_examples/tree/main/set_vm_hw_version)
