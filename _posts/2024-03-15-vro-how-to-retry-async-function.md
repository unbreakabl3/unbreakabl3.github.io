---
layout: post
title: How to retry the async function on vRO?
date: "2024-03-18"
categories: [VMware, Aria Orchestrator]
tags: [vmware, building_tools, typescript, aria_orchestrator, async_await]
---

Hey there! Do you happen to know how we can retry an async function on vRO? Let's figure it out.

## Problem

Let’s say we have some async function `myFunction` that we want to execute multiple times until we get the expected result.

```typescript
public myFunction = async ( varA: string ) => {
  return await someAsyncFunction( varA )
}
```

## Solution

To achieve this goal, we need a retry function. Let’s see how we can make one.

1. The function takes the following parameters:
   - `func`: An asynchronous function that will be retried.
   - `retries`: The maximum number of retries (default is 5).
   - `interval`: The interval (in milliseconds) between retries (default is 10,000 ms or 10 seconds).
   - `progressive`: A boolean flag indicating whether to use progressive backoff for the interval (default is false).
2. It tries to execute the provided function (func) using await.
3. If an error occurs during execution, it checks if there are retries left:
   - If retries are available, it logs the retry number, waits for the specified interval, and recursively calls itself with reduced retries and possibly an increased interval (if progressive is true).
   - If no retries are left, it throws an error indicating that the maximum retries have been reached for the given function.

```typescript
public async retryPromise<T>(
    func: () => Promise<T>,
    retries: number = 5,
    interval: number = 10000,
    progressive: boolean = false
  ): Promise<T> {
    try {
      return await func()
    } catch ( error ) {
      if ( retries ) {
        System.log( `Retry number ${retries}` )
        await new Promise( ( resolve ) => resolve( System.sleep( interval ) ) )
        return this.retryPromise( func, retries - 1, progressive ? interval * 2 : interval, progressive )
      } else throw new Error( `Max retries reached for function ${func.name}` )
    }
}
```

## Implementation example

In that example, we're executing a function `myFunction` 20 times, with interval of 60 seconds without progressive.

```typescript
public async main () {
  return this.retryPromise( () => this.myFunction( varA ), 20, 60000, false )
}
```
