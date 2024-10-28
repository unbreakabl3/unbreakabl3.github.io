---
layout: post
title: How to delete deployed resource automatically with VMware Aria Orchestrator - part 2
date: "2024-04-19"
media_subpath: /assets/img/vro-how-to-delete-deployed-resource-automatically-part2/
categories: [VMware, Build Tools, Aria Orchestrator, How To, vRO]
tags: [vmware, building_tools]
---

## Custom Form

Aria Build Tools allows us to create a custom form using the JSON file.
>Ensure the JSON file's name is the same as the workflow. We can create multiple workflows with their custom forms inside the same projects using this pattern. For example: *workflow1.wf.ts will always come with workflows1.wf.form.json and workflow2.fw.ts will come with workflow2.wf.form.json*
{: .prompt-info}

The custom form must have the following structure:

![Image](20aad31d-b730-4936-95f1-e3a89c636e5f.png){: .shadow }{: .normal }{: width="300" height="200" }

It may be complicated to create a custom form from scratch without knowing which keys, values, and structure it's expected to be. Fortunately, there is an excellent way, at least for the beginning - to create and customize a custom form in vRO's GUI as we want, export it, and paste it into our JSON. Since then, this JSON can be tracked in GIT and quickly restored in case some changes were made by mistake via the vRO's GUI.

### Design custom form in vRO

Open our POC Example workflow in vRO. The expected outcome should be as follows.

![Image](a90e4261-7ed9-45bb-9cb5-70f894351823.png){: .shadow }{: .normal }

Now, lets customize each one of the inputs.

![Image](7f3bd863-5ead-41c3-b0e9-9e1c05647105.png){: .shadow }{: .normal }

We want to show some inputs only if the "Is POC?" checkbox is checked.

![Image](22789b8d-fd1a-4bbf-af0f-1f89adc5de17.png){: .shadow }{: .normal }

We set the decommission delay to Read-Only because this is our minimum graceful period, and users should not change it.

![Image](a4a470bf-3780-4107-99f3-197ae47dd770.png){: .shadow }{: .normal }

![Image](742fef22-fbb1-49c5-ac58-604deb0d9bc9.png){: .shadow }{: .normal }

>As we can see, the input fields we set as required in the workflow file was indeed marked as required.
> ![Image](0a301f04-5582-432e-a4f3-6a2caa74af22.png){: .shadow }{: .normal }
{: .prompt-info}

Save the changes and test the behavior.

![Image](F2B71A48-1B0F-41D3-9E32-3E2A4FD47F59.gif){: .shadow }{: .normal }

### Export custom form configuration

After making changes, we can export them as YAML by going to Workflow > Version History and collapse the fields. The result is nearly identical to what we expect.

![Image](e7472a56-91f9-4b1d-af45-480d0432c96b.png){: .shadow }{: .normal }

Copy everything between *inputForms*  and *workflowSchema*. Convert YAML to JSON using any available tool and save it into *poc.wf.form.json*.

### External Validation

I always strive to create solutions that are as error-free as possible, anticipating any possible user mistakes in advance. For instance, in our current scenario, one of the errors a user could make is to input an invalid decommission date that has already passed. To prevent this, we can create an external validation action that verifies the provided date against the current date. If the date is in the past, we can issue a clear and easy-to-understand warning message to the user.

To the VSCode!

#### Write a function

We need to create a simple function that takes a date from the user and compares it to the current date.

```typescript
/**
 * Checks if the provided date is not in the past
 *
 * @param {Date} providedDate - The name of the virtual machine to check.
 * @returns {string} - An error message if the provided date is in the past
 */
(function validateDecommissionDate(providedDate: Date) {
  const dateTime = new Date();
  if (providedDate && providedDate <= dateTime) {
    return `Provided decommission date cannot be in the past`;
  }
});
```

Push the package with a new function to vRO and confirm the result.

```shell
mvn -T 10C clean install vrealize:push -Pvro01
```

![Image](a8f295ca-96a3-479e-b88c-422f138e6ba5.png){: .shadow }{: .normal }

#### Configure External Validation

Open the workflow and create new validation using our new function *validateDecommissionDate*.
>We can highlight specific files if the error occurs. This can be a handy UI tip, which can help users identify where exactly the problem is, especially if there are many inputs.
{: .prompt-tip}

![Image](9876c44a-5df4-4131-9834-0c01e3e299c7.png){: .shadow }{: .normal }

#### Run the test

Pick up any date in the past. Vivia, we got an error with an exact error description we wrote, and the proper field is highlighted with a bold red line.
![Image](84dbdd09-db58-4bd2-8b03-2ae320bfb2a8.png){: .shadow }{: .normal }

#### Update the custom form JSON

After implementing the validation, we had to modify our custom form JSON to avoid overwriting the changes made through the GUI during the next package push. To accomplish this, we should check the workflow Version History to identify what has changed since our last modification. Luckily, the vRO GUI allows us to perform this task quickly. We can see the configuration on the left side before we added the validation and on the right side after.

![Image](18a12fb8-75a0-4f7f-9cf4-0954897c4fdd.png){: .shadow }{: .normal }

As we can see, we have an updated section on the right called *options*, which now has values (is empty by default). 'Options' - is the name of the External Validation in the vRBT.

 The rest of the process is the same - copy, convert from YAML to JSON, and paste into the *poc.wf.form.json*.

### Summary

In today's session, we focused on enhancing the quality and reliability of our workflow. Specifically, we delved into External Validation, which helped us achieve a more aesthetically pleasing and error-free outcome.

### Next step

In part three, we will try to see how to achieve similar functionality in Aria Automation.

## Source Code

The source code with the unit tests can be found [here](https://github.com/unbreakabl3/vmware_aria_orchestrator_examples/tree/main/poc_example).

The vRO package is also available [here](https://github.com/unbreakabl3/vmware_aria_orchestrator_examples/raw/refs/heads/main/poc_example/com.clouddepth.poc_example-1.0.9.package) and the external ECMASCRIPT package [here](https://github.com/unbreakabl3/vmware_aria_orchestrator_examples/raw/refs/heads/main/poc_example/com.vmware.pscoe.library.ecmascript-2.43.0.package).

> All packages should be imported
{: .prompt-info}
