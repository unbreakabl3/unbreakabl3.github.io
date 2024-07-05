---
layout: post
title: How to use external validations with VMware Aria Orchestrator
date: '2024-03-30'
media_subpath: /assets/img/vro-how-to-use-external-validations/
categories: [VMware, Build Tools, Aria Orchestrator, How To, vRO]
tags: [vmware, building_tools, external_validation]
---

## Problem

Ever crafted a meticulously designed workflow, only to have it fail midway through execution? The ensuing troubleshooting dilemma is familiar: manually complete the remaining steps or dismantle everything and start anew. Often, we design the "commissioning" workflow, neglecting a crucial counterpart – the "decommissioning" process that gracefully handles errors and cleans up created objects.

This is where **[external validation](https://docs.vmware.com/en/VMware-Aria-Automation/8.16/Using-Automation-Orchestrator/GUID-97313488-7EBA-45F3-9A5A-3732ACDADC32.html)** becomes a critical tool. It acts as a proactive safeguard, preventing errors before workflow execution commences. For workflows with numerous inputs, its importance is magnified. Imagine seamlessly validating user existence in Active Directory, remote server availability before REST API calls, or data uniqueness within databases or files. A seemingly trivial omission, like insufficient storage space for significant file creation, can be identified beforehand.

While external validation can't guarantee absolute future success (server outages can still occur), it mitigates initiating workflows destined for failure. This translates to a vastly improved user experience. Instead of cryptic error messages after the fact, users receive clear explanations based on the specific validation performed. This transparency fosters trust and eliminates the frustration of a workflow failing silently in the background.
In essence, external validation serves as a pre-emptive quality check, similar to pre-flight inspections in aviation. By incorporating it, you streamline workflow execution and ensure your users have a positive experience.

External Validation is always an Action Element, which should always return string, in case of problem (the string should contain the message which user will see) or return nothing if the validation pass.
Let's take a simple example that we want to create a new VM. Without the validation, if the VM is already exists, the vCenter will tell it, but it will be to late, because the workflow is already started. Therefore, lets check the VM existence in advance.

## Example

We have a workflow that deploys a simple VM. We need to check if a VM with the provided name already exists before deploying it.

## Solution

The external validation uses an Action Element (cannot be a Workflow), which is triggered before the Workflow starts to run. Let's create one.

### Create Action Element "validateVM"

This function below takes a `vmName` parameter and checks if a virtual machine with that name already exists. It does this by calling the `VcPlugin.getAllVirtualMachines` method, which returns an array of `VcVirtualMachine` objects. The code checks if the array length is greater than zero, which means that a virtual machine with that name exists. If so, the code returns a message saying so.

The `getAllVirtualMachines` method takes two parameters: a filter object and a query specification. In this case, the filter is set to null, which means that all virtual machines are returned, and the query specification is set to an XPath expression that matches the name property of the virtual machine and checks if it matches the given `vmName` parameter.

```typescript
/**
 * Checks if a virtual machine with the given name already exists.
 *
 * @param {string} vmName - The name of the virtual machine to check.
 * @returns {string} - An error message if the virtual machine exists, or undefined if it does not.
 */
(function validateVM(vmName: string) {
  const vms: VcVirtualMachine[] = VcPlugin.getAllVirtualMachines(null, `xpath:name[matches(.,'${vmName}')]`);

  if (vms.length > 0) {
    return `Virtual Machine with that name is already exists`;
  }
});
```

Transpiled code in Javascript

```javascript
/**
 * @type {Array<VirtualMachine>}
 */
var vms = VcPlugin.getAllVirtualMachines(null, "xpath:name[matches(.,'" + vmName + "')]");
if (vms.length > 0) {
    return "Virtual Machine with that name is already exists";
}
```

> JDOCs in the Action Element are parsed by vRBT into, inputs, input description, and the return type as shown below.
{: .prompt-tip }

![action_element](e62af892-258f-4dd5-92d4-bb5ec7b694d9.png){: .shadow }{: .normal }

### Configure Workflow

Our Sample Workflow has an input called `vmName` for a type `string`.
> Sample Workflow is just an example of some workflow that may deploy a VM in your environment, but it doesn't include an actual code for deploying a VM in that example.
{: .prompt-info }
![workflow](318637dc-753c-4b94-a9f0-1fc625101fe9.png){: .shadow }{: .normal }

### Configure External Validation

Open a Workflow > Input Form > Validation. Drag and drop the `Validation Types` to the canvas.
On the right side, fill in the `Validation Label`, `Select Action`, and `Action Inputs` as shown below, and save the workflow.

![workflow](b3d4f436-dfe6-480d-9f2d-5f13a863b643.png){: .shadow }{: .normal }

### Test the external validation

Open the workflow, provide the name of the existing VM `vm01` and run the it. As we can see, the validation throws an error, which shows precisely the error description we wrote in the code.

![workflow](38bd587a-f2be-4d81-af18-56a285a4193c.png){: .shadow }{: .normal }

Now let's run the same workflow, but in that time, we provide the name of the VM, which doesn't exist - `vm02`. As we can see, the workflow passed the validation and was executed successfully.

![workflow](71bdce0a-9499-4ca9-aef4-f82be3fa355e.png){: .shadow }{: .normal }

## Caveats

>The quote below is relevant only to the action elements requiring input from the user. All other use cases where the action element requires inputs passed by another action element or workflow can be written as a normal TS/JS function with its parameters because they don't need to be exposed in vRO GUI.
{: .prompt-tip }
> The external validation method is testing the user's inputs. That means the Action Element must include inputs (at least one). To convert the TS/JS function into an Action Element in vRBT, it is necessary to transform it into a [Self-Executing Anonymous Function](https://developer.mozilla.org/en-US/docs/Glossary/Self-Executing_Anonymous_Function). This means that the function must be anonymous and enclosed in parentheses. However, this also makes testing the function impossible because it must be exported and imported into Jasmine, and the Self-Executing Anonymous Function function cannot be exported.
>
> Despite my efforts, I could not find a way to do so. If you happen to find a solution, please feel free to share it in the comments section.

### Update 1

The problem regarding unit testing has been resolved and the solution can be accessed [here](https://github.com/unbreakabl3/vmware_aria_orchestrator_examples/blob/main/general_examples/src/test/validateVM.test.ts)

## Summary

The external validation can significantly reduce the risk of errors, improve the quality of the user's experience, and ensure that the workflow's outcome meets the user's expectations.

## Source Code

The source code with the unit tests can be found [here](https://github.com/unbreakabl3/vmware_aria_orchestrator_examples/blob/main/general_examples/src/actions/external_validation/validateVM.ts).
