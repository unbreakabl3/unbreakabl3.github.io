---
layout: post
title: VMware Cloud Foundation Orchestrator User Interaction - part 1
date: "2025-02-01"
media_subpath: /assets/img/vro-user-interaction/
categories: [VMware, Broadcom, VMware Cloud Foundation Automation, VMware Cloud Foundation Orchestrator, Build Tools, How To, User Interaction]
tags: [vmware, vro]
---
Have you ever wished that instead of failing, a workflow could prompt you for a missing value? That would be pretty cool, right? Today, we'll discuss one of vRO's often overlooked features: "User Interaction." This feature is incredibly useful and has many practical applications. Two major use cases stand out:

1. If there's a decision point in the workflow, someone should be able to approve it before it proceeds.
2. If a variable has a null value, it would be helpful to allow someone to provide that value instead of causing the entire workflow to fail.

Enhancing user experience in this way would be a significant improvement!

## Introduction

User Interaction is a feature introduced in vRO 7 that allows workflows to pause and request input from users. This feature is particularly useful for making workflows more interactive and user-friendly. A key aspect of User Interaction is its support for AD/LDAP groups and users, which enables the assignment of approval tasks to specific individuals or groups. If no one approves the task within a specified time frame, the workflow will fail after a predetermined duration that can be defined. For example, if a workflow may be triggered over the weekend and no one is available to approve the task, we can set it to fail after 48 hours.

> If the workflow is in a `waiting` state, it will remain in that state even if vRO is powered off or restarted, as this information is stored in the database.
{: .prompt-tip}

We will explore two methods for creating a workflow that involves user interaction. The first method utilizes the User Interaction element within the web client and REST API, while the second method involves Javascript/Typescript coding to achieve the same goal. In this section, we will focus on the web client aspect.

## How to use User Interaction

User Interaction is an an canvas element that can be added to the workflow. It has the following properties:

| Property Name       | Type            | Description                                      |
|---------------------|-----------------|--------------------------------------------------|
| security_assignees  | array/LdapUser (can be empty)  | List of LDAP user/s allowed to respond to the user interaction |
| security_assignee_groups     | array/LdapGroup (can be empty) | List of LDAP group/s allowed to respond to the user interaction |
| security_group    | LdapGroup (can be empty) | LDAP group allowed to respond to the user interaction. |
| timeout_date     | Date (can be empty) | Absolute date, until which the workflow waits for a user to respond to a user interaction  |

This element can be integrated into the workflow and set up to request input from the user. The workflow will pause at this element and wait for the user to provide their input. Users can submit their input through the vRO web client or via the REST API.

## Use case

We have a workflow that relies on a variable named `var_0`. The workflow will terminate if the value of `var_0` is **not** `123`. However, if `var_0` is equal to `123`, the user will be prompted to provide a different value.

> The following use case is simplified to illustrate the concept of User Interaction. In reality, it will be more complex and involve additional steps.
{: .prompt-info}

## Standalone vRO

### Using the Web Client

Lets create a workflow and make it look like this:

![image](image.png){: .shadow }{: .normal }
_User Interaction workflow example_

Let's create a variable named `var_0` and focus on it. In our example, we want to ensure that this variable does not hold the value `123`. If it does, we will prompt the user to provide a different value. To implement this, we will use a Decision element. Inside the Decision element, we will add straightforward logic:

```javascript
if (var_0 === 123) {
    return true;
}
```

In this scenario, we are stating that if the value of `var_0` is `123`, we should return `true`, allowing the workflow to proceed. Before initiating the User Interaction, it's essential to inform users that they must provide value by emailing them. For this purpose, we will utilize a built-in vRO workflow called `Send notification (TLSv1.2)`. While I won't detail this workflow but use it, we need to supply a few values, such as the SMTP host and authentication credentials, if required. The only aspect I want to highlight is the content of the email.

#### Email content

Create a new `Email Preparation` scriptable task and add the following code:

```javascript
var executionId = workflow.id
userInteractionLink = "https://vro/orchestration-ui/#/runs/" + executionId
content =
    "Dear user,\n\n" +
    "You have a pending user interaction in vRealize Orchestrator.\n\n" +
    "Click the link below to respond:\n" + userInteractionLink + "\n\n" +
    "Regards,\nvRO Automation Team"
```

`workflow` will return the currently running workflow as an object, with the property `id` representing the unique ID of the current execution. We can use this ID to construct a `userInteractionLink` link. When the user clicks the link, they will be directed to the vRO web client, where they can provide the input.

Once we have all the details to send an email, we can forward it to the `Send Notification (TLSv1.2)` workflow. The email will provide the user with a link to the User Interaction.

> You might ask, "Who will receive that email?" Most likely, it should be directed to the person who can answer the question. However, that's not always the case. For instance, a manager might receive the email and delegate it to someone else. We simply need to ensure that this individual is a member of a specific group, like `security_group`. In many instances, the same person will be responsible. I'm not a fan of hardcoding solutions, which means we shouldn't hardcode the recipient's email address. Furthermore, if the assignee is a group of users, what should we do then? In both situations, it's possible to programmatically retrieve the user's email address attribute on the fly or use an existing distribution group.
>
>Dynamic user's email fetching could be a topic for another post, so feel free to reach out if you want me to cover it.
{: .prompt-tip}

When the email is sent, we should receive a response like this:

![image](image-1.png){: .shadow }{: .normal }

#### User Interaction element

Now, let's add the User Interaction element to the workflow. We need to configure it to ask the user for the value of `var_0`. The User Interaction element has the following properties:

![image](image-2.png){: .shadow }{: .normal }

Under the Options section, we need to create and bind all the variables. In the Form Inputs section, we need to add the variable `var_0` because that is the variable we want the user to provide. That is an important step. Without doing that, the User Interaction will do nothing. And, of course, we can customize it using the Edit Input Form to provide more details to the user about this request.

![image](image-3.png){: .shadow }{: .normal }

Ultimately, we want to print an updated value of `var_0` to the log. We need to add a Scriptable Task called "Print updated value" to do that.

![image](image-5.png){: .shadow }{: .normal }

### Testing

Now, let's trigger the workflow and see how it works. The current value of `var_0` is `123`. The workflow should continue: prepare an email and send it to the user. As we can see, we have the label "Fix me" and the description, which should help the user understand what is happening.

![image](image-6.png){: .shadow }{: .normal }

We should provide value and click "Answer". The workflow needs to continue, and the updated value of `var_0` will be printed in the log.

![image](image-8.png){: .shadow }{: .normal }

Once the execution is finished, we will see in the logs the following:

```text
INFO Current value of 'var_0': 123
INFO Updated value of 'var_0': 777
```

> Because we don't know when the workflow will be triggered, we can calculate the relative timeout for the user interaction. We can do this by adding a Scriptable Task element before the User Interaction element and setting the `timeout.date` variable to the current date plus a specified duration. For example, if we want the user interaction to expire in 48 hours, we can set the `timeout.date` property to the current date plus 48 hours.
> ```javascript
timerDate = new Date();
timerDate.setTime( timerDate.getTime() + (86400 * 1000) ); //ms
System.log( "User Interaction will expire at '" + timerDate + "'" );
```
{: .prompt-tip}

### REST API

Let's talk about APIs. We can use the REST API to obtain information about user interactions and provide input. The first step is to retrieve the interactions related to the workflow. We can accomplish this by sending a GET request to the following URL:

```bash
curl -X 'GET' \
  'https://vro/vco/api/workflows/389a01cd-ac99-4180-93d1-577ec97bad47/interactions' \
  -H 'accept: application/json' \
  -H 'Authorization: Basic ABCD...'
```

> Here, `389a01cd-ac99-4180-93d1-577ec97bad47` is the workflow ID, which can be obtained from the Summary page of the workflow.
{: .prompt-tip}

As a result, we'll get a list of interactions. Each interaction has a link to the interaction itself. We can get the information about the interaction by sending a GET request to the interaction URL.

```json
{
  "link": [
    {
      "attributes": [
        {
          "value": "User Interaction Example : User interaction",
          "name": "name"
        },
        {
          "value": "WAITING",
          "name": "state"
        },
        {
          "value": "8a7480f894ac90710194cba0b1c10394",
          "name": "id"
        },
        {
          "value": "2025-02-03T11:44:53.696Z",
          "name": "creationDate"
        }
      ],
      "href": "https://vro/vco/api/workflows/389a01cd-ac99-4180-93d1-577ec97bad47/executions/6a50ed53-42fb-4bce-bb57-6dd3ffde73af/interaction/",
      "rel": "interaction"
    }
  ],
  "total": 1
}
```

In that case, we have only one interaction represented by `href` key on line 22. We can get the information about the interaction by sending a GET request to the interaction URL.

```bash
curl -X 'GET' \
  'https://vro/vco/api/workflows/389a01cd-ac99-4180-93d1-577ec97bad47/executions/6a50ed53-42fb-4bce-bb57-6dd3ffde73af/interaction' \
  -H 'accept: application/json' \
  -H 'Authorization: Basic ABCD...'
```

From the response below, we can see the information about the interaction. We can see the input parameters, the assignees, the interaction state, and the link to the interaction itself. We want to provide input for that interaction. In that case, we can send a POST request to the interaction URL and provide the input parameters, which we can dynamically get from the response under the `input-parameters` key and construct the request.

```json
{
  "href": "https://vro/vco/api/workflows/389a01cd-ac99-4180-93d1-577ec97bad47/executions/6a50ed53-42fb-4bce-bb57-6dd3ffde73af/interaction/",
  "relations": {
    "link": [
      {
        "href": "https://vro/vco/api/workflows/389a01cd-ac99-4180-93d1-577ec97bad47/executions/6a50ed53-42fb-4bce-bb57-6dd3ffde73af/",
        "rel": "up"
      },
      {
        "href": "https://vro/vco/api/workflows/389a01cd-ac99-4180-93d1-577ec97bad47/executions/6a50ed53-42fb-4bce-bb57-6dd3ffde73af/interaction/presentation/",
        "rel": "down"
      },
      {
        "href": "https://vro/vco/api/workflows/389a01cd-ac99-4180-93d1-577ec97bad47/executions/6a50ed53-42fb-4bce-bb57-6dd3ffde73af/interaction/",
        "rel": "add"
      }
    ]
  },
  "input-parameters": [
    {
      "type": "number",
      "name": "var_0"
    }
  ],
  "group-assignees": [],
  "user-assignees": [
    "info@clouddepth.com"
  ],
  "name": "User Interaction Example : User interaction",
  "state": "waiting",
  "id": "8a7480f894ac90710194cba0b1c10394",
  "assignees": [
    "info@clouddepth.com"
  ]
}
```

To create the POST request, the request's body will always begin with the `parameters` key, containing an array of objects constructed from the `input-parameters` key in the previous response.

```bash
curl -X 'POST' \
  'https://vro/vco/api/workflows/389a01cd-ac99-4180-93d1-577ec97bad47/executions/6a50ed53-42fb-4bce-bb57-6dd3ffde73af/interaction' \
  -H 'accept: */*' \
  -H 'Authorization: Basic ABCD...' \
  -H 'Content-Type: application/json' \
  -d '{"parameters":[{"name":"var_0","type":"number","value":{"number":{"value":777}}}]}'
```

As a result, we'll see exactly the same output.

```text
INFO Current value of 'var_0': 123
INFO Updated value of 'var_0': 777
```

## vRA

When the same workflow with User Interaction is triggered via Service Broker, the behavior is the same. The user, who is a member of a security group or a single user, will receive an email automatically, based on their email attribute coming from AD or LDAP, with a link to the User Interaction. The user can provide the input and continue the workflow.

Let's see how it looks in the Service Broker. We'll trigger the workflow in Service Broker. Immediately, we can see it is waiting for a user to respond.

![image](image-12.png){: .shadow }{: .normal }

The user will receive an email with a link to the User Interaction.

![image](image-9.png){: .shadow }{: .normal }

Clicking on a Service Broker link will forward us directly to the User Interaction request.

![image](image-13.png){: .shadow }{: .normal }

Inside, we can see exactly the same input form as in the vRO web client.

![image](image-11.png){: .shadow }{: .normal }

Provide a value and Answer or Reject the request.

### REST API

vRA's endpoints are different the vRO's, but the logic is the same. We can use the REST API to obtain information about user interactions and provide input. The first step is to retrieve the interactions related to the workflow. We can accomplish this by sending a GET request to the following URL:

```bash
curl -X 'GET' \
  'https://vra/workitem/api/workitems/deployment/45c01740-a47b-4c27-b962-449cddc266eb/workitems?state=ACTIVE' \
  -H 'accept: application/json' \
  -H 'Authorization: Bearer ABCD...'
  ```

Here, we're querying the workitem endpoint and providing the deployment ID `45c01740-a47b-4c27-b962-449cddc266eb` and filtering only the `ACTIVE` workitems.

> There are multiple ways to get deployment ID on the fly if full automation of this process is required.
{: .prompt-info}

```json
{
  "content": [
    {
      "workItem": {
        "id": "dd50736c-ff7e-4c7d-8e0e-29f518608569",
        "name": "User Interaction Example : User interaction",
      ...
    }
  ]
}
```

In that case, we have only one interaction represented by `workitem.id`. We can get the information about the interaction by sending a GET request to the interaction URL using the `workitem.id`.

```bash
curl -X 'GET' \
  'https://vra/workitem/api/workitems/dd50736c-ff7e-4c7d-8e0e-29f518608569?expand=true' \
  -H 'accept: application/json' \
  -H 'Authorization: Bearer ABCD...'
  ```

As a result, we will get a big JSON response with many details. But we're interested only in an `inputs` key. Here we can fetch the name of the variable we want to update, its current value and its type. In addition, we can check the status of that interaction before doing a change. Currently it is in a `WAITING` state.

```json
{
  "id": "dd50736c-ff7e-4c7d-8e0e-29f518608569",
  "name": "User Interaction Example : User interaction",
  "description": "User Interaction Example : User interaction",
  "state": "ACTIVE",
  "workItemDetails": {
    "formDetails": {
      "executionItemId": "a67379b0-92f9-403d-85c9-64b2c924d87d",
      "form": {
        ...
      "inputs": {
        "entries": [
          {
            "key": "var_0",
            "value": {
              "value": 123,
              "type": {
                "dataType": "decimal",
                "isMultiple": false
              }
            }
          }
          ...
        }
      ],
      "status": "WAITING"
    }
  }
}
```

Now, we're ready to update the value of variable `var_0` to 777 by sending a POST request to the interaction URL and providing the input parameters.

```bash
curl -X 'POST' \
  'https://vra/workitem/api/workitems/dd50736c-ff7e-4c7d-8e0e-29f518608569/answer' \
  -H 'accept: application/json' \
  -H 'Authorization: Bearer ABCD...' \
  -H 'Content-Type: application/json' \
  -d '{
  "formEntries": {
    "entries": [
      {
        "key": "var_0",
        "value": {
          "type": {
            "dataType": "decimal"
          },
          "value": 777
        }
      }
    ]
  }
}'
```

The response will include the state of the interaction, which is `ANSWERED` (or `REJECTED`)

```json
{
    "id": "93b5ad55-fea3-4c5d-be76-d798c1da1804",
    "name": "User Interaction Example : User interaction",
    "description": "User Interaction Example : User interaction",
    ...
    "state": "ANSWERED"
    ...
}
```

> Bonus question - what do you think will happen, if instead of providing a response value 777, which is expected to be a `decimal`, the value will be a string for example? If your answer was - it will fail, you're right. The response will be:
> ```json
{
    "message": "Error occurred while invoking vRO to perform action on the work item : 400 Bad Request: \"{\"message\":\"Input with name 'var_0' and value: TEST does not contain a parsable number.\",\"messageId\":0,\"stackTrace\":[\"com.vmware.automation.vro.gateway.blueprint.provider.util.ValueUtils.getValueForSingularType(ValueUtils.java:123)\"],\"statusCode\":0,\"errorCode\":0}\"",
    "statusCode": 500,
    "errorCode": 71014
}
```
{: .prompt-tip}

> If we need to reject the user interaction, we can use the `reject` endpoint: `https://vra/workitem/api/workitems/{id}/reject`.
{: .prompt-tip}

> It is important to remember that the user, which execute REST API calls, must have the proper permissions to do that. For example, it should be a member of the `security.group` in each workflows with User Interaction.
{: .prompt-info}

### Conclusion

In this post, we discussed the User Interaction feature in vRO. We explored how to use it in the web client and REST API, and how to send an email to notify users about pending interactions. Additionally, we covered how to handle user interactions in vRA, including the importance of permissions. I hope this post has been helpful and that you can now effectively use User Interaction in your workflows. Stay tuned for the next post, where we will discuss best practices for implementing User Interaction using JavaScript/TypeScript.
