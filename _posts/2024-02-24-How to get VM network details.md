---
title: How to get VMware vCenter VM network details
date: 2024-02-24
categories: [VMware, Aria Orchestrator]
tags: [vmware, building_tools, typescript, aria_orchestrator, unit_test, jasmine]
---

Let's take a quick look at a simple one that returns the IP details of the VM running in VMware vCenter, using VSCode, Build Tools for VMware Aria, and Typescript project.

## Create a class and a method

We'll create a sample class using one method to achieve this goal.
First, our (method) is a part of the Network class, where we can create more methods in the same topic if needed.
Secondly, VSCode shows us the type of the returned values, which helps us understand how to extract necessary values. For example, variable `vcGuestNicInfo` has type `vcGuestNicInfo`

![img-description](/assets/img/vmware-how-to-get-VM-network-details/Pasted%20image%2020240224140910.png){: .shadow }

Code snippet:

```typescript
export class SampleClass {
    public getNetworkDetails (vm: VcVirtualMachine) {
        const vcGuestNicInfo = vm.guest.net;
        vcGuestNicInfo.forEach(function (element) {
            System.log(`MAC:  ${element.macAddress}`);
            System.log(`IP Address:  ${element.ipConfig.ipAddress[0].ipAddress}`)
            System.log(`Prefix:  ${element.ipConfig.ipAddress[0].prefixLength}`)
        });
        const vcGuestStackInfo = vm.guest.ipStack
        vcGuestStackInfo.forEach(function (element) {
            System.log(`Gateway: ${element.ipRouteConfig.ipRoute[0].gateway.ipAddress}`)
        });
    }
}
```

## Unit Test

Now, we want to test our function. Build Tools has a built-in support for Jasmine. Because we don't want to execute real calls to the vCenter, we'll mock the returned object with the values and test only the method.

```typescript
import { SampleClass } from "./sample"

describe('getNetworkDetails', () => {
    let vm;
    let SystemSpy;
    const func = new SampleClass()

    beforeEach(() => {
        // Create a mock VcVirtualMachine object
        vm = {
            guest: {
                net: [
                    {
                        macAddress: '00:11:22:33:44:55',
                        ipConfig: {
                            ipAddress: [{ ipAddress: '192.168.1.10', prefixLength: 24 }]
                        }
                    }
                ],
                ipStack: [
                    {
                        ipRouteConfig: {
                            ipRoute: [{ gateway: { ipAddress: '192.168.1.1' } }]
                        }
                    }
                ]
            }
        };
        SystemSpy = spyOn(System, 'log');
    });

    it('should log MAC address, IP address, prefix and gateway', () => {
        func.getNetworkDetails(vm);
        expect(SystemSpy.calls.allArgs()).toEqual([
            [`MAC:  00:11:22:33:44:55`],
            [`IP Address:  192.168.1.10`],
            [`Prefix:  24`],
            [`Gateway: 192.168.1.1`]
        ]);
    });
});
```

The result was successful.

![img-description](/assets/img/vmware-how-to-get-VM-network-details/Pasted%20image%2020240224141306.png){: .shadow }

## Transpiled result

After the TypeScript transpilation is executed, we're getting the following JavaScript code, which can be used as an Action Element or somewhere inside the Scriptable Task in the Workflow:

```javascript
/**
 * @return {Any}
 */
(function () {
    var exports = {};
    var SampleClass = /** @class */ (function () {
        function SampleClass () {
        }
        SampleClass.prototype.getNetworkDetails = function (vm) {
            var vcGuestNicInfo = vm.guest.net;
            vcGuestNicInfo.forEach(function (element) {
                System.log("MAC:  " + element.macAddress);
                System.log("IP Address:  " + element.ipConfig.ipAddress[0].ipAddress);
                System.log("Prefix:  " + element.ipConfig.ipAddress[0].prefixLength);
            });
            var vcGuestStackInfo = vm.guest.ipStack;
            vcGuestStackInfo.forEach(function (element) {
                System.log("Gateway: " + element.ipRouteConfig.ipRoute[0].gateway.ipAddress);
            });
        };
        return SampleClass;
    }());
    exports.SampleClass = SampleClass;
    return exports;
});
```

Today, we saw how to get network details of the VM and test our code.
