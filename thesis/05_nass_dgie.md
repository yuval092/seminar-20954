# Chapter 5: Proprietary Service Fuzzing: NASS and Dynamic Extraction

The introduction of Project Treble in Android 8 fundamentally changed the security research landscape. By isolating hardware-specific logic inside the Vendor Hardware Abstraction Layer (HAL) and forcing everything to communicate via Binder IPC, Google made the OS much more modular. However, this shift also hid a massive portion of the attack surface. As we saw in Chapters 3 and 4, static analysis tools like DIFUZE and FANS—which rely entirely on having open-source C++ code to read—became practically useless against these new, compartmentalized, proprietary binaries.

To audit this hidden layer, researchers introduced **NASS** (Native Android System Services) in 2025 [3]. NASS represents a necessary evolution in methodology: it abandons static source-code analysis completely in favor of dynamic binary instrumentation. This chapter explores why the proprietary attack surface is so important and breaks down NASS's core innovation: Deserialization-Guided Interface Extraction (DGIE).

## 5.1 The Proprietary Vendor HAL: The Modern Attack Surface

The impact of the "open-source blind spot" is huge. When researchers analyzed modern Commercial Off-The-Shelf (COTS) devices—like the Google Pixel 9 and the Samsung Galaxy S23—they found that over 60% of the running native system services were completely proprietary [3]. The vast majority of these closed-source services live right inside the HAL.

HAL services are incredibly attractive targets if you're trying to escalate privileges. By design, they hold the elevated SELinux permissions needed to talk directly to hardware-specific, vendor-modified kernel drivers. If an attacker who is stuck inside a restricted app sandbox can trigger a memory-corruption bug by sending a Binder RPC call to a proprietary HAL service, they can take over that service. This effectively bypasses the app sandbox and gives the attacker a highly privileged foothold right next to the kernel.

This isn't just theory. Real-world exploits, like CVE-2024-44068, have actually used system services (like `cameraserver`) as stepping stones to reach deeper kernel vulnerabilities [3]. Figuring out how to secure these proprietary binaries is absolutely critical.

## 5.2 Exploiting RPC Design Principles

So, how do you figure out the exact layout of a complex, nested Binder `Parcel` if you aren't allowed to read the source code? NASS pulls this off by exploiting the universal RPC Design Principles we introduced back in Chapter 2: Abstraction (Ab), Single Entry Point (Si), and Standard Deserialization (St).

Even if a vendor's code for processing hardware data is highly proprietary and heavily obfuscated, the way it *receives* that data over Binder is standardized. To work on Android, proprietary binaries have to link against standard Android libraries (like `libbinder.so`). When an IPC request hits the single entry point (`onTransact`), the auto-generated server stub is forced to validate the payload using standard deserialization routines (like `Parcel::readInt32()`) before it can even touch the secret proprietary logic.

The core insight behind NASS is that the server stub is actively enforcing the interface definition just by executing its code. By dynamically watching the sequence of standard deserialization calls the binary makes, a fuzzer can empirically reverse-engineer the interface structure without ever seeing the source code.

### 5.2.1 Empirical Validation of RPC Principles

To make sure this approach would actually work in the real world, the researchers behind NASS conducted a massive study of 316 proprietary native services across five modern phones. They wanted to see if the server stubs of these closed-source binaries actually followed the Ab, Si, and St principles in practice.

The results were overwhelmingly positive: 89% of the analyzed proprietary services strictly adhered to all three RPC design principles [3]. 

The few services that broke the rules usually violated the Abstraction (Ab) principle—for instance, by lazily pasting application-specific permission checks right into the auto-generated deserialization stub. A smaller handful violated the Standardization (St) principle by writing custom byte-parsing logic instead of using the standard `libbinder.so` routines. Still, the high rate of compliance across different hardware vendors proved that a dynamic extraction technique could effectively map almost the entire modern attack surface.

## 5.3 Deserialization-Guided Interface Extraction (DGIE)

NASS puts this observation strategy to work using an algorithm called **Deserialization-Guided Interface Extraction (DGIE)**. DGIE is an iterative, dynamic probing loop that reverse-engineers the interface signature through trial and error. It works by feeding the target service partially correct inputs and using Dynamic Binary Instrumentation (DBI) to see how the service reacts.

### 5.3.1 Phase 1: Preliminary Fuzzing (Discovery)
The first phase of DGIE is all about discovery. NASS fires off IPC requests with various transaction IDs to see which ones trigger a response from the `onTransact` dispatcher. It monitors the code coverage, and whenever it sees new basic blocks being executed, it knows it has found a valid, exposed RPC function.

### 5.3.2 Phase 2: Iterative Refinement
Once it knows a transaction exists, NASS moves to the refinement phase. This acts like a state machine. NASS sends a request with the valid transaction ID but a completely empty payload. Using DBI, NASS intercepts the target service's execution. 

If the server stub attempts to invoke a deserializer (e.g., `readInt32()`), the operation will fail because the payload is empty, and the transaction will abort. NASS records this required data type, generates a new `Parcel` containing a valid `Int32`, and sends the request again.

This loop just keeps iterating. NASS sequentially appends the identified data types to the `Parcel` until the service stops calling deserialization routines and successfully jumps into the underlying business logic. Once that happens, NASS knows the interface signature has been fully extracted.

To keep this process from running forever, NASS uses strict time limits. During the discovery phase, it tracks how fast it finds new paths. Once the discovery rate drops to ten times lower than the initial baseline, NASS assumes it has mapped out the deserialization paths and moves fully into refinement. As a hard failsafe, it will force the transition to refinement after 20 minutes of fuzzing per target [3].

## 5.4 Unrolling Complex Parcelables Dynamically

Where DGIE really proves its worth is in handling complex, high-level objects. In Android, data transmitted over Binder is often bundled into `Parcelable` objects. A static, black-box fuzzer struggles here because it can't possibly guess the internal class hierarchy or the member variables of an undocumented, proprietary `Parcelable`.

NASS sidesteps this problem by recognizing that at the binary execution level, object-oriented abstractions don't actually exist. When a compiled C++ stub deserializes a `Parcelable`, it doesn't magically spawn an object; it just executes a linear sequence of standard `read` operations corresponding to the object's internal fields. 

For example, imagine a proprietary service expects a `WiFiConfiguration` object. The fuzzer doesn't need to know the object's name, or that it contains a string for the SSID, an integer for the security protocol, and a boolean for the hidden-network flag. DGIE simply watches the dynamic execution flow and records the linear sequence of expected reads: `[String16, Int32, Bool]`.

By dynamically "unrolling" complex objects into a flat, sequential list of primitive reads, NASS turns structural inference into a simple observation task. This dynamic unrolling lets NASS generate correctly formatted `Parcel` payloads that slip right past the proprietary stub's validation checks. In testing, this technique achieved an 88% accuracy rate in signature recovery, vastly outperforming older, passive traffic-capture methods [3].

With the interface dynamically mapped, the structural barrier is broken. But to actually find deep logic bugs, NASS still needs to use this structural knowledge to drive an evolutionary fuzzing loop, which requires unprecedented control over multi-threaded execution environments.
