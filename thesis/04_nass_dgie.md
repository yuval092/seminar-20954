# Chapter 4: NASS and Dynamic Interface Extraction

The introduction of Project Treble in Android 8 really shook up the security research landscape. By isolating hardware-specific code within the Vendor Hardware Abstraction Layer (HAL) and standardizing communication over Binder IPC, Google made the platform much more modular. However, this architectural shift also inadvertently obscured a massive portion of the attack surface. As I discussed in the previous chapter, static analysis tools like DIFUZE and FANS—which completely depend on having open-source C++ code to compile—became basically ineffective against this newly compartmentalized, proprietary logic.

To audit this hidden layer, researchers developed **NASS** (Native Android System Services) in 2025 [3]. NASS represents a necessary evolutionary step: abandoning source-code analysis entirely in favor of dynamic binary instrumentation. This chapter examines why the proprietary attack surface is so important and details NASS's core innovation: Deserialization-Guided Interface Extraction (DGIE).

## 4.1 The Proprietary Vendor HAL: The Modern Attack Surface

It's hard to overstate the impact of the "open-source blind spot". If you look at modern Commercial Off-The-Shelf (COTS) devices—like the Google Pixel 9 or the Samsung Galaxy S23—you'll find that over 60% of all running native system services are entirely proprietary [3]. And the vast majority of these closed-source services operate right inside the HAL.

HAL services are incredibly attractive targets for attackers seeking privilege escalation. By design, they possess the elevated SELinux permissions required to interact directly with hardware-specific, vendor-modified kernel drivers. If an attacker stuck inside the restricted app sandbox can trigger a memory-corruption vulnerability via a Binder RPC call to a proprietary HAL service, they can compromise that service. This bypasses the app sandbox and provides a solid foothold right next to the kernel.

This threat isn't just theoretical. Real-world exploits, such as CVE-2024-44068, have utilized system services (like `cameraserver`) as intermediate steps to reach deeper kernel vulnerabilities [3]. Finding a way to secure these proprietary binaries is absolutely critical.

## 4.2 Exploiting RPC Design Principles

So, how can a fuzzer figure out the exact layout of a complex, nested Binder `Parcel` if it can't read the source code? NASS achieves this by taking advantage of the universal RPC Design Principles introduced in Chapter 2: Abstraction (Ab), Single Entry Point (Si), and Standard Deserialization (St).

Even if a vendor's code for processing camera sensor data is highly proprietary and obfuscated, the mechanism for receiving that data over Binder is standardized. To function within the Android OS, a proprietary binary *must* link against standard Android libraries (such as `libbinder.so`). When an IPC request hits the single entry point (`onTransact`), the auto-generated server stub *must* validate the payload by calling standard deserialization routines (e.g., `Parcel::readInt32()`) before it can even touch the proprietary logic.

The core insight behind NASS is that the server stub is actively enforcing the interface definition just by executing its code. By dynamically observing the sequence of standard deserialization calls the binary makes, a fuzzer can empirically figure out the interface structure without ever seeing the source code.

### 4.2.1 Empirical Validation of RPC Principles

To make sure this approach would actually work in the real world, the researchers behind NASS conducted a massive study of 316 proprietary native services across five modern COTS devices. They manually audited the server stubs of these closed-source binaries to see if they really did adhere to the Ab, Si, and St principles.

The results were incredibly validating: 89% of the analyzed proprietary services strictly adhered to all three RPC design principles [3]. 

The small minority of services that deviated typically violated the Abstraction (Ab) rule—for instance, by lazily pasting application-specific permission checks directly within the auto-generated deserialization stub. A smaller subset violated the Standardization (St) rule by writing custom byte-parsing logic instead of utilizing standard `libbinder.so` routines. Nevertheless, the high rate of compliance across different hardware vendors proved that a generalized, dynamic extraction technique could effectively map almost the entire modern attack surface.

## 4.3 Deserialization-Guided Interface Extraction (DGIE)

NASS puts this observation strategy to work using an algorithm called **Deserialization-Guided Interface Extraction (DGIE)**. DGIE is an iterative, dynamic probing loop that essentially reverse-engineers the interface signature. It works by feeding the target service partially correct inputs and using Dynamic Binary Instrumentation (DBI) to see how it reacts.

### 4.3.1 The Probing State Machine and Refinement Heuristics

The DGIE process isn't just a single pass. It uses a sophisticated two-phase approach: a *Preliminary Fuzzing Phase* to discover exposed RPC functions and trigger their entry points, followed by a *Refinement Phase* where it replays seeds to log the exact sequence of deserializers.

Let's imagine we're trying to reverse-engineer a specific transaction (say, `CONFIGURE_SENSOR`) on a proprietary `vendor.camera.hal` service. The DGIE refinement process acts as an iterative state machine:

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

This loop just keeps iterating. NASS continually appends the required data types to the `Parcel` until the service stops calling deserialization routines and successfully jumps into the underlying business logic. 

To keep this process efficient, NASS uses strict time-based heuristics to figure out when an interface has been fully mapped. During the preliminary fuzzing phase, NASS tracks how fast it discovers new seeds, establishing a baseline during the first two minutes. It then compares this baseline against subsequent two-minute intervals. If the discovery rate drops to ten times lower than the initial baseline, NASS assumes the server stub's deserialization paths are likely exhausted and officially transitions to the refinement phase. Also, just to prevent getting stuck in infinite loops on really defensive interfaces, NASS enforces a hard limit, forcing the transition to refinement after 20 minutes of fuzzing per target [3].

## 4.4 Unrolling Complex Parcelables Dynamically

Where DGIE really shines is when it has to deal with complex, high-level objects. In Android, data structures transmitted over Binder are often implemented as `Parcelable` objects. A static, black-box fuzzer struggles here because it can't guess the internal class hierarchy or member variables of a proprietary, undocumented `CameraConfig` Parcelable.

NASS sidesteps this by realizing that at the binary execution level, object-oriented abstractions don't exist. When a compiled C++ stub deserializes a `CameraConfig` Parcelable, it doesn't magically pop an object into existence; it simply executes a linear sequence of standard `read` calls corresponding to the object's internal fields. 

Think about how the proprietary C++ implementation of `CameraConfig::readFromParcel()` might actually execute under the hood:

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

DGIE doesn't bother trying to reconstruct the `CameraConfig` class hierarchy or guess what the variable names (`resolution_x`, `hardware_id`) might be. It just monitors the dynamic execution flow using Frida and records the linear sequence of expected reads: `[Int32, Int32, String16]`.

By "unrolling" this complex object into a flat, sequential list of primitive deserializers, NASS turns what would be a structurally impossible guessing game into a simple observation task. This dynamic unrolling allows NASS to generate perfectly formatted `Parcel` payloads that seamlessly slip past the proprietary stub's validation checks. In their evaluation, the researchers showed this achieved an 88% accuracy rate in signature recovery—which vastly outperforms passive traffic-sniffing techniques like the ones used in previous tools (e.g., BinderCracker) [3].

With the interface dynamically mapped, the structural barrier is finally broken. But to find deep logic bugs, NASS still has to leverage this knowledge to drive an evolutionary fuzzing loop. Doing that requires an unprecedented level of control over multi-threaded execution environments, which is exactly what I'll cover in the next chapter.