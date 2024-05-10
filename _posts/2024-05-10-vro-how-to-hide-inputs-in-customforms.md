---
layout: post
title: Advanced techniques for masking inputs in vRO custom forms.
date: "2024-05-10"
img_path: /assets/img/vro-how-to-hide-inputs-in-customforms/
categories: [VMware, Build Tools, Aria Orchestrator, How To, vRO]
tags: [vmware, vro, tricks, how-to]
---

The vRO custom form is an amazing tool that offers a wide range of built-in functionalities to create dynamic and flexible forms. What's even more remarkable is that we can add certain functionalities, which may not be available out-of-the-box, using other tools that are always readily accessible.

## Problem

We have a workflow with two inputs that need to be visible when the workflow starts. However, if any value is entered in input 1, field number 2 must be hidden. If we try to accomplish that using the OOB Appearance > Visibility conditions, we'll fail because they only meet some of our requirements.

### Solution

The appearance conditions are booleans - show the filed = yes, if... This means we can use External Source to customize the logic, which doesn't exist in the appearance conditions.

### Action Element

Let's create a straightforward action element with one input. The output should be of type _boolean_.

```javascript
if (!input) {
    return true
}
```

The final result should be like that.
![Image](c4a243d3-82e5-474c-ba2d-e098f629d6f2.png){: .shadow }{: .normal }

### Workflow Custom Form

As described in the problem section, both inputs should be visible when the workflow starts, but _input1_ is not empty, and _input2_ should disappear. To do so, we will attach the _checkInput_ action to _input2_ like that.
![Image](6b24fd58-0e0e-471c-a656-e329ca3a9e3c.png){: .shadow }{: .normal }

That's all. Let's test!

### Tests

When the workflow is started, we can see both inputs.
![Image](c1d48d58-75e0-4a8f-9276-cb7831dc0b21.png){: .shadow }{: .normal }

If we put some value in the input1, input2 disappears.
![Image](36effbe0-5be5-4073-a133-2767557e11b8.png){: .shadow }{: .normal }

## Summary
