# Chapter 5: Proprietary Service Fuzzing: NASS and Dynamic Extraction

Project Treble changed the landscape for security researchers by isolating hardware-specific logic inside the Vendor Hardware Abstraction Layer (HAL). While this modularized the system, it also concealed a large portion of the attack surface. Static analysis tools like DIFUZE and FANS—dependent on source code—cannot reach the compartmentalized, proprietary binaries that now populate this layer.

**NASS** (Native Android System Services), introduced in 2025 [3], responds to this limitation by moving from static analysis to dynamic binary instrumentation. 

## 5.1 The Proprietary Vendor HAL: The Modern Attack Surface

Over 60% of native system services on modern Commercial Off-The-Shelf (COTS) devices—such as the Google Pixel 9 and Samsung Galaxy S23—are proprietary [3]. Most of these closed-source services operate within the HAL, holding the elevated SELinux permissions needed for direct communication with vendor-modified kernel drivers. If an attacker in a restricted sandbox exploits a privileged service through a Binder RPC call, they bypass the app sandbox and gain a foothold next to the kernel. Exploits like CVE-2024-44068 demonstrate that these services are practical stepping stones to deeper kernel vulnerabilities.

## 5.2 Exploiting RPC Design Principles

NASS sidesteps the source-code requirement by exploiting a property common to nearly all RPC frameworks: even proprietary binaries must link against standard libraries to function. Proprietary services on Android link against `libbinder.so`. When an IPC request hits the `onTransact` entry point, the server stub is forced to validate the payload using standard routines, such as `Parcel::readInt32()`, before executing any proprietary logic. 

The server stub enforces the interface definition through its code execution. By dynamically monitoring the sequence of standard deserialization calls, a fuzzer can reverse-engineer the interface without source code.

An analysis of 316 proprietary services across five devices found that 89% follow these principles [3]—a high rate, though the sample is skewed toward flagship devices from major vendors. Budget-tier hardware, where ODM software quality varies significantly, may tell a different story.

The 11% that violate the standard principles are not a statistical rounding error. On a device with 316 services, that is roughly 35 services where DGIE provides no coverage. These are likely the most complex—and possibly most buggy—of the lot.

## 5.3 Deserialization-Guided Interface Extraction (DGIE)

NASS utilizes **Deserialization-Guided Interface Extraction (DGIE)**, an iterative probing loop that reverse-engineers signatures through dynamic analysis. It monitors how the target service reacts to partially correct inputs.

### 5.3.1 Phase 1: Preliminary Fuzzing (Discovery)
In the discovery phase, NASS sends IPC requests with various transaction IDs to identify which trigger responses from the `onTransact` dispatcher. It monitors code coverage; new basic blocks signal a valid, exposed RPC function.

### 5.3.2 Phase 2: Iterative Refinement
Once a transaction is identified, NASS sends a request with the valid ID but an empty payload. If the server stub attempts to invoke a deserializer, like `readInt32()`, the operation fails, and the transaction aborts. NASS records the required data type, appends it to a new `Parcel`, and repeats the request. This cycle continues until the service stops calling deserialization routines and enters the business logic, signaling that the interface signature is extracted.

To manage execution time, NASS moves to refinement when the discovery rate drops significantly below the initial baseline. A hard failsafe forces this transition after 20 minutes per target [3].

## 5.4 Unrolling Complex Parcelables Dynamically

DGIE is effective at handling high-level objects. In Android, Binder data is often bundled into `Parcelable` objects. Black-box fuzzers struggle here because they cannot guess the internal hierarchy or member variables of undocumented objects. 

NASS recognizes that object-oriented abstractions disappear at the binary level. When a compiled stub deserializes a `Parcelable`, it executes a linear sequence of standard `read` operations for the object's fields. A `WiFiConfiguration` object might expect a string, an integer, and a boolean; DGIE simply records the sequence `[String16, Int32, Bool]`. 

NASS reports 88% signature-recovery accuracy [3], but this figure requires careful reading. Services that violate the design principles DGIE relies on are excluded from the denominator, not counted as failures. The real-world rate on a population that includes principle-violating services is unknown. This is not a flaw in the paper—it is a standard evaluation scoping decision—but it is worth keeping in mind when projecting NASS's coverage onto an arbitrary device. nonetheless, the technique significantly outperforms passive traffic-capture methods.
