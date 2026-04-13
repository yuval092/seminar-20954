# Chapter 3: The Origins of Interface-Aware Fuzzing (DIFUZE & FANS)

The effort to apply automated, generation-based fuzzing to Android's structured interfaces initially focused on the platform's open-source components. Before researchers developed dynamic binary analysis techniques to handle proprietary vendor code, they relied on the Android Open Source Project (AOSP) and the Linux kernel source tree. This chapter examines the "Static Analysis Era" of interface-aware fuzzing, focusing on two pivotal systems: DIFUZE, which targeted the kernel, and FANS, which addressed the userspace Binder layer.

Analyzing these systems establishes the core concepts of interface recovery and highlights the limitations of relying solely on static analysis within the fragmented Android ecosystem.

## 3.1 The Static Analysis Era

In the mid-to-late 2010s, Android's attack surface was heavily concentrated in the Linux kernel. A malicious application within the userspace sandbox would often target a vulnerable kernel device driver to achieve privilege escalation. Because the Android kernel's source code (including major vendor modifications) must be available under the GPL license, researchers had access to the C/C++ files defining these interfaces.

Static analysis allows security tools to deduce a program's structure without executing it. By processing the source code through compiler infrastructures like LLVM or Clang, researchers could construct precise models of an interface's expectations.

## 3.2 DIFUZE: Overcoming the ioctl Barrier via Bitcode Analysis

Published in 2017, **DIFUZE** (Interface Aware Fuzzing for Kernel Drivers) [1] was the first system to automatically recover complex `ioctl` interfaces directly from kernel source code.

The `ioctl` barrier presents challenges due to arbitrary command IDs and nested C-structures containing memory pointers. DIFUZE addressed this by translating the Android kernel from standard GCC compilation commands into an LLVM bitcode representation. Custom LLVM analysis passes then mapped the kernel's attack surface using a specific pipeline:

1.  **Handler and Device Identification:** DIFUZE scanned the bitcode for assignments to known function pointer fields (e.g., `unlocked_ioctl` in a `file_operations` struct) to locate `ioctl` handler functions. It then traced the registration flow backward to determine the userspace device node string (e.g., `/dev/video0`).
2.  **Command Value Recovery:** It employed an inter-procedural Range Analysis algorithm to collect equality constraints on the `cmd` argument within the handler's execution paths, recovering the exact numerical command IDs supported by the driver.
3.  **Type Propagation:** To identify the required C-structure, DIFUZE traced the flow of the untyped data pointer (the variadic third argument of `ioctl`) until it reached a `copy_from_user()` call. The destination type of this copy operation revealed the expected C-structure.

### 3.2.1 The Pointer Fixup Mechanism

DIFUZE's most significant conceptual advancement was its handling of embedded memory pointers. Returning to the **Android Camera Subsystem** example, a fuzzer cannot supply a random 64-bit integer for a `calibration_ptr` field; it must provide a valid userspace memory address pointing to the correct nested structure.

DIFUZE resolved this dynamically through a technique called **Pointer Fixup**. The following pseudo-code illustrates this process within the on-device execution client:

```c
// 1. Independent Generation of Data
struct lens_calibration_data fuzz_child;
fuzz_child.focal_length = 50; // Fuzzed value
strcpy(fuzz_child.manufacturer_string, "A\x00\xff..."); // Fuzzed string

struct camera_sensor_config fuzz_parent;
fuzz_parent.sensor_id = 1;
fuzz_parent.resolution_width = 1920;

// 2. Runtime Allocation in Userspace
void* child_memory = mmap(NULL, sizeof(struct lens_calibration_data), 
                          PROT_READ|PROT_WRITE, MAP_ANONYMOUS|MAP_PRIVATE, -1, 0);
                          
// 3. Populate Child Buffer
memcpy(child_memory, &fuzz_child, sizeof(struct lens_calibration_data));

// 4. Pointer Fixup: Inject valid virtual address into parent struct
fuzz_parent.calibration_ptr = (struct lens_calibration_data*) child_memory;

// 5. Execution
int fd = open("/dev/video0", O_RDWR);
ioctl(fd, VIDIOC_S_SENSOR_CONFIG, &fuzz_parent);
```

By ensuring the kernel driver follows a valid pointer into the fuzzer's controlled memory space, DIFUZE prevents trivial kernel panics and enables exploration of deeper driver logic.

### 3.2.2 Real-World Efficacy and Case Studies

The necessity of the Pointer Fixup mechanism is demonstrated by the vulnerabilities DIFUZE discovered, such as the `qseecom` flaw (CVE-2017-0612) on the Google Pixel. 

The `qseecom` driver contained an `ioctl` handler that copied user data into a kernel struct with an embedded buffer pointer. A flawed size calculation (using `PAGE_ALIGN`) could cause the allocation of an undersized kernel buffer. However, this vulnerable memory copy was only triggered if the provided embedded pointer was valid. Without DIFUZE's precise structure instantiation and pointer fixup, a fuzzer would fail the pointer validation check, leaving the bug unreachable.

DIFUZE also uncovered severe logic flaws. On the Huawei Honor 8, the `nve` driver exposed an `ioctl` interface allowing userspace to read and write privileged bootloader variables, such as the device's serial number (`ro.serialno`). A standard app could overwrite this serial number because the `ioctl` handler lacked permission checks—a significant design flaw discovered by systematically exercising the recovered interface.

### 3.2.3 Critiques of the DIFUZE Methodology

Despite its conceptual strength, DIFUZE's reliance on converting GCC to LLVM bitcode proved fragile in practice. Commercial Android kernels frequently contain extensive vendor modifications and complex inline-assembly macros. Compiling a heavily customized vendor kernel under a different compiler infrastructure often required manual patching, limiting the tool's intended automation.

## 3.3 FANS: Navigating Binder Semantics via AST Extraction

As the Android ecosystem evolved, attackers shifted their focus toward native system services communicating via Binder IPC. Published in 2020, **FANS** (Fuzzing Android Native System Services) [2] recognized that `ioctl` fuzzing techniques could not be directly ported to Binder.

The barrier with Binder is semantic as well as structural. A Binder `Parcel` is a serialized byte stream deserialized sequentially. The presence of certain variables in the `Parcel` often depends on the runtime evaluation of previously deserialized variables.

To capture these semantics, FANS analyzed the **Abstract Syntax Tree (AST)** using Clang, moving away from LLVM bitcode. The AST preserves high-level semantics lost during compilation, notably exact variable names and complex type aliases. These details are crucial for generating contextually valid inputs. FANS categorized variables into patterns (sequential, conditional, loop) to understand `Parcel` structures based on runtime conditions.

### 3.3.1 Interface Dependency: The Multi-Level Interface Problem

FANS revealed that a significant portion of the Binder attack surface is hidden from naive fuzzers. Approximately 37% of native interfaces are not registered directly with the `ServiceManager`. These are "multi-level" interfaces retrieved dynamically.

FANS used AST analysis to map these **Generation and Use Dependencies**. When an upper-level interface generates a nested interface, it calls `writeStrongBinder` to serialize the new interface into the reply `Parcel`. Conversely, when an interface is expected as input, the service calls `readStrongBinder`. By linking these calls, FANS mapped paths to deeply nested targets, significantly expanding its fuzzing scope.

### 3.3.2 Inter-Transaction Dependency Inference

FANS addressed the highly stateful nature of Android system services. Critical vulnerabilities typically require a specific sequence of API calls rather than a single transaction. FANS formalized the extraction of these state machines through **Dependency Inference**.

Consider the Camera Subsystem example. A client app might send an `openConnection()` transaction, receiving a unique integer `session_id`. A subsequent `captureImage()` transaction requires this exact `session_id` in its request `Parcel`. A naive fuzzer guessing a random integer for `captureImage()` will fail the authorization check.

FANS automated the discovery of these relationships using a Name and Type Matching algorithm. The conceptual logic for generating this dependency graph is as follows:

```python
# FANS Algorithm 1: Inference of Inter-Transaction Dependency
DependencyGraph G = empty_graph()
InputVars I = extract_all_read_from_parcel_vars(AST)
OutputVars O = extract_all_write_to_parcel_vars(AST)

for iVar in I:
    for oVar in O:
        # Prevent self-dependency within the same transaction
        if iVar.transaction_id != oVar.transaction_id:
            
            # Rule 1: Types must strictly match
            if iVar.type == oVar.type:
                
                # Rule 2: Complex objects always depend, primitives require name similarity
                if is_complex_type(iVar.type):
                    G.add_edge(iVar, oVar)
                elif calculate_string_similarity(iVar.name, oVar.name) > THRESHOLD:
                    # Example: iVar="target_session_id" matches oVar="active_session_id"
                    G.add_edge(iVar, oVar)
                    
return G
```

This dependency graph allowed the fuzzer to execute multi-stage sequences, capturing outputs from initial transactions and supplying them to subsequent payloads. This navigation was essential for reaching vulnerable code within complex IPC flows.

### 3.3.3 Deep Vulnerability Discovery via State Navigation

The effectiveness of dependency inference is highlighted by the vulnerabilities FANS uncovered. For example, it discovered an unexpected stack buffer overflow within the Linux `ip6tables-restore` binary, reachable via Android's `netd` (network daemon) system service.

The vulnerable code path required an active Binder reference to a previously configured network interface. A standard fuzzer could not spontaneously instantiate a valid network configuration object. Because FANS' algorithm linked the output of an interface-creation transaction to the input of the vulnerable transaction, the fuzzer generated the multi-stage sequence needed to deliver the payload across three separate processes.

FANS also identified bugs resulting from inadequate server-side validation. In the `IDrm` interface, a `readVector` function allocated memory based on a `size` parameter from the `Parcel`. Because FANS extracted this parameter name, it generated valid semantic values (e.g., `-1`). The lack of a sanity check on this size led to a `new_capacity` overflow.

Furthermore, testing native C++ services with FANS yielded 138 unique Java exceptions, demonstrating how structural testing of the native layer indirectly stresses the Java application framework due to cross-language dependencies.

### 3.3.4 Critiques of the FANS Methodology

Similar to DIFUZE, FANS' methodology was limited by the fragility of static analysis. AST parsing is susceptible to macro-obfuscated C++ code or complex pointer passing, which can obscure variable origins and lead to incomplete dependency graphs. Additionally, FANS operated as a purely black-box fuzzer. While it generated structured inputs, it lacked the grey-box coverage feedback necessary to evolve those inputs based on actual runtime behavior.

## 3.4 The "Open-Source Blind Spot" and the Limits of Static Analysis

DIFUZE and FANS demonstrated that interface-awareness is crucial for bypassing shallow validation checks. However, their reliance on static source-code analysis ultimately proved to be a critical limitation.

The Android ecosystem is fragmented. While Google maintains the open-source AOSP framework, hardware vendors tightly control the logic interacting with their physical hardware. Project Treble mandated a separation between the framework and the Vendor Hardware Abstraction Layer (HAL), pushing the most privileged code into closed-source, proprietary binaries.

Because FANS relied entirely on parsing C++ ASTs, it was blind to these proprietary HAL services. Static analysis tools cannot audit a substantial portion of the modern Android attack surface on commercial devices. 

To effectively secure the Android platform, researchers needed a methodology capable of mapping complex IPC interfaces without source code access. This "open-source blind spot" necessitated the development of the dynamic, binary-only approach introduced by NASS.