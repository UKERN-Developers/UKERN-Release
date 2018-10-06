![](https://ukern.exploitation.cool/ukern.png)

**Author** :

S. Voigtlander

**Date** :

 September, 2018

**Location** :

Eindhoven,

 Fontys School for Information and Communication Technology



















# Table of contents

1. CONCEPT
2. DESIGN INTERNALS
3. LOGIC
4. REQUIRMENTS
5. STRUCTURE
6. ENTITY RELATIONSHIP DIAGRAMS
7. ARCHITECTURE
8. DATABASE DESIGN
9. USE CASES
10. TESTPLAN

# Concept

UKERN is an acronym for user land kernel.

It aims to provide a high-level layer that operates just like a kernel, but running from within a sandboxed app container.

UKERN mainly focusses on getting one programmable, accessible and fairly open modular interface inside a sandboxed system. It is designed mainly for developers and researchers of the iOS Operating System and users who are missing administrative features on their devices.

UKERN&#39;s strength is of that small solutions can be written such as extensions to the system for server-side capabilities or RFC protocols. UKERN does not follow the design guidelines of Apple Inc. but never was intended to. It opens up for developers to code binaries in the native language of the iOS operating system: C, C++ and Objective-C. Unlike other existing open environments that are created for iOS, it does not depend on specific system features or vulnerabilities and exploits. It fully integrates with iOS&#39; ecosystem and the restrictions of the sandbox still apply. To simulate a Unix-like system and file structure, kernel like features are written to simulate the authentication of users and process creation. In UKERN a highly insecure but extremely useful feature is integrated for executing arbitrary c functions, interpreted by name and arguments.

This makes UKERN unique in the way that it can use all loaded library functions in a user controllable environment.

UKERN by design is highly insecure but because it is designed for the highly restricted sandboxed environment no vulnerabilities exist of where private user data from the iOS System exists.

UKERN uses low latency system API&#39;s to get the best experience and performance, therefore UKERN is currently limited only to the iOS Sandbox and the iOS System but in the future support for other sandboxed environments can be added.

As UKERN&#39;s core principal is to bring more freedom to sandboxed environments UKERN should never be used for financial gain and therefore is provided free and open source under the MIT license.



# UKern design internals

UKERN is written in ANSI C, C++ and Objective-C.

It makes use of under more Capstone, liblorgnette and the OpenSSL library that provide basic analytic, cryptographic and logic functions that UKERN depends on.

Liblorgnette, a library for address space and library related analysis is at this time is designed for mac and iOS but a replacement could be written for Windows and other platforms as well.

## The loader

UKERN uses the MACH-O format and the dynamic library loader to load custom extensions into the runtime at load, this not only opens up for developers to code their own extensions to the app but also forms a great way for hijacking system library calls and debugging them. It is up to the developer of these extensions to choose what category the extension should be in.

In the initialization stage UKERN defines where to log standard output to, this by default is a file that is read into a buffer upon the detection of a write event to the file system.

UKERN chose this way because it makes it easier for software engineers to debug their processes, for security this might be rather insecure because software may disclose user data in the output logs.

However, no person will be able to read out this information as this is stored in the secure file storage of the sandbox.

Loading extensions works by mapping the files into the address space at runtime, but some requirements come with it because of Apple&#39;s code signing enforcement.

A breakout for code signing is being worked on that will use Return Oriented Programming and a custom Mach-O loader to execute arbitrary code in the address space, however, to Apple Inc. this is considered a creditable vulnerability eligible for bounty. I do not have plans to claim this bounty but understand that this code signing requirement needs to be there and that my application should not depend on bypasses for it.

Therefore I came up with the following problem:

- --Developer License holders of Apple are able to sign and distribute libraries and apps through websites using the iTunes services protocol to install them
- --SSL encryption on the host is required for distribution
- --An application from the same developer with the same bundle ID as an installed Application from the same developer is updated rather than installed over, which leaves the previous user data there.
- --You cannot launch code signed libraries from within the documents directory or temporary directory of the sandbox, the libraries need to be in the same directory as the App container which is marked as non-writable after installation.

The solution for this is as following:

- --License holders of Apple Inc. can install their exported signing certificate bundle and private key into the UKERN application.
- --UKERN integrates an Webserver with SSL support and the user is asked and required to install a self-signed SSL certificate, this is to match the requirement of having an SSL encryption for installing code signed web distributed apps
- --UKERN contains a writeable copy of itself in the documents directory of the app, this copy is downloaded directly from GitHub releases so that always the latest version will be used. The application will be (gpg, **NOT** code signed) signed by me, [s.voigtlander@jailed.ml](mailto:s.voigtlander@jailed.ml) , to verify the integrity of the download.
- --New modules can be downloaded into the documents directory of UKERN&#39;s app bundle.
- --A new build of UKERN with the same bundle ID is made with the dynamic libraries signed into the new application that is in the documents directory. This is distributed over the local webserver with ssl and UKERN will request the user to install the new application.
- --The new UKERN application is installed over the old UKERN application and the user now has UKERN with the old user data still intact and the new modules installed.

# The logic

## Process simulation

Because the sandbox in iOS does not allow the execution or even mapping of executable files for execution, forking or even virtual forks, UKERN works with thread based process separation. To simulate an actual process environment, logic is written for simulating the real kernel&#39;s logic of process creation.

A system call is used to read out the identifier of a thread and this automatically managed number is then used as the process ID.

Process structures have been simulated as well making UKERN support processes running with different simulated privileges and even sub processes or forked processes are possible.

Killing processes works by getting the thread id of a process by ID or name, which is done via a lookup in the global process listing table and then getting a reference to the thread by thread id, killing the thread with an implementation of pthread\_kill.

When time and support are, the simulation of MACH task ports will also be written so that processes can manage each other&#39;s fake stack / address space. Because of the layout of a task, this will work by still using the real process its task but locking the functionality limited to only the memory belonging to a certain thread.

UKERN supports getting the processor register values for threads by using an implementation of mach\_thread\_self and mach functionality for getting thread states.

This is great for debugging execution between fake (simulated) processes.

## Privilege simulation

Just like in any kernel privileges should be separated, but this not fully possible due to sandbox limitations, yet the simulation of it is possible and therefore makes a more comfortable, but **insecure** environment where users indeed need to enter there password for certain operations such as killing processes running under the fake &quot;root&quot; user, but through memory no separation is possible due to the design of a processor and the limitation of working in the same process it&#39;s address space inside of the sandbox.

The password of the user is stored in a hashing database, /etc/passwd on the simulated file system structure.

The hashing algorithm used is SHA512 and the implementations of the algorithm are provided by the OpenSSL foundation in libssl for iOS.

This hash is loaded into a structure upon initiating a login session request, in theory this makes the application vulnerable to timing attacks as well for enumerating the users if from a remote server-side authentication perspective such as via the SSH protocol, yet this is not seen as a major issue.

## Code Execution

Code Execution is done using an algorithm written on top of liblorgnette.

A symbol is lookup in the address space of the real process when a request to run a binary is done, if it is not in the process space, UKERN will try to load a dynamic library with that name from the extensions path, if no dylib is found with the symbol UKERN will return that the request to execute has failed.

Otherwise the address (binaryname) \_main entry is looked up and a function pointer with arbitrary arguments is set to that address, the parsed arguments are stored in an argv[] buffer and passed to the function and then the function is executed.

The addition of \_main to the &quot;command&quot; (in total it makes: command\_main) ensures that no arbitrary function can be called.

UKERN has its own improved implementations of strcat for merging strings, based on an Objective-C bridge.

Binaries are spawned as the user they are ran with, there is no privilege protection written for reading / writing to files yet but in the future this can be done when the virtual file system of UKERN reaches its final development state.

## Simulated (Virtual) File System

The structure of a virtual file system is maintained in UKERN as a wrapper to virtual or real storage devices.

The root file system follows the **UNIX** standards for directory structure but will also inherit the macOS structure.

Protection is not a primary goal as each process in UKERN&#39;s address space has the ability to modify those files.

## Network protocols and services

Network protocols primarily used by UKERN are TCP and solely for the purpose of remote debugging, testing and for accessibility through a remote shell.

## Launch daemons

Launch daemons are background tasks that are created at boot.











# Requirements

**MUST HAVE:**

- UKERN must be able to create process-structures similar to those in the XNU kernel
- UKERN must be able to call functions using the system() interface for executing commands or binary functions that are located in the same address space
- UKERN must be able to simulate user and kernel credentials for processes.
- UKERN must be able to run multiple processes at the same runtime.
- UKERN must be able to control and communicate to processes
- UKERN must be able to keep track of process identifiers of processes and must be able to get a reference to the corresponding process
- UKERN must be able to load and execute binary extensions
- UKERN must be able to get the main entry from a binary extension
- --UKERN must be able to get the processor register information in real-time

**SHOULD HAVE:**

- UKERN should be able to simulate a fully accessible file system structure
- UKERN should be able to control the simulated file system structure via a virtual file system
- UKERN should be able to dynamically manage allocations and free memory
- UKERN should have interfaces for performing encryption via RSA-4096
- UKERN should have interfaces for performing hashing via SHA512, SHA256, SHA1 and MD5
- UKERN should be able to connect with an endpoint for requesting updates
- UKERN should be able to launch predefined background processes upon initialization stage
- UKERN should be able to lock functions to certain users or credentials only.

**COULD HAVE:**

- UKERN could have the ability to, conforming to the MACH standards, control other processes via a task port and task.
- UKERN could be able to compress memory in the address space when limited space is left.
- UKERN could be able to communicate with real kernel drivers through a simple interface
- UKERN could be able to cache symbols of commonly used functions into an sqlite3 database
- UKERN could be able to map functions and files at a fixed address.
- UKERN could be able to backup user-data PGP-encrypted to a remote server via an API
- UKERN could be able to get update information via an API

**WONT HAVE:**

- UKERN will not focus on implementing global security standards due to it being designed mainly for debugging and development purposes, thanks to the sandbox in which it is executed no personal data will be affected by possible vulnerabilities.

**NON-FUNCTIONAL:**

- UKERN should be able to run inside Apple&#39;s sandbox
- UKERN should not use any exploits or rely on vulnerabilities in Apple's Operating System
- UKERN should be well-documented so that it can be re-used or developed further by any developer
- UKERN should not be used for commercial purposes
- UKERN's algorithms should be written in c and there should be Objective-C classes wrapping the data structures.
- UKERN should be able to run on any iOS device.
- UKERN should have documentation available via the web including the source code of which the syntax should be highlighted

















# Use cases

**ACTORS:**

1. The system
2. A process
3. The file system
4. Signals
5. The user
6. API

**THE SYSTEM**

The system (kernel) is always running and is the application itself.

Based on logical algorithms it can decide to perform operations without the user invoking them.

The system may communicate with other actors, update database information or change the flow of the application&#39;s logic.

**PROCES**

Processes are contexts that contain functions executed separately at the same time inside the system.

A process may communicate with other actors thus invoking actions but a process itself is mainly executing kernel or user defined modules.

**FILESYSTEM**

The file system is only instructed to do what the kernel commands it to do, it is running in kernel context and should not be mistaken as an actual stand-alone actor, yet the file system has support for performing operations upon file system event detection and therefore still is an actor in applicable use cases.

**SIGNAL**

A signal can be invoked by any actor in the program but may also be sent by the CPU or overlaying real kernel.

SIGNALS have predefined behavior and will communicate with the kernel to execute this behavior according to the context of the current state of the application.

**USER**

A user can by design communicate with all other actors but restrictions may apply.

**API**

The API on the remote side may invoke the kernel to perform an update or extends operation which is used in maintenance context, however most of the time the API will not contribute to the program as being an actor.

**Diagram**















# Test Plan

##### UKERN SYSTEM INTERFACE (KERNEL)

##### **Legend**

RQ = Requirement

TC = Test Case

UT = Unit Test

ITC = Integration Test Case

##### **Introduction**

This test plan focusses mainly on the operability of the primary components and requirements of the system.

###### **Goal**

_The test is meant to prove the system&#39;s primary requirements are met and function properly and conform the guidelines._

###### **Scope**

_ The core of the system with all its components. Per Example: The extension loader, the process creation and process management functionality._

###### **Performance**** measurements**:

_The system should never enter an endless loop in which the user is unable to interact with the system or the system is unable to provide and execute its primary requirements._

###### **Schedule**

_Each individual component should be tested according to this test plan, the time for those tests is not defined._

###### **Resources**

- _The documentation at_ [_https://ukern.exploitation.cool/docs_](https://ukern.exploitation.cool/docs)
- _The design guidelines at_ [_https://ukern.exploitation.cool/docs/designguidelines.pdf_](https://ukern.exploitation.cool/docs/designguidelines.pdf)
- _The concept at_ [_https://ukern.exploitation.cool/docs/concept.docx_](https://ukern.exploitation.cool/docs/concept.docx)

###### **Out**** of ****Scope**

_Security is in this test plan out of scope, memory leaks that are not fatal to the performance and functionality of primary modules are out of scope as well._

###### **Required**** resources**

- A computer running any form of macOS with Xcode
- A physical iOS device and/or an iPhone Simulator
- The project for testing available at [https://ukern.exploitation.cool/sources/testsource-latest.zip](https://ukern.exploitation.cool/sources/testsource-latest.zip)

###### **Dependencies**

_The system depends on an environment capable of running iOS higher than 10.3.3_

_The system depends on the ability to use dynamic loader functionality such as dlsym and specific mach system calls._

###### **Risks**

_The main risks are in the allocation and creation of processes as they are user controllable and use components bridged from Objective-C to C. This may lead to bugs in the management of the memory and these bugs usually can be identified by a crash log mentioning_ **KERN\_INVALID\_ADDR at** _._

_These crash logs can be accessed through the Settings application on iOS under Privacy\&gt;Analytics._

_The log can be identified by the name of the application (UKERN)._

_Please include these log files if available when submitting a bug report, log files mentioning_ **&quot;0x8badf00d&quot;** _are out of scope._

###### **Exclusions**

_The liblorgnette component will not be included in this test plan._

_The utilities component will not be included in this test plan._

##### **Test Coverage**

The tests will cover at least all primary requirements and will also test user interaction, they will not be limited to the system itself.

##### **Test Matrix**

_&#39;Just follow the white rabbit.&#39;_ **– Morpheus (From: The Matrix)**

(Image not included yet)

##### **Test Cases**

###### **TC1**

###### Title

_The system must be able to create process-structures similar to those in the XNU kernel_

###### Actors

_System, User_

###### Definitions

_Command: A one-line instruction that is parsed to a binary/c function name and arguments._

###### Description

_The user enters a command._

_The system allocates a process structure with arguments and passes that to a thread allocation function and calls the binary/c function entry point with the arguments._

###### Expected result

The system spawns a process and executes the provided function and arguments that were parsed from the user&#39;s command. **(1)(2)**

###### Exception

1. The user entered invalid arguments, the system returns null and notifies the user the command has not been found.
2. The system cannot find the binary parsed from the command, the system notifies the user that the command has not been found

###### Results

\_

\_

\_

\_

\_

\_

\_

\_

\_

\_

\_

\_

\_

\_

##### **TC2**

###### Title

###### Actors

###### Definitions

###### Description

###### Expected Result

###### Exceptions

###### Results
