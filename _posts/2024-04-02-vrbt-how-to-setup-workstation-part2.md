---
layout: post
title: How to setup the workstation for Build Tools for VMware Aria - part 2
date: '2024-04-02 '
img_path: /assets/img/vrbt-how-to-setup-workstation-part2/
categories: [VMware, Build Tools, Aria Orchestrator, How To, vRO]
tags: [vmware, building_tools]
---

In the [previous](https://www.clouddepth.com/posts/vrbt-how-to-setup-workstation/) step, we reached a point where we had an empty project with all the default, pre-defined packages and settings. This should look like the following:

![project_folder_structure](815ea80c-dd9a-4bf3-a13c-919bd3f19c56.png){: .shadow }{: .normal }{: width="400" height="300" }

> One important thing to remember is that certain elements, such as types, unit tests, etc., cannot be transferred to vRO. However, Build Tools can convert cool features like Classes and Promises to pure JS, making them compatible with vRO, which is using [Rhino Engine](https://cloudblogger.co.in/2022/12/02/inside-vros-javascript-engine-rhino-1-7r4-cb10102/) to execute JS code.
{: .prompt-info}

## Folders structure review

By default, when a new project is created, a few default examples are provided (Actions, Config Elements, Workflows, etc.), covering most of the use cases to start with, at least initially.

![project_folder_structure](37f732aa-a025-402b-945f-8b0482f67534.png){: .shadow }{: .normal }{: width="300" height="200" }

Let's quickly review what we have in our project:

* _actions_ - can hold _.js_ and _.ts_ files, which will be converted to the Action Elements
* _element_:
  * _config_ - can hold _.ts_ or _.yaml_ files, which will be converted to the Configuration Elements
  * _resource_ - can hold _.txt, .json, .yaml, .xml_ files, which will be converted to the Resources
* _policies_ - _.pl.ts_ files, which will be converted to the Policy
* _types_ - _.ts_ files, which will hold the types/interfaces. Not mandatory, but recommended, to store types/interfaces in that folder. [Here](https://github.com/unbreakabl3/vmware_aria_orchestrator_examples/blob/main/src/types/types.d.ts) is an example.
* _workflows_ - holds _.wf.ts_ and _wf.form.json_, which will be converted to the Workflow
* _pom.xml -_ includes all the necessary dependencies and defaults to build the package. This file can be adjusted accordingly to your needs.

Creating a dedicated _test_ folder in _/src_ for test files may be a good idea. Unit tests are executed automatically each time we run `mvn clean install vrealize:push -P vro01` if at least one test file is located <u>anywhere</u> in the project.

> All the pre-defined files can be removed from your project. They are here just for an example.
{: .prompt-info}

## First push

Firstly, let's confirm our ability to communicate with vRO by creating a vRO package and pushing it to vRO..

```shell
mvn clean install vrealize:push -P vro01
```

Maven will use the NodeJS we installed last time to convert the TS code into the JS code and build the vRO package.
Let's see what is happening:

* _clean_ - will delete _node_modules_ and _target_ folders and re-generate them based on the latest changes.
* _install_:
  * _mvn_ will look in _pom.xml_, which has all the instructions it needs to build, test, package, and deploy your project in a consistent and automated manner, and will store them in the _node_modules_ folder
  * _mvn_ will generate a _target_ folder and export into all transpiled TS files
  * _mvn_ will generate a package based on the _groupId_, _artifactId_ and version stored in _pom.xml_. In our case, the package's name will be _test.test-1.0.0-SNAPSHOT.package_. This is an encrypted container, which will be pushed to vRO and extracted there.
  
 ![project_folder_structure](26a5fc2f-93f9-4780-8c5c-e63eebc77a24.png){: .shadow }{: .normal }{: width="400" height="300" }
> If, for some reason, _target_ or _node_modules_ should be kept, remove _clean_ from the command: `mvn install vrealize:push -P vro01`
{: .prompt-tip}

* _`vrealize:push -P vro01`_ - will push created package into vRO instance called _vro01_ (the same we configured in _settings.xml_ before)
Maven will add a few supporting packages to the main package, like _com.vmware.pscoe.library.ecmascript-2.38.0_, which are includes all the magic code, that will help vRO "understand" the [ECMAScript 6](https://262.ecma-international.org/6.0/?_gl=1*pyfepb*_ga*MTgyNTg2MjM4Ni4xNzExOTk5Mjc0*_ga_TDCK4DWEPP*MTcxMTk5OTI3My4xLjAuMTcxMTk5OTI3My4wLjAuMA..) the features. [There are](https://www.w3schools.com/js/js_es6.asp) some of the features.
At the end, we should see our package in vRO packages menu:

 ![vro_package_structure](1d0b1e6f-ff7d-42b5-a7de-1b6f505e4eac.png){: .shadow }{: .normal }{: width="400" height="300" }

If we will open the test package, we'll see all the files transferred to vRO.

 ![vro_package_structure](5b635c69-ae52-4234-8ed3-f7c517f8786d.png){: .shadow }{: .normal }{: width="500" height="400" }

> Compare those files with the files you have in the VSCode project and see if there are some differences. Usually, there will be fewer files in the package because files like _tests_ or _types_ will not be added.
{: .prompt-info}

## Tricks

### Enable null checking

One of the first things to do is adjust Typescript defaults by adding the following to the _tsconfig.json_ file. This will make our code safer during development.

```json
{
  "strictNullChecks": true,
  "strict": true,
}
```

More information can be found [here](https://www.typescriptlang.org/tsconfig#strictNullChecks) and [here](https://www.typescriptlang.org/tsconfig#strict).
It's worth noting that regular _npm_ packages cannot be downloaded and used, so it's impossible to use [Fetch](https://www.npmjs.com/package/node-fetch) for REST API calls, for example. However, there are a few workarounds that may be helpful. We'll explore these in more detail later on.

### Performance

We can speed up our build a bit if we use more CPU cores. To do that, we need to add to Maven the following switches:

```shell
mvn -T 1C
```

## Summary

If you did work with vRO for some time, the default state of mind was to use vRO's canvas, drag the elements into, connect between them, etc.
This is what my workflows were look like. Until now.

 ![workflow example](5972bd81-c14b-49ff-a96d-b4e5f504d940.png){: .shadow }{: .normal }{: width="400" height="300" }

[Build Tools for VMware Aria](https://github.com/vmware/build-tools-for-vmware-aria) provides a different perspective, allowing us to approach tasks uniquely. We no longer need to create numerous scriptable tasks, each containing a small or large code snippet, linking one task's output to the next's input and so on. We no longer use canvas elements like Timer, Exception, Decision, or Counter elements. We can have only one scriptable task containing all our code, replacing all those elements. Additionally, all supporting functions can be (should be) written as Action Elements and imported within our primary scriptable task (see the example [here](https://github.com/unbreakabl3/vmware_aria_orchestrator_examples/blob/main/src/actions/waitForDNSResolve.ts)). This approach significantly improves code quality, making it much cleaner and more accessible to test. We can test our main code and all its Action Elements within the same project, ensuring it is far more stable.

That's how my Workflows are looks now :)

 ![workflow example](9911a03d-d5ae-4e12-80a9-4c8d6b4d0b3b.png){: .shadow }{: .normal }

The statement above represents my personal viewpoint. You may consider trying it out. It may require altering your mindset and coding habits, which can take some time, but the result is fantastic. If things don't work out, it's alright to keep creating multiple elements, linking them together, and so on while still utilizing Build Tools to make them where they are applicable.

In the upcoming section, we will delve deeper into the capabilities of Build Tools for VMware Aria. We will cover fundamental and advanced features, providing you with a comprehensive understanding of utilizing this tool to its fullest potential.
