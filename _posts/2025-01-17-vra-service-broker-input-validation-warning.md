---
layout: post
title: VMware Cloud Foundation Automation Service Broker Input Validation Deep Dive
date: "2025-01-17"
media_subpath: /assets/img/vra-service-broker-input-validation-warning/
categories: [VMware, Build Tools, Aria Orchestrator, Aria Automation, VMware Cloud Foundation Orchestrator, VMware Cloud Foundation Automation]
tags: [vmware, tips, validation]
---
With the release of VMware Aria Automation 8.18, a powerful new feature has been introduced - Form Actions. This functionality enables input validation on Service Broker Orchestrator custom forms, adding a new layer of stability and control. While the concept is promising, the implementation comes with its own set of challenges.

In this article, we’ll explore what Form Actions bring to the table, how they can enhance our automation workflows, and the potential pitfalls to watch out for. Let’s dive in!

## What is Form Actions

[Form Actions](https://techdocs.broadcom.com/us/en/vmware-cis/aria/aria-automation/8-18/consumption-on-prem-using-master-map-8-18/service-catalog-setting-up-service-catalog-for-your-organization/service-broker-custom-forms-customize-a-request-form/service-broker-custom-forms-learn-more-about-service-broker-custom-forms.html#:~:text=The%20Form%20Actions%20tab%20in%20the%20custom%20form%20designer%20shows%20a%20list%20of%20all%20actions%20that%20are%20used%20in%20the%20form.) is a new addition in Aria Automation 8.18 that provides a centralized view of all actions used in a custom form. This makes it incredibly convenient to see all external source actions in one place. However, in this discussion, we’ll focus specifically on its role in Input Validation and how it enhances general usability.
![image](image.png){: .shadow }{: .normal }
_vRA custom form actions example_

## Input Validation

Let's create a test workflow and use an **external source action element** in one of the inputs. The action itself will be straightforward, returning a simple string. This will help us understand how external source actions function within the custom form.

![image](image-2.png){: .shadow }{: .normal }
_vRO Action Element_
![image](image-1.png){: .shadow }{: .normal }
_vRA Service Broker test workflow Custom Form using action element_

As soon as we bind the action to the input, a **warning message** appears. This message highlights two important points:

1. The input for this action can be overridden when using the vRA.
2. Verify your action contains input validation.

Before we dive into understanding what this means, let's try saving our custom form. As expected, another warning message appears.

![image](image-3.png){: .shadow }{: .normal }

This message sheds more light on what's happening. The key takeaway from this warning is that an API call can be made to the catalog item, potentially **overriding its input(s)**. This is a crucial point because it means that we cannot fully trust the input received by the action, as it may be altered externally.

### Deep Dive

When a catalog item is submitted through vRA's Service Catalog GUI, we leverage all its built-in validation features, such as RegEx constraints, Min/Max values, or External Validation. In this scenario, input validation works as expected, ensuring data integrity before submission.  

However, there are many cases where the catalog item is **called indirectly**. For example, when using vRA's API or integrating with a third-party tool like ServiceNow plugin, the submission **bypasses** the UI level validations. This is because API calls do not trigger the same input validation mechanisms that are applied when manually providing inputs through the UI. As a result, relying solely on built-in validation may not be sufficient in such cases.

When the form is submitted via an API call, the inputs are passed as a JSON object. Since all values in the JSON structure are essentially strings, they can contain any type of data. Unlike the UI submission, an API call does not trigger dynamic field loading, data extraction, or built-in input validation.  

This means that if we expect an integer, we need to validate it. If we expect a string, we need to validate it. The same applies to booleans, lists, or any other data type. Without proper validation, incorrect or unexpected values could be passed, potentially causing errors or unintended behavior. In this process, vRA simply receives the JSON with all inputs and attempts to execute the workflow directly, without verifying the data beforehand.

Example: in a custom form, we might have a dropdown box populated with a predefined set of values. However, when submitting the form via API, there is no restriction preventing us from providing any value, even if it is not in the predefined list. This makes validation essential to ensure that only allowed values are accepted.  

Similarly, if we have a text field expecting a number within a specific range, the API submission does not enforce this constraint. It allows any value to be passed, even one that falls outside the expected range. The same issue applies to other input types - without proper validation, there is no guarantee that the provided data adheres to the expected format or constraints.

## Conclusion

The main purpose of this warning is to draw our attention to the need for input validation in our code. We should validate inputs before making an API call whenever possible or consume the form directly from the UI to ensure proper constraints are applied.  

In the API calls context, Service Broker and vRO do **not** verify the action element or check for conditions like `if (!input) return "invalid input"`. External validation in the custom form does not affect this process, and there is nothing we can do in the code to make the warning disappear - aside from clicking the **I UNDERSTAND** button, which is purely cosmetic.  

Despite this, the warning is crucial and should not be ignored. Proper input validation is always a best practice. Relying **only** on UI-level validation is risky, as API calls can bypass these checks. In the end, it is always better to enforce validation in the code to ensure data integrity and avoid potential issues.
