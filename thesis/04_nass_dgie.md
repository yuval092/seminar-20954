# Chapter 4: NASS and Dynamic Interface Extraction

The introduction of Project Treble in Android 8 significantly altered the security research landscape. By isolating hardware-specific code within the Vendor Hardware Abstraction Layer (HAL) and standardizing communication over Binder IPC, Google increased the platform's modularity. However, this architectural shift also obscured a substantial portion of the attack surface. As discussed in the previous chapter, static analysis tools like DIFUZE and FANS—which require open-source C++ code—became ineffective against this newly compartmentalized, proprietary logic.

To audit this hidden layer, researchers introduced **NASS** (Native Android System Services) in 2025 [3]. NASS marks a necessary methodological evolution: abandoning source-code analysis in favor of dynamic binary instrumentation. This chapter examines the significance of the proprietary attack surface and details NASS's core innovation: Deserialization-Guided Interface Extraction (DGIE).

## 4.1 The Proprietary Vendor HAL: The Modern Attack Surface

The impact of the "open-source blind spot" is considerable. Analysis of modern Commercial Off-The-Shelf (COTS) devices—such as the Google Pixel 9 or Samsung Galaxy S23—reveals that over 60% of running native system services are entirely proprietary [3]. The majority of these closed-source services operate within the HAL.

HAL services are highly attractive targets for attackers seeking privilege escalation. They possess the elevated SELinux permissions required to interact directly with hardware-specific, vendor-modified kernel drivers. If an attacker confined within the restricted app sandbox can trigger a memory-corruption vulnerability via a Binder RPC call to a proprietary HAL service, they can compromise that service. This bypasses the app sandbox and provides proximity to the kernel.

This threat is demonstrably real. Exploits such as CVE-2024-44068 have utilized system services (e.g., `cameraserver`) as intermediate steps to reach deeper kernel vulnerabilities [3]. Developing methods to secure these proprietary binaries is therefore critical.

## 4.2 Exploiting RPC Design Principles

How can a fuzzer determine the layout of a complex, nested Binder `Parcel` without source code access? NASS achieves this by leveraging the universal RPC Design Principles introduced in Chapter 2: Abstraction (Ab), Single Entry Point (Si), and Standard Deserialization (St).

While a vendor's implementation for processing camera sensor data may be proprietary, the mechanism for receiving that data over Binder is standardized. To function within the Android OS, a proprietary binary must link against standard Android libraries (such as `libbinder.so`). When an IPC request arrives at the single entry point (`onTransact`), the auto-generated server stub must validate the payload using standard deserialization routines (e.g., `Parcel::readInt32()`) before executing the proprietary logic.

The central insight of NASS is that the server stub actively enforces the interface definition during execution. By dynamically observing the sequence of standard deserialization calls, a fuzzer can empirically deduce the interface structure without requiring source code.

### 4.2.1 Empirical Validation of RPC Principles

To validate this approach, the developers of NASS conducted an extensive study of 316 proprietary native services across five modern COTS devices. They manually audited the server stubs of these closed-source binaries to assess compliance with the Ab, Si, and St principles.

The results confirmed the viability of the approach: 89% of the analyzed proprietary services strictly adhered to all three RPC design principles [3]. 

The minority of services that deviated typically violated the Abstraction (Ab) rule—for instance, by embedding application-specific permission checks directly within the auto-generated deserialization stub. A smaller subset violated the Standardization (St) rule by implementing custom byte-parsing logic instead of utilizing standard `libbinder.so` routines. Nevertheless, the high rate of compliance across different hardware vendors demonstrated that a generalized, dynamic extraction technique could effectively map the majority of the modern attack surface.

## 4.3 Deserialization-Guided Interface Extraction (DGIE)

NASS operationalizes this observation strategy through an algorithm called **Deserialization-Guided Interface Extraction (DGIE)**. DGIE is an iterative, dynamic probing mechanism that essentially reverse-engineers the interface signature. It operates by providing the target service with partially correct inputs and using Dynamic Binary Instrumentation (DBI) to monitor the service's response.

### 4.3.1 The Probing State Machine and Refinement Heuristics

The DGIE process consists of two distinct phases: a preliminary fuzzing phase to identify exposed RPC functions and trigger their entry points, followed by a refinement phase to map their specific deserializers.

Consider the process of reverse-engineering a specific transaction (e.g., `CONFIGURE_SENSOR`) on a proprietary `vendor.camera.hal` service. The DGIE refinement process functions as an iterative state machine:

```mermaid
graph TD
    A[Start: Generate Empty Parcel with target Tx ID] --> B(Send Parcel via IPC)
    B --> C{Hook onTransact via Frida}
    C -->|Stub calls readInt32()| D[Record Requirement: Int32]
    D --> E[Transaction Aborts due to EOF]
    E --> F[Generate New Parcel: Append Int32 + EOF]
    F --> B
    
    C -->|Stub calls readString16()| G[Record Requirement: String16]
    G --> H[Transaction Aborts due to EOF]
    H --> I[Generate New Parcel: Append String16 + EOF]
    I --> B
    
    C -->|No more reads, Logic Executes| J[Interface Fully Extracted]
    
    style A fill:#e6e6fa,stroke:#333,stroke-width:2px
    style J fill:#98fb98,stroke:#333,stroke-width:2px
```

NASS continuously appends the requisite data types to the `Parcel` until the service ceases calling deserialization routines and transitions to the underlying business logic. 

To maintain efficiency, NASS employs time-based heuristics to determine when an interface has been fully mapped. During the preliminary fuzzing phase, NASS monitors the discovery rate of new seeds, establishing a baseline during the first two minutes. It then compares this baseline against subsequent two-minute intervals. If the discovery rate drops to ten times lower than the initial baseline, NASS assumes the server stub's deserialization paths are exhausted and transitions to the refinement phase. Additionally, to prevent stalling on overly complex interfaces, NASS enforces a hard limit, transitioning to refinement after 20 minutes of fuzzing per target [3].

## 4.4 Unrolling Complex Parcelables Dynamically

DGIE is particularly effective when analyzing complex, high-level objects. In Android, such data structures transmitted over Binder are implemented as `Parcelable` objects. A static, black-box fuzzer struggles here, as it cannot infer the internal class hierarchy or member variables of a proprietary, undocumented `CameraConfig` Parcelable.

NASS overcomes this by recognizing that at the binary execution level, object-oriented abstractions are reduced to sequential instructions. When a compiled C++ stub deserializes a `CameraConfig` Parcelable, it executes a linear sequence of standard `read` calls corresponding to the object's internal fields. 

The proprietary C++ implementation of `CameraConfig::readFromParcel()` might execute as follows:

```cpp
// Proprietary logic inside vendor HAL
status_t CameraConfig::readFromParcel(const Parcel* parcel) {
    status_t err;
    
    // NASS Hook 1 observes: readInt32
    err = parcel->readInt32(&this->resolution_x);
    if (err != NO_ERROR) return err;

    // NASS Hook 2 observes: readInt32
    err = parcel->readInt32(&this->resolution_y);
    if (err != NO_ERROR) return err;

    // NASS Hook 3 observes: readString16
    err = parcel->readString16(&this->hardware_id);
    if (err != NO_ERROR) return err;

    return NO_ERROR;
}
```

DGIE does not attempt to reconstruct the `CameraConfig` class hierarchy or determine variable names (`resolution_x`, `hardware_id`). It simply monitors the dynamic execution flow via Frida and records the sequence of expected reads: `[Int32, Int32, String16]`.

By "unrolling" this complex object into a flat list of primitive deserializers, NASS transforms a complex structural estimation problem into a straightforward observation task. This unrolling enables NASS to generate correctly formatted `Parcel` payloads that pass the proprietary stub's validation checks. Evaluations demonstrated an 88% accuracy rate in signature recovery—significantly outperforming passive traffic-sniffing techniques used in previous tools (e.g., BinderCracker) [3].

With the interface dynamically mapped, the structural barrier is bypassed. However, discovering deep logic vulnerabilities requires leveraging this knowledge to drive an evolutionary fuzzing loop. This necessitates precise control over multi-threaded execution environments, which is the focus of the next chapter.