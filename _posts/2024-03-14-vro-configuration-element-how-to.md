---
layout: post
title: How to create a vRO Configuration Element?
date: '2024-03-14'
categories: [VMware, Aria Orchestrator, How To, vRO]
tags: [vmware, building_tools, typescript, aria_orchestrator, unit_test, jasmine, configuration_element]
---
## Configuration Element on steroids

Configuration element is one of the essential and outstanding features of vRO. It is recommended that some constant variables be updated manually from time to time without hardcoding them in the main code, for example.

## Creation of the Configuration Element

The vRBT allows us to create a Config Element using code. The example below shows how to do that.

```typescript
import { Configuration } from "vrotsc-annotations"

@Configuration ( {
    name: "power_settings",
    path: "my_path",
    attributes: {
        powerMin: {
            type: "number",
            value: "1",
            description: "Minimum power",
        },
        powerMax: {
            type: "number",
            value: "2",
            description: "Maximum power",
      },
      domainName: {
        type: "string",
        value: "domain.local",
        description: "Domain name",
    },
    },
} )
export class MyClass {}

```

As a result, we have a Configuration Element in vRO
![img-description](/assets/img/vro-configuration-element-how-to/image.png){: .shadow }

## Implementation

We’re going to create a new class with a few methods:

1. findConfigurationElementByName
2. getConfigElement
3. getConfElementAttributes
4. validateConfigElementAttributes
5. handleConfigError
In addition, we’ll implement a custom error handling class.

### getConfigElement

Configuration Element requires providing arguments to fetch it:

1. configuration element name
2. configuration element path

Both of them should be strings, and we’ll use the values from the previously created element:

1. `configName = power_settings`
2. `configPath = my_path`

First of all, we’re checking if they’re defined.

```typescript
if (!configName || !configPath) {
  return undefined;
}
```

After that, we tried to fetch all elements using `Server.getConfigurationElementCategoryWithPath`, which returns an array of `ConfigurationElement`.

```typescript
const configurationElements: Array<ConfigurationElement> = Server.getConfigurationElementCategoryWithPath (configPath)?.allConfigurationElements;
if (!configurationElements) {
  this.handleConfigError (configName, 'Configuration element category not found');
}
```

If something was found, we tried to find our specific element using `findConfigurationElementByName` method. If it was found, the function returned this element.

```typescript
public getConfigElement({ configName, configPath }: { configName: string; configPath: string }): ConfigurationElement | undefined {
  if (!configName || !configPath) {
    return undefined;
  }
  try {
    const configurationElements: Array<ConfigurationElement> = Server.getConfigurationElementCategoryWithPath (configPath)?.allConfigurationElements;
    if (!configurationElements) {
      this.handleConfigError (configName, 'Configuration element category not found');
    }

    const configurationElement: ConfigurationElement | undefined = this.findConfigurationElementByName (configurationElements, configName);
    if (!configurationElement) {
      this.handleConfigError (configName, 'Configuration element not found');
    }

    return configurationElement;
  } catch (error) {
    this.handleConfigError (configName, error);
  }
}
```

### findConfigurationElementByName

That is a straightforward function that receives an array of configuration elements and tries to find one matching the name.

```typescript
private findConfigurationElementByName ( elements: Array<ConfigurationElement>, name: string ): ConfigurationElement | undefined {
  return elements.find ( ( element ) => element.name === name );
}
```

### getConfElementAttributes

Once we find the element, we want to read its values. In that example, we have three key-value attributes:

1. `powerMin: number`
2. `powerMax: number`
3. `domainName: string`
TypeScript allows us to check/warn that the `getAttributeWithKey` method may return an empty value by using the `?` mark.
`const attributes: AttributeMap = {};`  - is a constant that has the type ￼`AttributeMap.` `AttributeMap` is a type we created to ensure we work only with predefined parameters.

```typescript
type AttributeMap = {
  powerMin?: number;
  powerMax?: number;
  domainName?: string;
};
```

If the attributes are found, the function will validate them by running the `validateConfigElementAttributes` function, which returns them.

```typescript
public getConfElementAttributes ( elementName: ConfigurationElement ): AttributeMap | undefined {
  const attributes: AttributeMap = {};
  try {
    attributes.powerMin = elementName.getAttributeWithKey ( "powerMin" )?.value;
    attributes.powerMax = elementName.getAttributeWithKey ( "powerMax" )?.value;
    attributes.domainName = elementName.getAttributeWithKey ( "domainName" )?.value;
  } catch ( error ) {
    throw new Error ( `Could not getAttributeWithKey() from element '${elementName}'. ${error}` );
  }
  
  if ( this.validateConfigElementAttributes ( attributes ) ) return attributes
}
```

### validateConfigElementAttributes

To ensure we’re getting the proper values from the Configuration Element, we need to validate them. We know that we’re expecting to get three attributes, two of which have type ￼`number`￼ and one ￼`string.`
The function is checking the type of each attribute. If it’s matching, the function returns the attributes.

```typescript
private validateConfigElementAttributes ( attrs: AttributeMap ): AttributeMap {
  const areAttributesValid = ( attrs: AttributeMap ): boolean =>
    attrs.powerMin !== undefined &&
    typeof attrs.powerMin === "number" &&
    attrs.powerMax !== undefined &&
    typeof attrs.powerMax === "number" &&
    attrs.domainName !== undefined &&
    typeof attrs.domainName === "string";

  if ( areAttributesValid ( attrs ) ) {
    return attrs;
  } else {
    throw new Error ( "One or more attributes are missing, null, or not a number (powerMin/powerMax) or string (domainName)" );
  }
}
```

### handleConfigError

This is a straightforward error-handling function that implements ￼the `InvalidConfigElementError` class. This class was created to show the best practices for implementing dedicated error handling per error. It is an elementary class, but we’ll make it more advanced in future examples.
We are using this method in each of the methods above to handle the exception.

```typescript
private handleConfigError ( configName: string, error: unknown ): never {
  throw new InvalidConfigElementError ( `Could not find configuration element ${configName}. ${error}` );
}
```

```typescript
export class InvalidConfigElementError extends Error {
  constructor ( message: string ) {
    super ( message );
    this.name = "InvalidConfigElementError";
  }
}
```

## Unit-Test

I decided not to post the unit tests here because they are mostly self-explanatory. And - they are a lot :)
I think the results of the tests should be enough. The tests themself can be found [here](https://github.com/unbreakabl3/vmware_aria_orchestrator_examples/blob/main/src/test/configElement.test.ts).
![img-description](/assets/img/vro-configuration-element-how-to/image%202.png){: .shadow }

## Source code

The source code can be found [here](https://github.com/unbreakabl3/vmware_aria_orchestrator_examples.git)
