# Chapter 5: Proprietary Fuzzing: NASS and Dynamic Extraction

Project Treble changed the game for security researchers by moving hardware logic into the Vendor HAL. While this modularized the system, it also hid a massive portion of the attack surface. Static analysis tools like FANS—which require source code—cannot reach these closed-source, proprietary binaries.

**NASS** (Native Android System Services), introduced in 2025 [3], responds to this limitation by moving from static analysis to dynamic instrumentation. 

## 5.1 The HAL Blind Spot

Over 60% of native system services on modern flagship devices—like the Google Pixel 9 and Samsung Galaxy S23—are proprietary. Most of these closed-source services hold elevated SELinux permissions needed for direct communication with vendor-modified kernel drivers. If an attacker exploits a privileged service through a Binder RPC call, they bypass the app sandbox and gain a foothold next to the kernel. Exploits like CVE-2024-44068 show that these services are perfect stepping stones to deeper kernel vulnerabilities.

## 5.2 Exploiting RPC Design Principles

NASS sidesteps the source-code requirement by exploiting a property common to almost all RPC frameworks: even proprietary binaries must link against standard libraries to function. Proprietary services on Android link against `libbinder.so`. When an IPC request hits the `onTransact` entry point, the server stub is forced to validate the payload using standard routines, like `Parcel::readInt32()`, before it executes any custom logic. 

The server stub basically enforces the interface definition through its execution. By dynamically monitoring the sequence of standard deserialization calls, a fuzzer can reverse-engineer the interface without ever needing source code.

An analysis of 316 proprietary services found that 89% follow these principles [3]—which is a high rate, though the dataset is mostly flagship devices. Budget-tier hardware might tell a different story.

That 11% that violates these principles is not just a rounding error. On a device with 300 services, that is roughly 35 services NASS simply cannot reach. Worse, there's no reason to think those 35 are the "boring" ones. Services complex enough to deviate from standard RPC patterns are, if anything, more likely to contain bugs—and NASS misses all of them.

## 5.3 DGIE's Probing Approach

NASS uses **Deserialization-Guided Interface Extraction (DGIE)**. This is basically an iterative probing loop that reverse-engineers signatures through dynamic analysis. It monitors how the target reacts to partially correct inputs.

### 5.3.1 Discovery
First, the probing loop sends IPC requests with various transaction IDs to see which ones trigger a response from the `onTransact` dispatcher. It monitors code coverage; new basic blocks signal a valid, exposed RPC function.

### 5.3.2 Iterative Refinement
Once a transaction is found, NASS sends a request with the valid ID but an empty payload. If the server stub tries to call a deserializer, like `readInt32()`, the operation fails, and the transaction aborts. 

NASS records that a data type is needed, appends it to a new `Parcel`, and tries again. This cycle continues until the service stops calling deserialization routines and enters the business logic. That's when you know the interface signature has been extracted.

To keep this from taking forever, NASS moves to refinement when the discovery rate drops. There is also a hard failsafe that forces the transition after 20 minutes per target.

## 5.4 Dynamically Unrolling Parcelables

DGIE is very effective at handling high-level objects. In Android, Binder data is often bundled into `Parcelable` objects. Black-box fuzzers struggle here because they can't guess the internal hierarchy of undocumented objects. 

NASS recognizes that object-oriented abstractions disappear at the binary level. When a compiled stub deserializes a `Parcelable`, it just executes a linear sequence of standard `read` operations for the object's fields. A `WiFiConfiguration` object might expect a string, an integer, and a boolean; DGIE simply records the sequence `[String16, Int32, Bool]`. 

NASS reports 88% signature-recovery accuracy [3], but you have to read that figure carefully. Services that violate the principles are excluded from the denominator, not counted as failures. The real-world rate on a population that includes principle-violating services is unknown. Still, the technique significantly outperforms passive traffic-capture methods.
