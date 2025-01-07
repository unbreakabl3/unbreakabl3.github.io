---
layout: post
title: Programmatically manage triggered alarms in vCenter with vRO
date: "2024-12-23"
media_subpath: /assets/img/vro-vcenter-alarm-manager/
categories: [VMware, Build Tools, Aria Orchestrator, How To, VMware Cloud Foundation Orchestrator]
tags: [vmware, building_tools, alarm_manager, vcenter]
---

If you ever tried to manage vCenter's alarms with vRO, you probably knows how interesting this rabbit hole can be. Today, we're going to see, how to acknowledge a specific alert in a vCenter with vRO and even try to make it as a pro, using some development best practices.

PS. As often happens, a quick idea for a post tends to grow as I dive into it! 😊 In this case, we’ll use vRBT to write our code in TypeScript and introduce a new `AlarmManagement` class. Along the way, we’ll aim to follow some best practices—highly recommend Uncle Bob’s [lectures](https://www.youtube.com/watch?v=7EmboKQH8lM&list=PLmmYSbUCWJ4x1GO839azG_BBw8rkh-zOj) for inspiration. This includes implementing a dedicated class for error handling, utilizing private methods, and more.

General goals:

* Programmatically acknowledge a specific triggered on ESXi hosts.
* Clear alarms of specific type.

The use case:

* Certain configuration activities may trigger alarms during execution, which we aim to handle appropriately.  
* Specific alarms, such as those caused by periodic actions like backups, require special consideration.

## The solution

Let's take an example where we want to acknowledge a specific alarm and explore it in detail.

### Acknowledge a specific alarm

#### Get triggered alarms

We begin with a method called `getTriggeredAlarms`. The alarm manager is a vCenter-level entity, which means in vRO, we must retrieve it through the SDK connection. To do this, we first need to obtain the alarm manager from the SDK object.

```typescript
if (!vCenter) {
  throw this.handleError("No vCenter connections available.");
}

const alarmManager = vCenter.alarmManager;
if (!alarmManager) {
  throw this.handleError("Alarm Manager not found.");
}
```

Now, we can get all ESXi hosts. Of course they can be filtered by some criteria, but for now, we'll just get all of them.

```typescript
const objectsToCheck = VcPlugin.allHostSystems;
if (!objectsToCheck || objectsToCheck.length === 0) {
  throw this.handleError("No host systems available to check.");
}
```

Let's create a helper function to build an object containing key alarm properties, making it easier to describe the current situation. Our goal is to fetch all alarms that are in a `red` or `yellow` state and are `unAcknowledgedAlarms`. Additionally, we want to extract basic details such as `alarmName`, `status`, `time`, and `alarmObject` to show them later in the logs and use in other functions. To do so, we can use a `filter` method to get only the alarms we're interested in and then `map` to extract the necessary information and contract it into the `alarmStates` object.

```typescript
const extractTriggeredAlarms = (managedObject, alarmStates) => {
  return alarmStates
    .filter((state) => state.overallStatus === VcManagedEntityStatus.red || state.overallStatus === VcManagedEntityStatus.yellow)
    .map((state) => ({
      vcHost: managedObject,
      alarmName: state.alarm.info.name,
      status: state.overallStatus.value,
      time: state.time,
      alarmObject: state.alarm
    }));
};
```

Now, we’re ready to retrieve all triggered alarms based on the pre-defined filter. Let's iterate through all ESXi hosts (`objectsToCheck`) and get the alarms for each of them. We'll use the `getAlarmState` method to get the alarm states for a specific managed object. If the alarm states are available, we'll filter them to get only the unacknowledged alarms. If the alarms are found, we'll extract the triggered alarms and push them to the `triggeredAlarms` array.

```typescript
const triggeredAlarms = [];
objectsToCheck.forEach((managedObject) => {
  try {
    const alarmStates: Array<VcAlarmState> = alarmManager.getAlarmState(managedObject);
    const unAcknowledgedAlarms: Array<VcAlarmState> = alarmStates.filter((state) => state.acknowledged === false);
    if (alarmStates) {
      triggeredAlarms.push(...extractTriggeredAlarms(managedObject, unAcknowledgedAlarms));
    }
  } catch (error) {
    System.warn(`Failed to get alarm states for ${managedObject.name}: ${error}`);
  }
});

return triggeredAlarms;
```

#### Process Alarms

With our alarms in hand, we can begin processing them. Since we’re working with an array of alarms, the looping will be handled in our main code. Each iteration will invoke the `processAlarms` method. Let’s take a closer look at it.

```typescript
public processAlarms(vCenter: VcSdkConnection, vcHost: VcManagedEntity, alarmName: string, alarm: VcAlarm): void {
  const alarmManager: VcAlarmManager = vCenter.alarmManager;
  const alarmToAcknowledge: VcAlarm | undefined = this.getAlarmByName(alarm, alarmName);
  if (alarmToAcknowledge) {
    this.acknowledgeSpecificAlarm(alarmManager, vcHost, alarmToAcknowledge);
  } else {
    System.warn(`Alarm with name ${alarmName} not found.`);
  }
}
```

First, to acknowledge a specific alarm, we need to locate it. We achieve this using the helper method `getAlarmByName`. This method takes the alarm object and the alarm name as arguments.

```typescript
private getAlarmByName(alarm: VcAlarm, alarmName: string): VcAlarm | undefined {
  return alarm.info.name === alarmName ? alarm : undefined;
}
```

If the alarm is found, we acknowledge it using the `acknowledgeAlarm` method. If the alarm is not found, we log a warning message.

```typescript
private acknowledgeSpecificAlarm(alarmManager: VcAlarmManager, vcHost: VcManagedEntity, alarm: VcAlarm): void {
  try {
    alarmManager.acknowledgeAlarm(alarm, vcHost);
  } catch (error) {
    this.handleError(`Failed to acknowledge alarm ${alarm.info.name} on ${vcHost.name}: ${error}`);
  }
}
```

#### Clear triggered alarms

Now, let’s move on to clearing alarms. We’ll create a method called `clearAlarms` to clear alarms based on the alarm type, trigger type, and status. We’ll use the `VcAlarmFilterSpec` object to filter the alarms based on the specified criteria. The `clearTriggeredAlarms` method will be used to clear the alarms. This method takes the `VcAlarmFilterSpec` object as an argument and its a bit limited. We can only filter by `typeEntity`, `typeTrigger`, and `status`.

`entityType` and `triggerType` are enums that define the entity and trigger types, respectively. The `status` is an array of `VcManagedEntityStatus` values that define the status of the alarms to be cleared.

`entityType` can be one of the following:

* `entityTypeAll`
* `entityTypeVm`
* `entityTypeHost` (our case)

`triggerType` can be one of the following:

* `triggerTypeMetric` (our case)
* `triggerTypeAll`
* `triggerTypeEvent`

`status` can be one of the following:

* `green`
* `yellow` (our case)
* `red` (our case)
* `gray`

To clear **all** triggered alarms on ESXi hosts, we can combine the values mentioned above in various ways. For instance, if we want to clear all `red` alarms (status: `red`) on ESXi hosts (type: `entityTypeHost`) caused by high CPU utilization (type: `triggerTypeMetric`), we can use the following code:

```typescript
private clearAlarm(
  alarmManager: VcAlarmManager,
  entityType: VcAlarmFilterSpecAlarmTypeByEntity,
  triggerType: VcAlarmFilterSpecAlarmTypeByTrigger,
  status: VcManagedEntityStatus
): void {
  var vcAlarmFilterSpec = new VcAlarmFilterSpec();
  vcAlarmFilterSpec.typeEntity = entityType.entityTypeHost;
  vcAlarmFilterSpec.typeTrigger = triggerType.triggerTypeMetric;
  vcAlarmFilterSpec.status = [status.red];

  try {
    alarmManager.clearTriggeredAlarms(vcAlarmFilterSpec);
  } catch (error) {
    this.handleError(`Failed to clear triggered alarms: ${error}`);
  }
}
```

The `clearAlarm` method can serve as an alternative to the `clearTriggeredAlarms` method, allowing alarms to be cleared based on specific criteria, such as those defined in the `processAlarms` method. It can also be adapted for other implementations depending on the requirements, enabling us to either acknowledge or clear the alarm.

### Bonus: Error handling

We can create a class to handle errors. This class will contain a method called `handleError` that will log the error message and throw an exception. This way, we can ensure that all errors are handled consistently.

```typescript
export class AlarmManagementError extends Error {
  constructor(message: string) {
    super(message);
    this.name = "AlarmManagementError";
  }
}
```

Now, we can use this class in our main `AlarmManagement` class to handle errors.

```typescript
  private handleError(errorDescription: string): never {
    throw new AlarmManagementError(`Error: ${errorDescription}.`);
  }

  // Usage
  this.handleError(`Error description: ${error}`);
```

## Testing the solution

Let's create a simple workflow to test our solution. We'll create a new workflow called `Acknowledge Alarm` and add an input parameter of type `string`  called `alarmName`. We'll also add a scriptable task to the workflow and write the following code:

```javascript
var vCenter = VcPlugin.AllSdkConnections[0];

var alarmManagementFunctions = System.getModule("com.examples.vmware_aria_orchestrator_examples.actions").alarmManagement();
var triggeredAlarms = alarmManagementFunctions.AlarmManagement.prototype.getTriggeredAlarms(vCenter)
for (var k = 0; k < triggeredAlarms.length; k++) {
    var alarm = triggeredAlarms[k];
    System.log("Triggered Alarm on " + alarm.vcHost.name + ": " + alarm.alarmName + " (" + alarm.status + ")");
    System.log("Time: " + alarm.time);
    
    alarmManagementFunctions.AlarmManagement.prototype.processAlarms(vCenter, alarm.vcHost, alarmName, alarm.alarmObject);
}
```

The logic is straightforward but sufficient for testing our solution. We retrieve all triggered alarms, log them, and then acknowledge them. The key aspect is leveraging the `AlarmManagement` class to handle the functionality. We define `alarmManagementFunctions` as a new instance of the `AlarmManagement` class and invoke the `getTriggeredAlarms` and `processAlarms` methods.

![Image](Screenshot 2025-01-06 at 19.57.46.png){: .shadow }{: .normal }

We have defined a new test alarm called `A test alarm 2` in vCenter.
![Image](Screenshot 2025-01-06 at 20.01.37.png){: .shadow }{: .normal }

We can now run the workflow and see the results. If the alarm is triggered, we should see it in the logs and then acknowledge it.
![Image](Screenshot 2025-01-06 at 20.00.45.png){: .shadow }{: .normal }
![Image](Screenshot 2025-01-06 at 20.05.03.png){: .shadow }{: .normal }

In case of failure, we'll see a nice and descriptive error message in the logs including the module name `AlarmManagementError`, which is very helpful for debugging.:

```log
errorError in (Dynamic Script Module name : alarmManagement#28) AlarmManagementError: Error: No vCenter connections available..
```

## Summary

In this post, we explored how to programmatically manage triggered alarms in vCenter with vRO. We created a new `AlarmManagement` class and introduced a few best practices, such as creating a class for errors handling, using private methods, and more. We acknowledged a specific alarm and cleared alarms of a specific type. We also discussed the use case and general goals of the solution.

## Source Code

The source code with the unit tests can be found [here](https://github.com/unbreakabl3/vmware_aria_orchestrator_examples/tree/main/general_examples/src).

The vRO package is also available [here](https://github.com/unbreakabl3/vmware_aria_orchestrator_examples/raw/refs/heads/main/general_examples/com.examples.vmware_aria_orchestrator_examples-1.0.42.package) and the external ECMASCRIPT package [here](https://github.com/unbreakabl3/vmware_aria_orchestrator_examples/raw/refs/heads/main/general_examples/com.vmware.pscoe.library.ecmascript-3.1.1.package).

> All packages should be imported
{: .prompt-info}
