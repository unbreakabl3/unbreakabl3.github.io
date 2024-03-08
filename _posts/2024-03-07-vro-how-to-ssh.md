---
date: '2024-03-07 18:33:02 +0100'
title: How to SSH to remote host from Aria Orchestrator
date: 2024-02-29
categories: [VMware, Aria Orchestrator]
tags: [vmware, building_tools, typescript, aria_orchestrator, unit_test, jasmine, ssh]
---

## Implementation

We’re going to create a new class `SSH` with a few methods:

1. executeSshCommand
2. setNewSshSessions

Connecting with SSH to the remote server is always an asynchronous action. Therefore, we need to wait until the execution will be done.

### executeSshCommand()

First, we will define a few mandatory constants, which are self-explanatory. One is the `path` where the private SSH key is located. VRO will use this key to ssh to the remote server. This path is hardcoded. More information can be found [here](https://docs.vmware.com/en/VMware-Aria-Automation/8.16/Using-Automation-Orchestrator-Plugins/GUID-192A5D75-8FD5-4F2C-ADA2-590D37A413BB.html)
To support the async execution, we’ll use Promise. If the execution is successful, the promise will resolve the `session.output` If not, the Promise will reject it.
In the end, we want to close the open session.

```typescript
public executeSshCommand ( { sshHostname, sshCommand }: { sshHostname: string; sshCommand: string } ): Promise<string> {
    const encoding = "UTF-8";
    const path = "/var/lib/vco/app-server/conf/vco_key";
    const sshKeyPassword = "";
    const sshPort = 22;
    const sshUsername = "root";
    return new Promise<string>( ( resolve, reject ) => {
      const session = this.setNewSshSessions( sshHostname, sshUsername, sshPort )
      session.connectWithIdentity( path, sshKeyPassword );
      session.setEncoding( encoding );
      System.log( `Connected to ${sshHostname}` );
      System.log( `Execute '${sshCommand}' using encoding '${encoding}'` );
      try {
        session.executeCommand( sshCommand, true );
        resolve( session.output )
      } catch ( error ) {
        reject( `Failed to execute SSH command. ${session.error}` );
      } finally {
        if ( session ) {
          session.disconnect();
        }
      }
    } )
  }

```

### setNewSshSessions()

That simple method initializes and returns the `SSHSession` class. The reason I made it `private`is because it is not going to be used outside of that class.

```typescript
  private setNewSshSessions ( host: string, sshUsername: string, port: number ): SSHSession {
    return new SSHSession( host, sshUsername, port )
  }
```

## Unit-Test

Here's a breakdown of what it does:

1. Configuring Mocks:

* The `executeCommand` method of `sessionMock` is mocked to:
  * Return "Directory listing" for the expected command (sshCommand).
  * Throw an error for any other command.

2. Verifying Results:

* The test asserts that:
  * `setNewSshSessions` was called by testInstance.
  * `connectWithIdentity` was called on `sessionMock` with the correct path and password.
  * `executeCommand` was called on `sessionMock` with the expected command and set to capture output (true).
  * `disconnect` was called on sessionMock.
  * The returned result from `executeSshCommand` matches the simulated output ("Directory listing").

In summary:
This test verifies that the `executeSshCommand` function successfully connects to an SSH server, executes a specific command, retrieves the output, and disconnects properly. It also checks for error handling if an invalid command is provided.

```typescript
import { SSH } from "../actions/ssh";

describe( "sshCommands", () => {
  it( "should execute SSH command successfully", async () => {
    const testInstance = new SSH();
    const sshHostname = "example.com";
    const sshCommand = "ls -l";
    const path = "/var/lib/vco/app-server/conf/vco_key" // A static path. Should be always the same
    const sshKeyPassword = ""

    const sessionMock = {
      connectWithIdentity: jasmine.createSpy().and.callThrough(),
      executeCommand: jasmine.createSpy().and.callFake( ( command: string, _: boolean ) => {
        if ( command === sshCommand ) {
          // Simulate successful execution
          sessionMock.output = "Directory listing";
        } else {
          // Simulate an invalid command
          throw new Error( "Invalid command" );
        }
      } ),
      disconnect: jasmine.createSpy(),
      error: "Error message",
      output: "",
      setEncoding: jasmine.createSpy(),
    } as unknown as SSHSession;

    spyOn<any>( testInstance, "setNewSshSessions" ).and.returnValue( sessionMock );
    const result = await testInstance.executeSshCommand( { sshHostname, sshCommand } );

    expect( testInstance["setNewSshSessions"] ).toHaveBeenCalled();
    expect( sessionMock.connectWithIdentity ).toHaveBeenCalledWith( path, sshKeyPassword );
    expect( sessionMock.executeCommand ).toHaveBeenCalledWith( sshCommand, true );
    expect( sessionMock.disconnect ).toHaveBeenCalled();
    expect( result ).toEqual( "Directory listing" );
  } );
```

Here's a breakdown of what it does:

1. Configuring Mocks:

* The `executeCommand` method of `sessionMock` is mocked to throw an error ("Invalid command").

2. Testing for Rejection:

* It attempts to call `executeSshCommand` with the hostname and invalid command data wrapped in an object.
* It uses a try...catch block to handle the expected rejection.

In summary:
This test verifies that the `executeSshCommand` function rejects the promise when an SSH command fails (as mocked by the `executeCommand` method). It checks if the caught error message includes "Failed to execute SSH command", indicating an issue during execution.

```typescript
  it( "should reject when SSH command fails", async () => {
    const testInstance = new SSH();
    const sshHostname = "example.com";
    const invalidCommand = "invalid-command";

    const sessionMock = {
      connectWithIdentity: jasmine.createSpy(),
      executeCommand: jasmine.createSpy().and.throwError( "Invalid command" ),
      disconnect: jasmine.createSpy(),
      setEncoding: jasmine.createSpy(),
    } as unknown as SSHSession;

    spyOn<any>( testInstance, "setNewSshSessions" ).and.returnValue( sessionMock );

    try {
      await testInstance.executeSshCommand( { sshHostname, sshCommand: invalidCommand } );
      fail( "Expected rejection but promise resolved." );
    } catch ( error ) {
      expect( error ).toContain( "Failed to execute SSH command. undefined" );
    }
  } );
```

### Result

![img-description](/assets/img/vro-how-to-ssh/image.png){: .shadow }

## Usage Example

```typescript
const example = new SSH()
    const vars = {
      sshHostname: "hostname FQDN",
      sshCommand: "uptime",
    };
    example.executeSshCommand(vars)
  }
```

## Source code

The source code can be found [here](https://github.com/unbreakabl3/vmware_aria_orchestrator_examples.git)
