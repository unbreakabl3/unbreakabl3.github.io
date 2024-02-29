---
title: How to resolve FQDN to an IP address in Aria Orchestrator
date: 2024-02-29
categories: [VMware, Aria Orchestrator]
tags: [vmware, building_tools, typescript, aria_orchestrator, unit_test, jasmine]
---

Let’s see how we can use the built-in `System.resolveHostName()` method in vRO to resolve the FQDN to the IP address.

## Implementation

We’re going to create a new class with a few methods:

1. waitForDNSResolve
2. isValidIPv4
3. throwError

Let’s say we created a new A record in the DNS server and must wait until the address is resolved to continue the workflow. It may take some time until the IP is resolved. Therefore, we’ll need to wait and try a few times. If, after a few retries, the IP is still not resolvable, we’ll throw an error.

### waitForDNSResolve()

Here, a function `waitForDNSResolve()` accepts a host FQDN as a string. The function will try to resolve the FQDN 20 times and wait for 60 seconds between retries.

```typescript
    public waitForDNSResolve ( hostFqdn: string ): void {
        const maxAttempts = 20;
        const sleepTime = 60000; // Sleep for a minute

        for ( let i = 0; i < maxAttempts; i++ ) {
            try {
                const ip = System.resolveHostName( hostFqdn );
                if ( ip && this.isValidIPv4( ip ) ) {
                    return; // DNS resolved successfully
                }
            } catch ( error ) {
                System.error( `Error resolving ${ hostFqdn }: ${ error.message }` );
            }
            System.log( `Not yet resolvable. Sleeping for ${ sleepTime / 1000 } seconds` );
            System.sleep( sleepTime )
        }
        System.warn( `DNS resolution failed after ${ maxAttempts } attempts.` );
    }
```

### isValidIPv4()

Once the FQDN is resolved, we want to make sure that the return IP address is a valid IP. For that, we have a function `isValidIPv4()`, which accepts the IP as a string, tests it against a regular expression, and returns a boolean if the address is valid.

```typescript
    public isValidIPv4 ( ip: string ): boolean {
        const ipv4Regex = /^((25[0-5]|(2[0-4]|1\d|[1-9]|)\d)\.?\b){4}$/;
        return ipv4Regex.test( ip )
    }
```

### throwError()

If the result of `isValidIPv4()` is true, the main function `waitForDNSResolve()` will return. But if after 20 retries, the FQDN is still not resolved, we will throw an error with a simple function called `throwError()` .

```typescript
    public throwError ( errorMessage: string ): never {
        throw new Error( errorMessage )
    }
```

### The final code

```typescript
export class SampleClass {
    public throwError ( errorMessage: string ): never {
        throw new Error( errorMessage )
    }

    public isValidIPv4 ( ip: string ): boolean {
        const ipv4Regex = /^((25[0-5]|(2[0-4]|1\d|[1-9]|)\d)\.?\b){4}$/;
        return ipv4Regex.test( ip )
    }

    public waitForDNSResolve ( hostFqdn: string ) {
        const maxAttempts = 20;
        const sleepTime = 60000; // Sleep for a minute

        for ( let i = 0; i < maxAttempts; i++ ) {
            try {
                const ip = System.resolveHostName( hostFqdn );
                if ( ip && this.isValidIPv4( ip ) ) {
                    return; // DNS resolved successfully
                }
            } catch ( error ) {
                System.error( `Error resolving ${ hostFqdn }: ${ error.message }` );
            }
            System.log( `Not yet resolvable. Sleeping for ${ sleepTime / 1000 } seconds` );
            System.sleep( sleepTime )
        }
        System.warn( `DNS resolution failed after ${ maxAttempts } attempts.` );
    }
}
```

## Unit Test

Creating a unit test for a code is always a good idea. Let’s develop tests for our functions with Jasmine, which is built into the [vRBT](https://github.com/vmware/build-tools-for-vmware-aria). The code is pretty self-explanatory.

### Testing isValidIPv4()

```typescript
describe( 'isValidIPv4', () => {
    let fakeIp;

    beforeEach( () => {
    } );

    it( 'should return true for a valid IP address', () => {
        fakeIp = '192.168.1.10';
        const result = func.isValidIPv4( fakeIp );

        expect( result ).toBe( true );
    } );

    it( 'should return false for an invalid IP address', () => {
        fakeIp = '256.0.0.1'; // Invalid IP
        const result = func.isValidIPv4( fakeIp );

        expect( result ).toBe( false );
    } );
} );
```

### Testing waitForDNSResolve()

```typescript
describe( 'waitForDNSResolve', () => {
    let fakeHostFqdn;
    let fakeIp;

    beforeEach( () => {
        fakeIp = '192.168.1.10';
        fakeHostFqdn = 'example.com';

        spyOn( System, 'error' );
        spyOn( System, 'log' );
        spyOn( System, 'warn' );
    } );

    it( 'should resolve DNS successfully', () => {
        spyOn( System, 'resolveHostName' ).and.returnValue( fakeIp );
        func.waitForDNSResolve( fakeHostFqdn );

        expect( System.resolveHostName ).toHaveBeenCalledWith( fakeHostFqdn );
    } );

    it( 'should log an error if DNS resolution fails', () => {
        const errorMessage = 'DNS resolution failed';
        spyOn( System, 'resolveHostName' ).and.throwError( errorMessage );
        func.waitForDNSResolve( fakeHostFqdn );

        expect( System.resolveHostName ).toHaveBeenCalledWith( fakeHostFqdn );
        expect( System.error ).toHaveBeenCalledWith( `Error resolving ${ fakeHostFqdn }: ${ errorMessage }` );
    } );

    it( 'should log a warning after max attempts', () => {
        spyOn( System, 'sleep' );
        func.waitForDNSResolve( fakeHostFqdn );

        expect( System.sleep ).toHaveBeenCalledTimes( 20 );
        expect( System.warn ).toHaveBeenCalledWith( `DNS resolution failed after 20 attempts.` );
    } );
} );
```

### The final code

```typescript
import { SampleClass } from "./sample"
const func = new SampleClass()

describe( 'isValidIPv4', () => {
    let fakeIp;

    beforeEach( () => {
    } );

    it( 'should return true for a valid IP address', () => {
        fakeIp = '192.168.1.10';
        const result = func.isValidIPv4( fakeIp );

        expect( result ).toBe( true );
    } );

    it( 'should return false for an invalid IP address', () => {
        fakeIp = '256.0.0.1'; // Invalid IP
        const result = func.isValidIPv4( fakeIp );

        expect( result ).toBe( false );
    } );
} );


describe( 'waitForDNSResolve', () => {
    let fakeHostFqdn;
    let fakeIp;

    beforeEach( () => {
        fakeIp = '192.168.1.10';
        fakeHostFqdn = 'example.com';

        spyOn( System, 'error' );
        spyOn( System, 'log' );
        spyOn( System, 'warn' );
    } );

    it( 'should resolve DNS successfully', () => {
        spyOn( System, 'resolveHostName' ).and.returnValue( fakeIp );
        func.waitForDNSResolve( fakeHostFqdn );

        expect( System.resolveHostName ).toHaveBeenCalledWith( fakeHostFqdn );
    } );

    it( 'should log an error if DNS resolution fails', () => {
        const errorMessage = 'DNS resolution failed';
        spyOn( System, 'resolveHostName' ).and.throwError( errorMessage );
        func.waitForDNSResolve( fakeHostFqdn );

        expect( System.resolveHostName ).toHaveBeenCalledWith( fakeHostFqdn );
        expect( System.error ).toHaveBeenCalledWith( `Error resolving ${ fakeHostFqdn }: ${ errorMessage }` );
    } );

    it( 'should log a warning after max attempts', () => {
        spyOn( System, 'sleep' );
        func.waitForDNSResolve( fakeHostFqdn );

        expect( System.sleep ).toHaveBeenCalledTimes( 20 );
        expect( System.warn ).toHaveBeenCalledWith( `DNS resolution failed after 20 attempts.` );
    } );
} );
```
