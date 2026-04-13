# Chapter 5: NASS — Fuzzing Proprietary Native Android System Services

As the Android security model matured, the limitations of static analysis became impossible to ignore. Project Treble’s architectural reorganization pushed the most hardware-proximate, highly privileged code into the Vendor Hardware Abstraction Layer (HAL). Because vendors like Samsung, Qualcomm, and Xiaomi rarely release the source code for their specific hardware integrations, these critical HAL services are deployed as stripped, proprietary binaries.

Static systems like DIFUZE and FANS, completely reliant on LLVM bitcode or Clang ASTs, found themselves locked out of this massive attack surface. In 2025, researchers introduced **NASS** (Native Android System Services) to break this deadlock. NASS abandoned source-code analysis entirely, pioneering a dynamic, binary-only approach that leverages the universal design patterns of RPC frameworks to reverse-engineer interfaces on the fly. Furthermore, it introduced a robust coverage-collection mechanism for multi-threaded daemons, finally bringing the power of evolutionary, grey-box fuzzing to the proprietary blind spot of the Android ecosystem.

## 5.1 The Proprietary Barrier: The Camera HAL

To illustrate the challenge NASS overcomes, we follow our **Camera Module** down to its lowest userspace level. When the open-source `cameraserver` framework daemon needs to physically turn on the camera lens, it initiates a Binder IPC transaction to a proprietary vendor HAL service, such as `vendor.qti.hardware.camera`.

Because we do not have the source code for this vendor service, we do not know its transaction IDs, nor do we know the structural layout of the `Parcel` it expects. Suppose the service requires a complex, nested object—a `CameraConfig` Parcelable—to initialize the hardware. If we send a malformed `Parcel`, the service's `onTransact` method will fail to deserialize it, and the transaction will abort.

Without source code to parse, how can a fuzzer possibly know how to construct a `CameraConfig` object? NASS answers this question by turning the service's own deserialization logic against it.

## 5.2 The Universal RPC Design Principles

The foundational insight of NASS is that, regardless of how proprietary or undocumented a system service's business logic may be, its "front door" is heavily constrained by the architectural necessities of Inter-Process Communication. The researchers formalized these constraints into three universal **RPC Design Principles** (Ab, Si, St), which we introduced in Chapter 2.

In the context of an Android HAL service, these principles manifest practically:
1.  **Si (Single Entry Point):** Every transaction must pass through the `onTransact` function, providing a singular, predictable chokepoint for dynamic instrumentation.
2.  **Ab (Abstraction):** The `onTransact` stub must fully deserialize the `Parcel` *before* invoking the proprietary camera logic. If we observe the stub, we are observing pure structural validation, cleanly separated from complex application state.
3.  **St (Standard Deserialization):** Most importantly, the proprietary HAL service cannot use proprietary byte-parsing logic. To interoperate with the rest of Android, it must link against `libbinder.so` and call standard, publicly known routines like `Parcel::readInt32()` or `Parcel::readString16()`.

By dynamically monitoring the execution of these standard `St` routines within the `Si` entry point, NASS can empirically deduce the expected interface structure without ever seeing a line of source code.

## 5.3 DGIE: Dynamic Interface Recovery

The core innovation of NASS is **Deserialization-Guided Interface Extraction (DGIE)**. DGIE is an iterative, probing-based algorithm that dynamically reverse-engineers the interface signature by feeding the target service partially correct inputs and watching how it reacts.

### 5.3.1 Unrolling the Parcelable
The genius of DGIE lies in its approach to complex, high-level objects like our `CameraConfig` Parcelable. A static black-box fuzzer fails because it cannot guess the internal class structure of the object. 

NASS recognizes that at the binary level, object-oriented abstractions disappear. When a C++ stub deserializes a `CameraConfig` Parcelable, it simply executes a linear sequence of standard `read` calls. For instance, the `CameraConfig::readFromParcel()` function might compile down to a `readInt32()`, followed by a `readString16()`, followed by another `readInt32()`. DGIE does not attempt to reconstruct the `CameraConfig` class hierarchy; it simply monitors the dynamic execution flow and records the linear sequence of expected reads. By "unrolling" the complex object into a flat sequence of primitive deserializers, NASS reduces a structurally impossible guessing game into a simple observation task.

### 5.3.2 The Iterative Probing Loop
DGIE constructs the interface map through a precise feedback loop, utilizing Frida to hook the standard deserialization routines exported by `libbinder.so`.

1.  **Discovery:** NASS first iterates through all possible transaction IDs (which, in Binder, are bounded numerical values), sending empty `Parcels` and monitoring code coverage to see which IDs trigger valid execution paths in the server.
2.  **Initial Probe:** For a discovered transaction ID, NASS sends an empty request `Parcel`.
3.  **Observation:** The proprietary service's `onTransact` stub begins processing. NASS's Frida hooks observe that the service immediately calls `Parcel::readInt32()`. Because the `Parcel` is empty, the read fails, and the service aborts the transaction.
4.  **Refinement:** NASS updates its internal interface model for that transaction ID: it now knows the first required argument is a 32-bit integer.
5.  **Iteration:** NASS generates a *new* request `Parcel`, packing a valid random integer followed by an End-Of-File (EOF) marker. It sends this new `Parcel` to the service.
6.  **Subsequent Observation:** The service successfully reads the integer, advances its instruction pointer, and then calls `Parcel::readString16()`. The string read fails due to the EOF, and the transaction aborts. NASS records the string requirement and repeats the process.

This iterative loop continues until the service stops calling deserialization routines and successfully jumps into the underlying business logic. Through empirical probing, DGIE essentially forces the proprietary binary to map out its own interface requirements, achieving an 88% accuracy rate in signature recovery—vastly outperforming passive traffic-sniffing techniques.

## 5.4 Coverage Collection in Multi-Threaded Daemons

With the interface dynamically mapped, NASS can generate structurally valid payloads. However, to find deep logic bugs, it requires the evolutionary guidance of grey-box fuzzing. Collecting stable, accurate code coverage from an Android system service is notoriously difficult. These daemons are highly concurrent, multi-threaded processes that are constantly handling background requests from the operating system, creating a chaotic environment of execution "noise."

If a fuzzer simply attached a coverage tracker to the entire `cameraserver` process, the resulting bitmap would be polluted by hundreds of unrelated background threads, destroying the evolutionary algorithm's ability to correlate specific fuzzer inputs to specific code paths.

NASS solves this concurrency problem through a highly surgical application of dynamic binary instrumentation via **Frida Stalker**, tied directly to the Binder IPC semantics:
1.  **PID-Based Caller Isolation:** NASS places a hook at the very beginning of the `onTransact` entry point. When this hook triggers, it immediately inspects the kernel-provided caller credentials associated with the IPC transaction. If the caller's Process ID (PID) does not exactly match the PID of the NASS fuzzing client, the hook silently detaches, allowing normal system traffic to process without interference.
2.  **Thread-Localized Tracing:** If the PID matches, indicating that this specific transaction was generated by the fuzzer, NASS activates Frida Stalker *only on the specific thread* executing the `onTransact` function. 
3.  **Synchronous Capture:** Stalker records every basic block executed by that specific thread as it processes the fuzzer's payload. As soon as the `onTransact` function returns, completing the transaction, NASS terminates the Stalker trace.

This architecture guarantees that the coverage bitmap fed back to the evolutionary engine is perfectly isolated, representing only the synchronous execution path triggered by the fuzzer's specifically crafted input. 

## 5.5 Real-World Impact on Commercial Devices

By combining dynamic interface recovery (DGIE) with isolated, thread-localized coverage tracking, NASS successfully applied grey-box fuzzing to the most heavily guarded layer of the Android ecosystem. 

Evaluated across five modern commercial devices (including the Google Pixel 9 and Samsung Galaxy S23), NASS discovered 12 unique memory-corruption vulnerabilities, resulting in five assigned CVEs. Critically, many of these vulnerabilities resided in proprietary vendor HAL services—codebases that are entirely invisible to source-reliant tools like DIFUZE and FANS.

In one revealing case study, NASS discovered a heap buffer overflow in the proprietary `vendor.samsung.hardware.radio.network` HAL service on the Galaxy S23. Through DGIE probing, NASS unrolled a highly complex nested network message structure, determining that it required a specific sequence of seven distinct deserializers, including a signed integer representing the payload length. Once the structural barrier was bypassed, NASS's coverage-guided fuzzing engine efficiently explored the bounds-checking logic. By correlating basic-block coverage with input mutations, the fuzzer quickly discovered that passing a negative value for the length bypassed a poorly implemented size constraint, resulting in a heap overflow when the payload was subsequently copied.

This discovery highlights the profound evolution of the field: by systematically reverse-engineering the universal serialization patterns of RPC frameworks, modern fuzzing techniques can completely bypass the necessity of source code, shining a light into the proprietary blind spots that protect the lowest levels of modern mobile devices.
