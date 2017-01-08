---
title: Useful things to do with libdispatch in Swift 3
description: How to do a "dispatch_after" in swift 3 with nice syntax, and dealing with crazy callbacks when setting up a concurrent dispatch queue.
---

## Snippet: Some useful solutions for libdispatch in Swift 3

### Problem: You need to dispatch after some amount of time

**Solution:**

Use `DispatchTime` and `DispatchTimeInterval` for a readable time to wait.

Looking at the new interface for libdispatch in Swift 3, you might notice that there is a handy `enum` called `DispatchTimeInterval` that looks like this:

```swift
public enum DispatchTimeInterval {

    case seconds(Int)

    case milliseconds(Int)

    case microseconds(Int)

    case nanoseconds(Int)
}
```

But you might also notice that `DispatchQueue.asyncAfter` doesn't take this enum. Not to fear however, as when we look further in to the new libdispatch swift interface, we find these helpful methods:

```swift
public func +(time: DispatchTime, interval: DispatchTimeInterval) -> DispatchTime
public func +(time: DispatchTime, seconds: Double) -> DispatchTime

public func -(time: DispatchTime, interval: DispatchTimeInterval) -> DispatchTime
public func -(time: DispatchTime, seconds: Double) -> DispatchTime
```

So when you want to use these together, you might write something that looks like this:

```swift
DispatchQueue.main.asyncAfter(deadline: .now() + .seconds(1)) {
    // Do some work
}
```

And of course, if this is still too much typing, you can always add a little dollop of syntax sugar (but please just pick _one_):

```swift
extension DispatchTime {
    /** Allow a dispatch time to be created using a `DispatchTimeInterval` since now.

    Example:
    DispatchTime.after(.seconds(10))
    */
    public static func after(_ interval: DispatchTimeInterval) -> DispatchTime {
        return .now() + interval
    }

    /** Allow a dispatch time to be created using a `DispatchTimeInterval` since now.

    Example:
    DispatchTime.in(10, interval: DispatchTimeInterval.seconds)
    */
    public static func `in`(_ int: Int, interval: (Int) -> DispatchTimeInterval) -> DispatchTime {
        return .now() + interval(int)
    }

}

let deadline    = DispatchTime.after(.seconds(10))
let altDeadline = DispatchTime.in(10, interval: DispatchTimeInterval.seconds)
```

### Problem: You need to wait for some library that you don't control to do some async setup before sending it work

**Solution:**

Use a `DispatchWorkItem` with the `.barrier` flag and a `DispatchSemaphore`

A less common case, but something that I've experienced: You want to do some
heavy setup asynchronously, but that setup depends on third-party code that
also has async setup.

You don't want any work to be done until setup is complete both for your code,
and your library. However, after the setup is done, work can be done in
parallel, so it doesn't make sense to have a serial queue either.

This is where a combination of a semaphore and dispatch barrier may be useful.
You can create a semaphore to synchronize async work that you don't control and
wait until it's all completed.

First, what is a dispatch barrier? If you look at the documentation in Swift, you might notice the description is:

> No Description

How... helpful.

Switching over to Objective-C, we get more details:

> A dispatch barrier allows you to create a synchronization point within a concurrent dispatch queue. When it encounters a barrier, a concurrent queue delays the execution of the barrier block (or any further blocks) until all blocks submitted before the barrier finish executing. At that point, the barrier block executes by itself. Upon completion, the queue resumes its normal execution behavior.

We can use this to ensure that we don't let work be done until setup is
complete while also having our dispatch queue configured to be concurrent.

Here's a very dense example of:

1. Creating a concurrent work queue
2. Adding an item that blocks all other work until it completes
3. Waits to complete on _another_ thread that is not controlled by our app
4. Does additional configuration
5. Allows the queue to start doing work concurrently

```swift
// Create a custom queue that you can dispatch work to
let thirdPartyWorkQueue = DispatchQueue(label: "com.example.third-party-work-queue", qos: .utility)

// Create a dispatch work item with the `.barrier` flag
let thirdPartyLibSetup = DispatchWorkItem(flags: .barrier) {
    customInitialization()

    let waitForThirdPartySetup = DispatchSemaphore(value: 0)

    // initialize the BlahBlah library
    startBlahBlah(apiKey: "supersecret") { success in
      // This is a callback *after* setup has happened on some other private
      // queue, so we need to wait for it to be done
      //
      // `signal` sets a +1 count on our semaphore, where 0 means done
      waitForThirdPartySetup.signal()
    }

    // Wait for the callback indicating that the other thread we don't control
    // has finished doing the setup for the other library
    //
    // `wait` decrements the count of the semaphore, so until the signal increments it
    // back to zero, this thread will be blocked.
    waitForThirdPartySetup.wait()

    // At this point, setup is complete but the queue we're on is still blocked
    //until we finish the setup we want to do
    moreCustomizationBasedOnLibrary() 

    // Once we reach this point, the `barrier` will stop blocking other work on the queue and it can resume being non-serial
}

thirdPartyWorkQueue.async(execute: thirdPartyLibSetup)
```

There's a lot going on here, and there are definitely cases where this might
not work well (for example, if the callback here was to try to dispatch back to
the calling thread, we'd be deadlocked). This is a pretty complex chunk of
code, so please carefully consider if it works for your use case and ideally,
spend some time with the documentation of libdispatch to make sure you
understand the concepts at work here.

[The documentation is pretty great (and the concepts are well described in the Objective-C documentation).](https://developer.apple.com/reference/dispatch)


