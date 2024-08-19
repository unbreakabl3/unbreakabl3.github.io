---
layout: post
title: Event-Driven Datacenter with vRO
date: "2024-08-14"
media_subpath: /assets/img/event-driven-datacenter-with-vro/
categories: [VMware, Build Tools, Aria Orchestrator, Aria Automation, How To]
tags: [vmware, building_tools, policy]
---

I like the concept of the Self-Healing, Event-Driven datacenter. The fundamental principle involves automatically responding to predefined events. For instance, suppose we have a logging system (the specific implementation is inconsequential) capable of detecting a predefined issue and initiating an HTTP POST request to an external system. The external remediation system will receive this HTTP call and subsequently take appropriate action. By adhering to this straightforward logic, we can automate a wide range of tasks.
Numerous excellent tools are available to help us achieve this objective. However, many of them are tied to a specific piece of software or vendor. What if we wanted to create a general solution that can cover various use cases? Let’s explore how we can implement this concept using VMware Aria Orchestrator and improve its reliability and scalability.

**The goals:**

- Implement an automated solution for day-to-day occurrences.
- Proactively address security concerns with automatic remediation of potential threats.
- Seamlessly handle a multitude of events simultaneously.
- Empower our workflows with code-driven remediation capabilities.

To reach our goals, we will need to enhance our infrastructure with additional components, including webhook servers capable of receiving HTTP requests and passing it to the message queue. The purpose of the message queue is straightforward: with potentially dozens or even hundreds of alerts co-occurring, we want to ensure that all are noticed. This brings us to the opportunity to leverage a vRO feature called AMQP Policy.

**The use case:**

We want to disconnect the VM from the network, if there is a specific event was occurring.

1. The event was triggered in the Virtual Machine (VM).
2. The event was sent to the Logging server.
3. Logging server then sent a POST API call to the Webhook server, which contained all the necessary details (like a VM’s hostname).
4. The Webhook server validated the received data and sent the message to the queue.
5. In the queue, the message is routed to the specific destination based on the key received from the event (allows multiple types of events to be supported and sent to different queues).
6. The orchestrator monitored the queue and retrieved the message.
7. The orchestrator then executed the workflow, which involved disconnecting the VM from the network.

![Image](image 2.png){: .shadow }{: .normal }

## The solution

### VM

The implementation method will vary depending on the chosen solution for monitoring the OS event. However, in general, most modern centralized logging solutions gather logs from the guest OS (both Windows and Linux) either using a built-in logging solution or by installing certain agents.

### Logging

A central logging system enables the creation of alerts based on specific criteria. Therefore, you can define your own alerts based on your needs.

### Webhook server

This task may require finding an existing solution or creating your own. Luckily, there are numerous ways to do this nowadays. A straightforward solution could be to use [FastAPI](https://fastapi.tiangolo.com/). It is possible to make a small container to run an API server that receives HTTP requests and passes them to the queue using the AMQP protocol.

### Message Queue

One of the most popular message queue services is RabbitMQ. It is fully compatible with vRO. Therefore, we’ll use it as a queue server.
Deploy a RabbitMQ server and create a new queue `myRemoteQueue` with all the defaults.

![Image](image.png){: .shadow }{: .normal }

### vRO

Now, we can start to configure our vRO:

1. Broker configuration
2. Remediation implementation
3. AMQP policy configuration

#### Broker configuration<!-- {"fold":true} -->

In vRO, find a workflow called “Add a broker” and run it. Give it some name.

![Image](image 4.png){: .shadow }{: .normal }

Provide a message queue server FQDN, port, and credentials, and configure the certificate based on your environment. Run the workflow.

![Image](image 3.png){: .shadow }{: .normal }

In vRO, find a workflow called “Subscribe to queues”, run it, and give it a name.

![Image](image 5.png){: .shadow }{: .normal }

Select the previously defined broker.

![Image](image 6.png){: .shadow }{: .normal }

Provide the name of a queue that was previously configured. Run the workflow.

![Image](image 7.png){: .shadow }{: .normal }

We will see our new message queue server in the vRO inventory if everything goes well.

![Image](Screenshot 2024-08-12 at 2.26.10 PM.png){: .shadow }{: .normal }

#### Remediation implementation

The process of disabling the vNIC is straightforward. First, we locate the VM we want to disconnect from the network. The VM name is received as part of the payload from the logging server. Once found, it is returned as an array of objects. After that, we identify all the network adapters for that VM and set them to disconnect.

Therefore, we have a `disableVirtualNetworkAdapter` function, which is actually disconnect the vNIC.

```javascript
var devices = vm.config.hardware.device;
for (var i in devices) {
  if (
    !System.getModule("com.vmware.library.vc.vm.network").isSupportedNic(
      devices[i]
    )
  ) {
    continue;
  }
  var confSpec = new VcVirtualDeviceConfigSpec();
  confSpec.operation = VcVirtualDeviceConfigSpecOperation.edit;
  confSpec.device = devices[i];
  confSpec.device.connectable.startConnected = false;
  confSpec.device.connectable.connected = false;
  var spec = new VcVirtualMachineConfigSpec();
  spec.deviceChange = [confSpec];
  try {
    vm.reconfigVM_Task(spec);
  } catch (error) {
    System.error("Error reconfiguring VM: " + error);
  }
}
```

#### AMQP policy configuration

Create a new policy. Select the “AMQP:Subscription”, select “OnMessage”, find the workflow, and select “event.key.body” as a payload. The body of the message will be passed on to the workflow’s input.
Save the policy.

![Image](Screenshot 2024-08-12 at 4.45.25 PM.png){: .shadow }{: .normal }

We have created a new policy, but it’s not running yet.

![Image](Screenshot 2024-08-12 at 4.49.09 PM.png){: .shadow }{: .normal }

Click on the policy and hit the RUN button.

![Image](Screenshot 2024-08-12 at 4.49.47 PM.png){: .shadow }{: .normal }

Provide the name and select the message queue we defined previously in Policy Element 1. Run the policy.

![Image](Screenshot 2024-08-12 at 4.51.51 PM.png){: .shadow }{: .normal }

## Test

Let’s connect to a Linux VM and execute the following command. This will generate a log in SYSLOG containing that record. We have configured our logging system to filter and capture all messages related to this specific event.

![Image](Screenshot 2024-08-12 at 4.22.53 PM.png){: .shadow }{: .normal }

The central logging system will capture the event and send it to the webhook server, which will then parse the message and forward it to the message queue.

![Image](Screenshot 2024-08-12 at 5.26.57 PM.png){: .shadow }{: .normal }

When a new message appears in the queue, it will initiate a workflow in vRO. The details of the message will be passed as input into the payload of the workflow. The workflow 'Disable vNIC' will then parse the payload to locate the hostname on which it needs to perform tasks. Once the VM is found, the workflow will call our `disableVirtualNetworkAdapter` function.

![Image](Screenshot 2024-08-15 at 11.39.06 AM.png){: .shadow }{: .normal }

In a received payload we can see some details. The most important one is the hostname of the VM need to look for.

![Image](Screenshot 2024-08-12 at 4.19.31 PM.png){: .shadow }{: .normal }

If everything went well, in the vCenter, we will see that the VM was reconfigured.

![Image](Screenshot 2024-08-12 at 4.16.21 PM.png){: .shadow }{: .normal }

In the network section of the VM settings, both “Connected” and “Connect At Power On” are disabled now.

![Image](Screenshot 2024-08-12 at 4.24.42 PM.png){: .shadow }{: .normal }

## Summary

Following the provided example, we can effectively solve almost any problem using the existing tools with minimal modifications.

Any feedback is highly appreciated.

## Source Code

The source code with the unit tests can be found [here](https://github.com/unbreakabl3/vmware_aria_orchestrator_examples/tree/main/policy_management/src) and [here](https://github.com/unbreakabl3/vmware_aria_orchestrator_examples/blob/main/general_examples/src/actions/virtualNetworkManagement.ts).

The vRO packages are also available [here](https://github.com/unbreakabl3/vmware_aria_orchestrator_examples/blob/main/policy_management/com.clouddepth.policy_management-1.0.6.package) and [here](https://github.com/unbreakabl3/vmware_aria_orchestrator_examples/blob/main/general_examples/com.examples.vmware_aria_orchestrator_examples-1.0.21.package).
