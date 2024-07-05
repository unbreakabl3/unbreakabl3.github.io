---
layout: post
title: How to delete deployed resource automatically with VMware Aria Orchestrator (Automation) - part 3
date: "2024-05-03"
media_subpath: /assets/img/vro-how-to-delete-deployed-resource-automatically-part3/
categories: [VMware, Build Tools, Aria Orchestrator, How To, vRO]
tags: [vmware, building_tools]
---

## Problem

Let's say we have a ready-to-use template in vRA Assembler that deploys a virtual machine. Now, we aim to implement our proof of concept (POC) strategy (where the deployed resource should be automatically destroyed after X amount of time). The vRA GUI has a feature that allows us to apply day2actions to the deployed resource but requires manual intervention. **The critical point of this strategy is to allow a user to choose the decommission date**. We may limit the maximum amount of time the user cannot exceed.

What if we want to automate this process based on certain conditions?

## Solution

To complete this task, we need to be able to modify the lease date of the deployed resource, which is set to *never* by default. To do so, we will use REST API because such functionality doesn't exist OOB. As far as I know, at least :)

Built-in vRO workflows and actions allow the addition of a remote server into the inventory and pre-define the API calls. Those flows can be helpful for some basic tasks because it has some limitations and is not very flexible. I always used some custom functions to execute REST API calls instead of using built-in ones. I found an excellent [example](https://www.simplygeek.co.uk/2019/05/16/vrealize-orchestrator-http-rest-client/) and used it as a template here. It was adjusted a bit to support a TypeScript and converted to [HttpClient class](https://github.com/unbreakabl3/vmware_aria_orchestrator_examples/blob/main/poc_example_for_vra/src/actions/httpClient.ts).

So, what should be done?

1. Create change lease date workflow
2. Configure Subscription in Assembler Extensibility to trigger change lease date workflow

### Create change lease date workflow

#### Configuration Element

To connect with REST to vRA, we require credentials to be stored securely. The ideal place to store these credentials is within a Configuration Element, which enables us to encrypt secrets. However, there is a catch - first, we must store the secret in vRO and then retrieve it from vRO to store it in the Configuration Element. Otherwise, if we go to vRO's GUI and set the values into the config element, they will be overwritten on the next push.

There are two methods to achieve this:

1. Manually create a configuration element in vRO and provide a password as a *SecureString*. Once saved, the password will be encrypted, and we can fetch it.
2. Create a configuration element in vRBT with all the keys and values and push it to vRO; once pushed, we can fetch the encrypted password and store it safely in the vRBT configuration element.

The first option is easy, therefore, lets focus on the second one.
We need to create a new configuration element file named "vraProperties.conf.ts." This file will have three keys: "username," "password" and "hostname". The type of the password is *SecureString* which means that vRO will encrypt it once it is packaged and pushed to vRO.

```typescript
import { Configuration } from "vrotsc-annotations";

@Configuration({
  name: "vra_properties",
  path: "vra",
  attributes: {
    username: {
      type: "string",
      value: "admin",
      description: "vRA username",
    },
    password: {
      type: "SecureString",
      value: "P@ssw0rd",
      description: "vRA password",
    },
    hostname: {
      type: "string",
      value: "vra01.domain.local",
      description: "vRA FQDN",
    },
  },
})
export class VRAProperties {}
```

Push the configuration element to vRO.

>By default, the configuration element values are not pushed to vRO. Only keys. Two switches should be added to fix that:
- *Dvro.packageImportConfigurationAttributeValues=true* - will include the regular values
- *Dvro.packageImportConfigSecureStringAttributeValues=true* - will include SecureStrings
{: .prompt-info}

```bash
mvn -T 10C clean install vrealize:push -Pvra01 -Dvro.packageImportConfigurationAttributeValues=true -Dvro.packageImportConfigSecureStringAttributeValues=true
```

Open the vRealize Developer Tool in VSCode and expand Configurations. We should be able to see the *vra* folder with the *vra_properties* element. The line we're interested in is *the value encoded in the password*. Copy everything inside the CDATA[] > 65BT31...

![Image](20642fff-e53d-43a0-a5b4-c9fb91823917.png){: .shadow }{: .normal }

Paste it into the password value. Since then, each time we push the updated configuration, it will always include encrypted credentials inside.

```typescript
import { Configuration } from "vrotsc-annotations";

@Configuration({
  name: "vra_properties",
  path: "vra",
  attributes: {
    username: {
      type: "string",
      value: "admin",
      description: "vRA username",
    },
    password: {
      type: "SecureString",
      value: "65BT31W71Q33M55I76R01O62I1BI...",
      description: "vRA password",
    },
    hostname: {
      type: "string",
      value: "vra01",
      description: "vRA FQDN",
    },
  },
})
export class VRAProperties {}
```

#### Workflow

Our workflow must communicate with vRA, which requires a Refresh Token and Access Token. Once we have obtained these tokens, we can adjust the lease time. We will convert our API calls to ASYNC calls to communicate with the remote server.

The workflow contains one primary function called *main*, which calls other functions asynchronously.
![Image](da89381e-a88d-4345-9142-3ae0da05f896.png){: .shadow }{: .normal }

Therefore, the logic will be:

1. Get vRA credentials from the configuration element
2. Get refresh token
3. Get access token
4. Get day actions availability
5. Set desired decommission date
6. Confirm the change lease date is updated

##### Get Configuration Element

An implementation of the configuration element is available in the [following example.](https://www.clouddepth.com/posts/vro-configuration-element-how-to/) In this example, we import the username, password, and hostname of the vRA instance using the configuration element we created earlier.

```typescript
const confElementVars = {
  configName: "vra_properties",
  configPath: "vra",
};
const vraConfigElement = config.getConfigElement(confElementVars);
const configElementAttributes = vraConfigElement
  ? config.getConfElementAttributes(vraConfigElement)
  : undefined;
if (!configElementAttributes) {
  throw new Error(
    `Failed to get properties from the configuration element ${confElementVars.configName}`
  );
}
const creds = {
  username: configElementAttributes.username,
  password: configElementAttributes.password,
  hostname: configElementAttributes.hostname,
};
```

Let's create a "functions.ts" file to store the Functions class' external functions. I will only explain some API calls in detail since they are similar and self-explanatory. We'll discuss the main ones.

##### Get vRA refresh token

As I mentioned above, because those are calls to the remote server, we'll use Promises to make our calls asynchronous.

```typescript
public getVraRefreshToken({ username, password, hostname }: { username: string; password: string; hostname: string }): Promise<string> {
  return new Promise((resolve, reject) => {
    const headers = [];
    headers.push({
      key: "Content-Type",
      value: "application/json"
    });
    const jsonContent = {
      username: username,
      password: password
    };
    const restAttr = {
      restUri: "/csp/gateway/am/api/login?access_token",
      contentType: "application/json",
      content: JSON.stringify(jsonContent),
      expectedResponseCodes: [200],
      headers: headers
    };
    const responseContent = new HttpClient(`https://${hostname}`);
    const response: RESTResponse = responseContent.post(restAttr);
    if (response.statusCode >= 400) {
      reject(`Failed to get refresh token. Status code: ${response.statusCode}`);
    } else {
      resolve(JSON.parse(response.contentAsString).refresh_token);
    }
  });
}
```

##### Get vRA Access token

The logic is the same as with the refresh token, but instead of providing a username and password to get the access token, we do provide a refresh token first.

##### Get day two actions

This step is crucial. The day action will only be available after a successful deployment, and we can access it either through the GUI or programmatically.

```typescript
public getVraDayTwoActions(vraHostname: string, deploymentId: string, accessToken: string): Promise<Array<DeploymentDay2Actions>> {
  return new Promise((resolve, reject) => {
    const headers = [];
    headers.push({
      key: "Content-Type",
      value: "application/json"
    });
    headers.push({
      key: "csp-auth-token",
      value: accessToken
    });
    const restAttr = {
      restUri: `/deployment/api/deployments/${deploymentId}/actions`,
      contentType: "application/json",
      expectedResponseCodes: [200],
      headers: headers
    };
    const responseContent = new HttpClient(`https://${vraHostname}`);
    const response: RESTResponse = responseContent.get(restAttr);
    if (response.statusCode >= 400) {
      reject(`Failed to get day 2 actions. Status code: ${response.statusCode}`);
    } else {
      resolve(JSON.parse(response.contentAsString));
    }
  });
}
```

The method above returns an array of all available day2actions for that particular deployment. There are many of them. The output below provides an example. However, we are interested in only one specific action, which is *Deployment.ChangeLease.*

```json
[
  {
    "id": "Deployment.ChangeLease",
    "name": "ChangeLease",
    "displayName": "Change Lease",
    "description": "Set a deployment's expiration date",
    "valid": true,
    "actionType": "RESOURCE_ACTION"
  },
  {
    "id": "Deployment.ChangeOwner",
    "name": "ChangeOwner",
    "displayName": "Change Owner",
    "description": "Change owner of a deployment",
    "valid": true,
    "actionType": "RESOURCE_ACTION"
  },
  {
    "id": "Deployment.ChangeProject",
    "name": "ChangeProject",
    "displayName": "Change Project",
    "description": "Change project of a deployment",
    "valid": true,
    "actionType": "RESOURCE_ACTION"
  },

...

]
```

If the value of the *valid* key for that action is *true*, we can change the lease date. To ensure that the *Deployment.ChangeLease* action is available, we can create a small method to validate it. This method will return *true* if the action is available and *false* if it isn't.

```typescript
public validateChangeleaseAction(changeleaseAction: Array<DeploymentDay2Actions>): boolean {
  return changeleaseAction.some((item) => {
    return item.id === "Deployment.ChangeLease" && item.valid === true;
  });
}
```

##### Set decommission date

Let's add an input called *Decommission Date* to our deployment custom form. Using that input, the user will select when the deployment should be deleted.

![Image](d463aa03-65fb-4cd1-8fa6-948acfbef642.png){: .shadow }{: .normal }

>The logic for showing the decommission date and external validation function can be taken from the [previous article](https://www.clouddepth.com/posts/vro-how-to-delete-deployed-resource-automatically-part2/) and applied here if needed.
{: .prompt-info}

In that function, we're executing a POST call and providing the ID of the deployment we want to change (the ID comes from the subscription input properties) in the body - *Deployment.ChangeLease* and the property in the action we want to update - *Lease Expiration Date*. The call will return a *task ID* (line 40), which will be monitored later.

```typescript
public setLeaseDate({
  vraHostname,
  deploymentId,
  accessToken,
  changeleaseDate
}: {
  vraHostname: string;
  deploymentId: string;
  accessToken: string;
  changeleaseDate: Date;
}): Promise<string> {
  return new Promise((resolve, reject) => {
    const headers = [];
    headers.push({
      key: "Content-Type",
      value: "application/json"
    });
    headers.push({
      key: "csp-auth-token",
      value: accessToken
    });
    const jsonContent = {
      actionId: "Deployment.ChangeLease",
      inputs: {
        "Lease Expiration Date": changeleaseDate
      }
    };
    const restAttr = {
      restUri: `/deployment/api/deployments/${deploymentId}/requests`,
      contentType: "application/json",
      content: JSON.stringify(jsonContent),
      expectedResponseCodes: [200],
      headers: headers
    };
    const responseContent = new HttpClient(`https://${vraHostname}`);
    const response: RESTResponse = responseContent.post(restAttr);
    if (response.statusCode >= 400) {
      reject(`Failed to get access token. Status code: ${response.statusCode}`);
    } else {
      resolve(JSON.parse(response.contentAsString).id);
    }
  });
}
```

##### Confirm change lease date is updated

The change of any of the day2action should pass a few stages before it is applied: PENDING > INITIALIZATION > CHECKING_APPROVAL > INPROGRESS > SUCCESSFUL.
That's why if we immediately check the deployment status, we'll see one of those steps. The example below.

```json
{
  "id": "ddc322cb-6b0d-446f-89a5-399286ff74b3",
  "name": "Change Lease",
  "requestedBy": "admin",
  "actionId": "Deployment.ChangeLease",
  "deploymentId": "9ea77758-e0e4-42a4-9d7b-58969166b4a9",
  "resourceIds": ["9ea77758-e0e4-42a4-9d7b-58969166b4a9"],
  "inputs": {
    "Lease Expiration Date": "2024-10-29T12:27:00.000Z"
  },
  "status": "PENDING",
  "details": "Waiting to start execution",
  "createdAt": "2024-04-25T12:47:35.991504Z",
  "updatedAt": "2024-04-25T12:47:35.991504Z",
  "totalTasks": 1,
  "completedTasks": 0,
  "cancelable": true
}
```

**It's essential not to assume that tasks will always be completed successfully without verifying them. Therefore, we can write a while loop to continuously check the status of the change lease action until it's completed.**

>Please keep in mind that when using the REST API plugin, it may not always be easy to execute ASYNC calls. Additionally, external NPM libraries like FETCH may not always be available. This can result in a 4XX HTTP error code, indicating that the remote host (such as vRA in our example) is unavailable. If this happens, it is recommended that a retry function be implemented to handle these errors.
{: .prompt-tip}

Example of the retry function:

```typescript
let retries = 0;
while (retries < 20) {
  if (vRaGetChangeleaseActionStatusResult !== "SUCCESSFUL") {
    System.log(
      `vRaGetChangeleaseActionStatusResult is ${vRaGetChangeleaseActionStatusResult}`
    );
    vRaGetChangeleaseActionStatusResult = await func.retryPromise(
      () =>
        getDeploymentChangeleaseActionStatus(
          creds.hostname,
          vRaAccessToken,
          vRaSetLeaseDateTaskId
        ),
      20,
      1000,
      false
    );
  }
  System.sleep(1000);
  retries++;
}
```

### Configure Subscription in Assembler Extensibility to trigger change lease date workflow

Let's create a new subscription for the "Deployment Completed" event topic. I selected this topic because updating the lease date at the beginning or during the deployment is unnecessary if the deployment fails. Hence, we should wait until the end of the deployment and only update the lease date if the deployment is successful.
![Image](image.png){: .shadow }{: .normal }

One of the parameters we'll use in our workflow when this topic fires is *deploymentId*. We need to know which deployment to set the lease date for.
![Image](image-1.png){: .shadow }{: .normal }

### Test

Let us create a new catalog item request and deploy some VM.
![Image](image-2.png){: .shadow }{: .normal }

As a result, we have set the expiration date for our deployment, which will expire in 11 days.
![Image](da89381e-a88d-4345-9142-3ae0da05f895.png){: .shadow }{: .normal }

### To think about

In our workflow, we use ASYNC functions. However, vRBT doesn't allow async/await at the top level. If an error occurs within any inner function, we can catch it, print it to the console, and take any necessary action except throwing an error, which would stop the workflow. Because of that, the workflow will always be completed successfully, but we want it to fail if an error occurs. To make this happen, I set a variable (in my example, *workflowExecutionResult*) to *false* if any ASYNC functions throw an error. Then, I check whether the variable is false, and if so, I throw an error and fail the workflow.

If you have or can think of a better solution, please welcome to the comments.

```typescript

main().catch((err) => {
      System.error(`Stack: ${err}`);
      workflowExecutionResult = false;
    });
    if (!workflowExecutionResult) throw new Error("The workflow execution failed");
```

## Summary

Today we saw how we can make any deployed resource be automatically destroyed after the provided amount of time.

## Source Code

The source code with the unit tests can be found [here](https://github.com/unbreakabl3/vmware_aria_orchestrator_examples/tree/main/poc_example_for_vra)
