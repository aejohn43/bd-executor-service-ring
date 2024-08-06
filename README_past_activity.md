## ExecutorServices: Ring Device Updates Check

No AWS resources are required for this activity

### Introduction

We are working on a feature to let a customer check all of the
Ring devices associated with their account to see if any need
firmware updates. The firmware is the code that runs on the device
itself, kind of like the operating system on your laptop, but for
a doorbell, a camera, motion sensor etc.

We want the response to the customer to be fairly quick, so they
know that their devices have been checked and are being updated
if necessary.

It takes a few seconds to request the system information from a
Ring device, but if a customer has dozens of devices around their
home (or homes), this could take a while if we performed one at a
time. So we will want to run these device checks concurrently, rather
than iteratively. (However, to measure the performance savings, we will
implement it iteratively first!)

We will treat each device version check as a separate task, so we will
use an `ExecutorService` to kick off a thread for each device the
customer owns. Each task will fetch the system information from the
specified device (if possible) and determine if the device requires
an update.

We will then dive into the logic that is responsible for pushing
the firmware updates, and run the device updating inside a thread
of its own. The updates can take a few minutes, so we don't want
the customer to have to wait for the firmware updates to complete
before responding to the customer's update request.

### Phase 0: Check out the code and make sure it builds

#### Code intro

Take a look at each of these classes in the code base and
get a sense for what they do.

1. You will implement the `DeviceCheckTask` class in Phase 1.
   This class checks whether a single device can be updated, and initiates
   the update if so. You will use the `RingDeviceCommunicatorService` to
   retrieve device information, and `DeviceChecker` to initiate an update
   (more on those classes in a moment).
1. The `DeviceChecker` class in the
   `com.amazon.ata.executorservice.checker` package
   is mostly empty right now. It consolidates all the services needed to
   find and check devices.
   You will be modifying this class's methods in Phases 2-4 to actually
   do the firmware version checking and updates. To compare performance,
   you'll implement one iterative method and one concurrent method.
   Both methods will use your `DeviceCheckTask`.
1. To fetch the devices for a specific customer, you'll use the
   `CustomerService` that is passed to the `DeviceChecker` constructor.
   You should not need to modify this class or its associated
   objects.
1. To get device information and initiate updates, you'll use the
   `RingDeviceCommunicationService` that is also passed to the
   `DeviceChecker` constructor.
   You should not need to modify this class or its associated
   objects.
1. Since services' models usually don't include any logic, we've created the
   `KnownRingDeviceFirmwareVersions` utility. It includes some known version
   numbers that are only used for testing and mocking, along with a
   `needsUpdate()` method that handles all the weird edge
   cases in the &lt;major&gt;.&lt;minor&gt;.&lt;patch&gt; versioning system.
   You'll need this in Phase 1 to compare firmware versions before asking
   for a time-consuming update.

#### Try building

Run the `Phase1Test` file and make
sure the code builds but tests fail.

**GOAL:** Become familiar with the code and makes sure that it builds

Be ready to answer these questions with the class:
1. What method updates a device's firmware?
1. What method finds all a customer's devices?
1. What will the `DeviceCheckTask` have to do?
   1. What service clients does it need access to?
1. What method will the *user* call to update their devices?
   1. Which version does the firmware get updated to?

Phase 0 is complete when:
- You have found the classes mentioned above and can answer the
  questions provided.
- The code is building in your workspace (but tests are failing)

### Phase 1: Define your task

Your first step is to implement the `DeviceCheckTask` class,
which is responsible for providing a public method that:
1. Calls the `RingDeviceCommunicatorService` to determine the
   device's firmware version number
2. Decides if that version is out of date, by comparing the device's
   version to a `latestVersion`
3. Calls `deviceChecker.updateDevice()` if the device needs updating

The `DeviceCheckTask`s will eventually be submitted to an `ExecutorService`
to run in a thread managed by its thread pool.

Notes for your design:
- Does the `DeviceCheckTask` need to implement a specific
  functional interface to be submitted to the `ExecutorService`?
  Which one? Hint: the method it implements has no argument, and no
  return value.
- Despite the *method* not having any arguments, the `DeviceCheckTask`
  will still need access to the device identifier, and the
  Ring device firmware version we're looking for. You can make
  these available to the `DeviceCheckTask` by updating the **constructor**
  to accept and store the values your task will need.

**GOAL:** Implement the `DeviceCheckTask` class.

Phase 1 is complete when:
- You've implemented the `DeviceCheckTask` class according to the
  requirements above
- In particular, your `DeviceCheckTask` is ready to be passed to the
  `submit()` method on an `ExecutorService`.
- `Phase1Test` tests are passing

### Phase 2: Use your task iteratively

Implement `checkDevicesIteratively`. Feel
free to create helper methods (that can be shared with the eventual
concurrent implementation). The logical steps that need to be
performed are:
1. Use the `CustomerService` class to retrieve all devices that the
   given customer is using.
1. Iterate through your list of devices, creating a `DeviceCheckTask`
   for each one, and explicitly running it inside your loop.

Return the total number of devices found and checked.

**GOAL:** Implement the iterative version of the device-checking
          logic

Phase 2 is complete when:
- You have implemented `checkDevicesIteratively`, making use of
  `DeviceCheckTask`
- `Phase2Test` tests are passing

### Phase 3: Use your task concurrently

Implement `checkDevicesConcurrently`. Feel free to share helper
methods with `checkDevicesIteratively`, but do not run the task in
the loop!

Instead, find a way to provide the `DeviceChecker` with an
`ExecutorService` (hint: create a cached thread pool) that can
be reused for all the devices in the `checkDevicesConcurrently`
method.

Use the same logic as above, but instead of running the `DeviceCheckTask`s
directly inside a loop, submit the tasks to the `ExecutorService`.
Create and shut down the `ExcecutorService` in the method; if another user
calls the `checkDevicesConcurrently` method,
it should use a new `ExecutorService`.

**Once your concurrent implementation is complete:**

Run the `deviceChecker_checkDevicesIteratively_Timer` test from `Phase3Test`
in IntelliJ so that you can see the "logging" output. Note the time required;
the log will say "On average, checkDevicesIteratively(1)" with
a number in seconds. It should appear near the bottom of the log.

Then do the same for the `deviceChecker_checkDevicesConcurrently_Timer` test.
This time the log will say "On average, checkDevicesConcurrently(1)". Because
this test runs concurrently, the line may appear anywhere in the log.

1. Based on the test output, which implementation
   appears to be faster? About how many times faster for customer "1"?
   Compare this to the number of devices for the customer.
1. Compare the order that the `RingDeviceCommunicationsService` receives
   and responds to requests. Are they the same in the concurrent implementation
   as they are in the iterative implementation? If there is a difference
   between the two, why might that be?

Add the average times your team observed for the iterative and concurrent
implementations of DeviceChecker to the table in the digest.

**GOAL:** Implement the concurrent version of the device-checking
          logic

Phase 3 is complete when:
- You have implemented `checkDevicesConcurrently`, making use of
  `DeviceCheckTask`
- You have discussed the two questions above with your teammates, and have
  included your team's experimental results in the table in the digest.
- `Phase3Test` tests are passing

### Extension: Phase 4: Submit the update requests

If you take a look at the `updateDevice()` method that the `DeviceCheckTask`
calls on out-of-date devices, it's currently only printing to stdout.
Add the logic to actually call the `RingDeviceCommunicatorService`'s
UpdateDeviceFirmware operation. But because this operation may take a long
time, do not make this call synchronously.

Instead, create an `ExecutorService` to submit a new `Runnable` each time
`updateDevice()` is called. Don't forget to shut it down.

You may use either an explicit `Runnable` class that you implement
(similar to the `DeviceCheckTask`), or even better, you may
implement this logic using a `Runnable` lambda expression.

#### Option 1: Explicit `Runnable` class

Create a new `Runnable` class that contains a device identifier
and a firmware version. It uses these to call the
`RingDeviceCommunicatorService`'s UpdateDeviceFirmware operation.

#### Option 2: lambda expression

Submit a lambda expression directly to the `ExecutorService`. Note
that your lambda expression cannot accept any arguments (why?), so
you'll need these to be available in your lambda expression. The good
news is that local variables that are in scope where the lambda is
*defined* are in scope when the lambda *runs* as well.

One thing to be careful of: If you're reusing an object instance across
`Runnable`s, you want to be very sure you are not changing the
value from one `Runnable` to the next. So if you're using a
`RingDeviceFirmwareVersion` object in your `Runnable`, make sure that
each one sees a different instance. (more on this pitfall in Thread Safety,
a later lesson!)

Your lambda expression should call the
`RingDeviceCommunicatorService`'s UpdateDeviceFirmware operation,
passing in the device identifier, and the desired firmware version.

#### In either case...

Whichever method you choose above (and feel free to try both),
verify that you now see logging statements in your console from
the UpdateDeviceFirmware operation showing up.

**GOAL:** Inside `DeviceChecker.updateDevice()`, Use `Runnable`s to
request firmware updates for each of the out-of-date devices.

Phase 4 Extension is complete when:
- `DeviceChecker.updateDevice()` is implemented
- `Phase4Test` tests are passing
