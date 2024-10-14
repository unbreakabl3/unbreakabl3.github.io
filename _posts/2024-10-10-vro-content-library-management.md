---
layout: post
title: vCenter Content Library management with vRO
date: "2024-10-10"
media_subpath: /assets/img/vro-content-library-management/
categories: [VMware, Build Tools, Aria Orchestrator, How To]
tags: [vmware, building_tools, content_library_management, tricks]
---

In the world of virtual infrastructure management, efficiency and automation are key to maintaining smooth operations.
VMware's vCenter Content Library is a powerful tool for managing templates, ISO files, and other VM-related content across multiple vCenter instances.
However, as an environment grows, managing and automating these resources manually can become increasingly challenging.
In this article, we'll explore the step-by-step process of how to manage VMware vCenter Content Library using vRealize Orchestrator, discussing key benefits, potential use cases, and practical workflows to help you leverage these tools for seamless automation in your virtual infrastructure.

## General goals

* Support multiple vCenter instances and VAPI endpoints.
* Support content libraries that offer locally or subscribed content.
* Support dynamic content fetching.
* Write all the code in JavaScript.
* Automate everything as much as possible

A use case:

We want to be able to fetch and show all the items of the selected content library. To do so, we need:

1. Configure VAPI Endpoint/s
2. Get VAPI Endpoints
3. Get all configured content libraries for the selected VAPI Endpoint
4. Show all the items of the selected library
![Image](image 4.png){: .shadow }{: .normal }
_Logic diagram_

## The solution

### Configure VAPI Endpoint

The vCenter VAPI (vSphere Automation API) Endpoint refers to the REST API interface provided by VMware vCenter Server. It allows programmatic access to various vCenter functionalities, such as managing virtual machines (VMs), datastores, networks, and other vSphere resources.

The necessity of this feature stems from the fact that not all vCenter functionalities are fully implemented and accessible through the JavaScript API. The missing APIs can be effectively utilized by VAPI.

To add a new VAPI endpoint, there is a built-in workflow for that called “Add vAPI endpoint”.
![Image](image.png){: .shadow }{: .normal }
Provide vCenter FQDN in URL format and credentials.
![Image](image 2.png){: .shadow }{: .normal }
If everything is done properly, we should see a new item in the Inventory under the vAPI plugin.
![Image](Screenshot 2024-10-08 at 14.50.58.png){: .shadow }{: .normal }
And new APIs will be available for us in the API explorer.
![Image](image 3.png){: .shadow }{: .normal }

## Get VAPI Endpoints

It's possible to configure more than one vCenter/VAPI endpoint. This means the user should be able to select the appropriate one. Naturally, we don't want to hardcode anything but rather fetch all the VAPI endpoints dynamically. Let's accomplish that.

Let's create a basic function named `getAllVapiEndpoints.js`.
The function will attempt to locate all instances of `Server.findAllByType` all the objects of type `VAPI Endpoint`. If we find their names, let's return them. We want to show only their names in the custom form dropdown menu, but not the whole object, since objects can't be shown there.

```javascript
 */
/**
 * @returns {Array/string} - Array of all VAPI endpoints
 */
(function () {
  let vapiEndpoints = [];

  try {
    vapiEndpoints = Server.findAllForType("VAPI:VAPIEndpoint");
  } catch (e) {
    throw new Error(`Failed to retrieve vAPI Endpoints: ${e.message || e}`);
  }

  return vapiEndpoints.length ? vapiEndpoints.map((endpoint) => endpoint.name) : [];
});

```

### Get content libraries

Once the endpoint is selected, we want to retrieve all the existing content libraries associated with that endpoint. To achieve this, let's define another function called `getAllIContentLibraries.js`.
Here, we'll employ an intriguing enhancement: instead of writing all the code within the action element as a single, extensive function, we'll divide our code into multiple, concise functions.
This approach will facilitate code cleanliness, enhanced readability, and testability.

So, what we have here:

1. Validated provided inputs
2. Find an API endpoint object based on the provided name
3. Find content library objects
4. Get content library names from the objects.

We must implement all of these features to display the dropdown list of all libraries in the custom form. This will significantly enhance the user experience, as it will be much easier for users to locate the required content.

```javascript
/*-
 * #%L
 * content_library_management
 * %%
 * Copyright (C) 2024 TODO: Enter Organization name
 * %%
 * TODO: Define header text
 * #L%
 */
/**
 * @param {string} vapiEndpoint - vCenter VAPI Endpoint Name
 * @returns {Array/string} - Arrays of all content libraries
 */
(function (vapiEndpoint) {
  // Main function to fetch OVF library items
  if (!validateParameters(vapiEndpoint)) {
    return [];
  }
  var vapiEndpointObject = findEndpoint(vapiEndpoint);
  var client = vapiEndpointObject.client();
  try {
    var contentLibraries = findContentLibraryObjects(client);
    if (!contentLibraries.length) {
      System.log("No content libraries found.");
      return [];
    }
    var ovfLibraryItems = findContentLibraryNames(client, contentLibraries);
    return ovfLibraryItems.sort();
  } catch (error) {
    System.error("Error occurred: " + error.message);
    throw error;
  } finally {
    client.close();
  }
  // Function to validate input parameters
  function validateParameters(vapiEndpoint) {
    return vapiEndpoint;
  }
  // Function to find the vapiEndpoint
  function findEndpoint(vapiEndpoint) {
    var vapiEndpointObject = VAPIManager.findEndpoint(vapiEndpoint);
    if (!vapiEndpointObject) {
      throw new Error("Failed to find vapiEndpoint: " + vapiEndpoint);
    }
    System.log("VAPI Endpoint: " + vapiEndpointObject.endpointUrl);
    return vapiEndpointObject;
  }

  function findContentLibraryObjects(client) {
    var contentLibraryService = new com_vmware_content_library(client);
    return contentLibraryService.list();
  }

  function findContentLibraryNames(client, libraries) {
    var contentLibraryService = new com_vmware_content_library(client);
    return libraries.map(function (libraryId) {
      return contentLibraryService.get(libraryId).name;
    });
  }
});
```

>As previously mentioned, a single action element can contain multiple functions.
{: .prompt-info}
>Outputs of all functions in that article will be shown in the custom form. Because we have a dependency between inputs and outputs, if there is no output from step one, step two will not work. 
By default, vRO will throw an  `undefined` error. To prevent it and make the user's visual experience nicer and without errors, we can check the status of the inputs and return an empty string `””` or an empty array `[]` in case of dropdowns like in our example.
```javascript
if (!validateParameters(vapiEndpoint, contentLibraryType)) {
  return [];
}
```
{: .prompt-tip}

### Get content of the selected library

For that, we will write another function called `getAllItemsFromContentLibrary.js`.

```javascript
/*-
 * #%L
 * content_library_management
 * %%
 * Copyright (C) 2024 TODO: Enter Organization name
 * %%
 * TODO: Define header text
 * #L%
 */
/**
 * @param {string} vapiEndpoint - vCenter VAPI Endpoint Name
 * @param {string} contentLibraryName - Content Library Name
 * @returns {Array/string} - Array of content library objects
 */
(function (vapiEndpoint, contentLibraryName) {
  // Main function to fetch OVF library items
  if (!validateParameters(vapiEndpoint, contentLibraryName)) {
    return [];
  }

  var vapiEndpointObject = findEndpoint(vapiEndpoint);
  var client = vapiEndpointObject.client();

  try {
    var contentLibraries = findContentLibraries(client, contentLibraryName, vapiEndpoint);
    if (!contentLibraries.length) {
      System.log("No content libraries found.");
      return [];
    }

    var ovfLibraryItems = getOvfLibraryItems(client, contentLibraries);
    return ovfLibraryItems.sort();
  } catch (error) {
    System.error("Error occurred: " + error.message);
    throw error;
  } finally {
    client.close();
  }

  // Function to validate input parameters
  function validateParameters(vapiEndpoint, contentLibraryName) {
    return vapiEndpoint && contentLibraryName;
  }

  // Function to find the vapiEndpoint
  function findEndpoint(vapiEndpoint) {
    var vapiEndpointObject = VAPIManager.findEndpoint(vapiEndpoint);
    if (!vapiEndpointObject) {
      throw new Error("Failed to find  VAPI Endpoint: " + vapiEndpoint);
    }
    System.log("VAPI Endpoint: " + vapiEndpointObject.endpointUrl);
    return vapiEndpointObject;
  }

  // Function to find content libraries
  function findContentLibraries(client, contentLibraryName, vapiEndpoint) {
    var contentLibrarySpec = new com_vmware_content_library_find__spec(client);
    contentLibrarySpec.name = contentLibraryName;
    contentLibrarySpec.type = String(setLibraryType(vapiEndpoint, contentLibraryName));
    var contentLibraryService = new com_vmware_content_library(client);
    var libraries = contentLibraryService.find(contentLibrarySpec);
    return libraries;
  }

  function setLibraryType(vapiEndpoint, contentLibraryName) {
    return System.getModule("com.clouddepth.content_library_management.actions").getContentLibraryType(vapiEndpoint, contentLibraryName);
  }

  // Function to get OVF library items from found libraries
  function getOvfLibraryItems(client, libraries) {
    var libraryItemService = new com_vmware_content_library_item(client);
    var ovfLibraryItems = [];

    libraries.forEach(function (library) {
      var items = libraryItemService.list(library);

      if (items && items.length > 0) {
        items.forEach(function (item) {
          var ovfItem = libraryItemService.get(item.toLowerCase());
          System.log("Found content library item: " + ovfItem.name);
          ovfLibraryItems.push(ovfItem.name);
        });
      } else {
        System.log("No items found in content library: " + library.name);
      }
    });

    return ovfLibraryItems;
  }
});
```

The interaction with the content library here is similar to the one in the previous function, but let's focus on `findContentLibraries` function. This function is preparing a spec to query the library properly. To do so, we need to specify two important inputs:

1. Library type: LOCAL or SUBSCRIBED
2. Library name

As usual, we want to make this process fully automated. Let's introduce a new function called `getContentLibraryType`.
The main idea of it is to get all library names by running `contentLibraryService.get(libraryId)` and compare with the one provided by the user. Once found, we can query it and find its type: LOCAL or SUBSCRIBED.
That's all. Now, we can pass it to the `getAllItemsFromContentLibrary.js` function and use its value to set `contentLibrarySpec.type`.

```javascript
...
  function findContentLibraryType(client, libraries, contentLibraryName) {
    var contentLibraryService = new com_vmware_content_library(client);

    // Filter libraries by name
    var matchingLibraries = libraries.filter(function (libraryId) {
      return contentLibraryService.get(libraryId).name === contentLibraryName;
    });

    // Map to extract the 'type' of the matching libraries
    return matchingLibraries.map(function (libraryId) {
      return contentLibraryService.get(libraryId).type;
    });
  }
... 
```

And now, we can use it dynamically in our main code by providing a `contentLibraryType` as a user input.

```javascript
contentLibrarySpec.type = setLibraryType(contentLibraryType);
```

>The type of `contentLibraryService.get(libraryId).type` is a string. `contentLibrarySpec.type` expecting to get a string. But for some reason, it's not recognized as a string by runtime. To fix it, I force it to be a string by using `String()` method: `contentLibrarySpec.type = String(setLibraryType(vapiEndpoint, contentLibraryName));`.
{: .prompt-tip}

## Test

Awesome! Let's kick off our workflow and see the results.

The best part is that all our inputs are automatically updated. So, we can pick the VAPI Endpoint, browse through all the available libraries there, and make our choices.
![Image](Screenshot 2024-10-10 at 12.38.45.png){: .shadow }{: .normal }
Check out all the items in that library. Once you select one, the workflows can take care of it whenever they need to.

![Image](Screenshot 2024-10-10 at 12.40.16.png){: .shadow }{: .normal }

## Summary

As we can observe, the content library is a highly versatile and powerful tool. In the future, we will attempt to enhance its capabilities by adopting a code-based approach.

## Source Code

The source code can be found [here](https://github.com/unbreakabl3/vmware_aria_orchestrator_examples/tree/main/content_library_management)

The vRO packages are also available [here](https://github.com/unbreakabl3/vmware_aria_orchestrator_examples/raw/refs/heads/main/content_library_management/com.clouddepth.content_library_management-1.0.40.package)  and the external ECMASCRIPT package [here](https://github.com/unbreakabl3/vmware_aria_orchestrator_examples/raw/refs/heads/main/content_library_management/com.vmware.pscoe.library.ecmascript-2.41.0.package).

> Both packages should be imported
{: .prompt-info}
