---
layout: post
title: How to update and remediate hostprofile
date: "2024-06-10"
media_subpath: /assets/img/vro-how-to-update-and-remediate-hostprofile/
categories: [VMware, Build Tools, Aria Orchestrator, How To, vRO]
tags: [vmware, building_tools, hostprofile]
---

## Problem

Sometimes, we need to assign specific tasks to someone, but often, we have to provide detailed instructions like "Don't forget to do A if A happens and B if B happens". For example, "When applying host profile 1, remember to set the hostname in FQDN format and assign the IP address with the subnet mask." However, this approach usually doesn't work in most cases. That's why developing bullet-proof solutions is the best way to minimize human errors. Today, we will explore how to automate host profile management using intelligent techniques, which can be tailored and expanded to any use case.
We're going to create a new workflow which will support the following use cases:

1. See the current host status
2. Update the currently attached profile: hostname, IP address, and subnet mask
3. See all available host profiles
4. Be able to attach another host profile
5. Validate provided inputs

The following diagram shows the logic of the upcoming workflow.

![Image](e6ee2551-ca36-4928-83a9-014f78f900d9.png){: .shadow }{: .normal }{: width="300" height="200" }

> The original size PDF of the diagram can be found [here](https://github.com/unbreakabl3/unbreakabl3.github.io/blob/main/assets/img/vro-how-to-update-and-remediate-hostprofile/hostprofiles-workflow.pdf).
{: .prompt-info}

## Solution

### Host profile preparation

To update the host profile, several variables must be defined first:

- Host profile manager configuration

```typescript
const sdkConnection: VcSdkConnection = vmHost.sdkConnection;
const hostProfileManager: VcProfileManager = sdkConnection.hostProfileManager;
const hostToConfigSpecMap = [];
const vcHostProfileManagerHostToConfigSpecMap =
  new VcHostProfileManagerHostToConfigSpecMap();
const configSpec = new VcAnswerFileOptionsCreateSpec();
configSpec.validating = true;
vcHostProfileManagerHostToConfigSpecMap.configSpec = configSpec;
vcHostProfileManagerHostToConfigSpecMap.host = vmHost;
const userInput: Array<VcProfileDeferredPolicyOptionParameter> = [];
```

- Host name policy configuration

To set the hostname, the _HostNamePolicy_ policyId is used and _hostName_ variable is provided as a hostname.

```typescript
const hostNameInput = new VcProfileDeferredPolicyOptionParameter();
hostNameInput.inputPath = new VcProfilePropertyPath();
hostNameInput.inputPath.policyId = "HostNamePolicy";
hostNameInput.inputPath.profilePath =
  'network.GenericNetStackInstanceProfile["key-vim-profile-host-GenericNetStackInstanceProfile-defaultTcpipStack"].GenericDnsConfigProfile';
const hostNameParameter = new VcKeyAnyValue();
hostNameParameter.key = "hostName";
hostNameParameter.value_AnyValue = hostFQDN;
hostNameInput.parameter = [hostNameParameter];
Input.push(hostNameInput);
```

- IP address policy configuration
  
The same for the _hostSubnet_ subnetmask and _hostIP_ IP address.

```typescript
const ipInput = new VcProfileDeferredPolicyOptionParameter();
const subnetParameter = new VcKeyAnyValue();
subnetParameter.key = "subnetmask";
subnetParameter.value_AnyValue = hostSubnet;
const ipParameter = new VcKeyAnyValue();
ipParameter.key = "address";
ipParameter.value_AnyValue = hostIP;
ipInput.inputPath = new VcProfilePropertyPath();
ipInput.inputPath.policyId = "IpAddressPolicy";
ipInput.inputPath.profilePath = 'network.hostPortGroup["key-vim-profile-host-HostPortgroupProfile-ManagementNetwork"].ipConfig';
ipInput.parameter = [subnetParameter, ipParameter];
userInput.push(ipInput);
configSpec.userInput = userInput;
hostToConfigSpecMap.push(vcHostProfileManagerHostToConfigSpecMap);
```

### Update host with provided values

To do so, we have `updateHostCustomization` method.

```typescript
func.updateHostCustomization(hostToConfigSpecMap, hostProfileManager);
```

The method takes a previously prepared variables and execute `updateHostCustomizations_Task()` method.

```typescript
public updateHostCustomization(hostToConfigSpecMap: Array<any>, hostProfileManager: VcProfileManager) {
  try {
    const task = hostProfileManager.updateHostCustomizations_Task(hostToConfigSpecMap);
    System.getModule("com.vmware.library.vc.basic").vim3WaitTaskEnd(task, true, 2);
    return;
  } catch (error) {
    throw new Error(`updateHostCustomization: ${error}`);
  }
}
```

If everything goes well, we should see the similar logs in console
![Image](08dab0c9-fa5b-4f49-9e7a-fb7c97225850.png){: .shadow }{: .normal }

### Profiles

First of all, to make our life easier (informative), we're showing the currently attached host profile in the custom form. This will help to understand what to do next. To achieve that, we have a small function called `getAttachedProfile`.
This simple function gets _vmHost_ as an input and shows the attached profile.

```typescript
/**
 *
 *
 * @param {VC:HostSystem} vmHost - The name of the ESXi host to check.
 * @returns {string} - Host profile name.
 */
(function getAttachedProfile(vmHost: VcHostSystem): string | null {
  const hostProfileManager: VcProfileManager = vmHost.sdkConnection.hostProfileManager;
  try {
    const profiles: Array<VcProfile> = hostProfileManager.findAssociatedProfile(vmHost);
    return profiles.length > 0 ? profiles[0].name : null;
  } catch (error) {
    throw new Error("Unable to find attached host profile");
  }
});
```

>In vRBT, the object type should be the same as in vRO. For example, in vRBT, it's `VcHostSystem` when used inside the code, but it should be `VC:HostSystem` if we want to use it in the Actions params.
>In addition, we want to show the administrator all available profiles in the vCenter to which the ESXi host is connected. To make it work, we have a function called `getAllAvailableHostProfiles`.
>The ideas are the same, but we return an array of profiles in that case and will show them in the dropdown box.
{: .prompt-info}

```typescript
/**
 *
 *
 * @param {VC:HostSystem} vmHost - The name of the ESXi host to check.
 * @returns {Array/string} - Host profile details
 */
(function getAllAvailableHostProfiles(vmHost: VcHostSystem) {
  const hostProfileManager: VcProfileManager = vmHost.sdkConnection.hostProfileManager;
  const profileDetails: Array<string> = [];
  let customProperties;
  const profiles: VcProfile[] = hostProfileManager.profile;
  if (isArrayNotEmpty(profiles)) {
    profiles.forEach((element) => {
      profileDetails.push(element.name);
    });
  }
  return profileDetails.sort();

  function isArrayNotEmpty<T>(array: T[]): array is [T, ...T[]] {
    return array.length > 0;
  }
});
```

After selecting the profile, we aim to display specific details about that profile in a custom form to assist the administrator in making the proper selection. To achieve this, we have a function called `getHostProfileDetails`. This function retrieves an array of profiles along with their properties (`customProperties`). The critical property, `profileObject_value`, contains a VcProfile object that we will utilize later.

```typescript
/**
 *
 *
 * @param {VC:HostSystem} vmHost - The name of the ESXi host to check.
 * @param {string} hostProfileName
 * @returns {Array/Properties} - Host profile details
 */
(function getHostProfileDetails(vmHost: VcHostSystem, hostProfileName: string): Properties[] {
  const hostProfileManager: VcProfileManager = vmHost.sdkConnection.hostProfileManager;
  const profileDetails: Array<Properties> = [];
  let customProperties;
  let filteredProfile: VcProfile[] = [];
  const profiles: VcProfile[] = hostProfileManager.profile;
  if (isArrayNotEmpty(profiles)) {
    filteredProfile = profiles.filter((profile) => {
      return profile.name === hostProfileName;
    });
  }

  if (isArrayNotEmpty(filteredProfile)) {
    filteredProfile.forEach((element) => {
      customProperties = {
        profileName: element.name,
        profileValidationState: element.validationState,
        profileDescription: element.config.annotation,
        profileObject: element**
      };
      profileDetails.push(customProperties);
    });
  }
  return profileDetails.sort();

  function isArrayNotEmpty<T>(array: T[]): array is [T, ...T[]] {
    return array.length > 0;
  }
});
```

### Profile remediation

To remediate the host profile on the host, we first need to check if the selected profile is the same one that is already attached. If it is, we just remediate the profile with updated values. However, if the selected profile differs, we should first attach it to the host. To make it work, we'll need to follow a few simple steps:

- Check if the selected profile is not empty

```typescript
const selectedProfile: Properties[] = System.getModule("com.clouddepth.host_profiles.actions").getHostProfileDetails(vmHost, availableHostProfiles);
if (!func.isArrayNotEmpty(selectedProfile)) throw new Error(`${selectedProfile} is not an array`);
```

- Get the current profile and selected profile names and compare them. If they are different, we'll attach the selected profile to the host

```typescript
const selectedProfileDetails: Properties = selectedProfile[0];
const attachedProfile: VcProfile | null = func.findAssociatedProfile(vmHost, hostProfileManager);
if (attachedProfile && selectedProfileDetails.profileName != attachedProfile.name) {
  func.associateHostProfile(vmHost, selectedProfileDetails.profileObject);
}
```

- Remediate the host

```typescript
const profileExecuteResult: VcProfileExecuteResult = func.executeHostProfile(vmHost, userInput, selectedProfileDetails.profileObject);
func.applyHostConfig(vmHost, profileExecuteResult);
```

## Final result

In our vCenter we have two host profiles: Host Profile 1 and Host Profile 2
![Image](6325e847-5060-4692-a2b9-56e8b064b7bb.png){: .shadow }{: .normal }

Let's start the workflow and select one of the ESXi hosts to update. We can see that this host already has Host Profile 2 attached. This profile doesn't require additional input.

Although we can view the list of all available profiles.

![Image](14bb619d-f61e-40d5-a9db-e4f32ff43e46.png){: .shadow }{: .normal }

If we don't remember what is Host Profile 2 about, we can select it and see more details.

![Image](f150ebdc-38fb-4929-b210-7994a0dc7e72.png){: .shadow }{: .normal }

Now we can see that this profile is probably related to the vSAN hosts. Let's assume we attach another profile because our host is not going to be a vSAN host anymore. To do so, select from the list of available profiles profile number 1. In addition, we can see some placeholders.

![Image](41b8bcf9-ebd7-40de-9881-119ee1417629.png){: .shadow }{: .normal }

As shown above, this profile is related to the VDI, and we <u>must</u> provide some inputs. As we mentioned at the beginning, we want to make our flows bulletproof flows. We are making our inputs mandatory only if profile number 1 is selected. This can be easily done with a custom form.

![Image](e3b22a3e-a797-4e6d-8756-0f7b2616f9c7.png){: .shadow }{: .normal }

Let's fill the inputs out. As described before, we're using some regular expressions to validate our inputs and prevent human mistakes

![Image](ac9eca66-459f-438e-b6aa-05dd49355276.png){: .shadow }{: .normal }

>Example of Host FQDN input with regular expression
>![Image](02fbb9c4-b848-4040-911b-ea0e61c37327.png){: .shadow }{: .normal }
{: .prompt-tip}

After all the mistakes were fixed, the workflows should run as expected and be completed successfully.

![Image](e3186f13-a4ce-45ec-ab0a-c61321245d05.png){: .shadow }{: .normal }

## Summary

Today, we tried to automate the ESXi host profile remediation by developing a self-service form, which can be safely delegated to the team members. In addition, we are trying to cover our code base with some unit tests to make it as safe and stable as possible.

Any feedback is highly appreciated.

## Source Code

The source code with the unit tests can be found [here](https://github.com/unbreakabl3/vmware_aria_orchestrator_examples/tree/main/host_profiles)
