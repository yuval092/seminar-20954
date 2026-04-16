# Chapter 6: NASS and Grey-Box Coverage in Multi-Threaded Daemons

Dynamic interface extraction overcomes the initial structural barrier, but constructing valid `Parcels` is not enough to find logical vulnerabilities. Evolutionary guidance is required to navigate complex stateful logic; the fuzzer must see which code paths its inputs execute to reward mutations that reach deeper states. This feedback loop depends on accurate code coverage—straightforward in principle, but expensive in practice when concurrent daemons are involved.

## 6.1 The Concurrency Problem in Android System Services

Traditional grey-box fuzzers assume that a single target process corresponds to a single fuzzer input. Android services break this assumption. A daemon such as `cameraserver` or a vendor's HAL is a concurrent, multi-threaded process constantly running in the background. At any moment, it might handle framework requests, background hardware interrupts, or internal maintenance.

Naive coverage tracking on a HAL service process results in noise. The fuzzer cannot distinguish which basic blocks were executed due to its payload and which were triggered by unrelated background traffic. This pollution breaks the link between mutations and code paths, rendering coverage guidance ineffective.

## 6.2 Thread-Localized Tracing via PID Filtering

NASS isolates its execution footprint from background noise by tying instrumentation to the Single Entry Point (Si) principle. When an IPC request triggers `onTransact`, a NASS instrumentation hook intercepts execution and retrieves the Process ID (PID) of the client. If the calling PID matches the NASS fuzzer client, the hook activates Frida Stalker exclusively on the current thread. Unrelated transactions proceed without instrumentation.

When `onTransact` returns, NASS deactivates Stalker and ships the coverage bitmap back to the fuzzer. This guarantees that the resulting trace is deterministic and represents only the logic triggered by the fuzzer's input. NASS reports a 30–400 executions/second range [3], which is wide enough to make benchmarking nearly meaningless. A service that runs at 400 ex/sec and one that runs at 30 ex/sec are in completely different fuzzing regimes; one discovers complex state spaces in hours, the other in days. NASS's aggregate results tend to paper over this variance.

## 6.3 The Evolutionary Fuzzing Feedback Loop

With interface definitions and isolated coverage, NASS implements an interface-aware mutator built on LibFuzzer. Rather than mutating a `Parcel` as a raw byte array, the mutator targets the specific data types identified during DGIE. NASS applies semantics-aware mutations tailored to the 23 supported Binder argument types:

1.  **Primitive integers and floats** are mutated using boundary-value heuristics—`MAX_INT`, zero, and negative values—to stress standard size validations.
2.  **Strings** get length expansions, malformed UTF encodings, and special character insertions.
3.  **Vectors** undergo element-count mutations while preserving the outer `Parcel` layout—the critical constraint that prevents structural breakage—so that elements can be added or removed dynamically.
4.  **Special objects** require more ceremony. For arguments that need file descriptors, NASS writes fuzzed data to a temporary file and serializes the descriptor, or serializes a handle to a fuzzer-controlled service to simulate complex inter-process callback loops.

The fuzzer dispatches a mutated `Parcel`, collects the coverage bitmap, and analyzes the results. If a mutation bypasses a validation check to execute a new basic block, it is added to the corpus.

## 6.4 Real-World Results

The proof of this combined approach is in its empirical findings. NASS discovered 12 memory-corruption vulnerabilities across 316 proprietary services on five devices, leading to five CVEs [3]. 

**Heap Buffer Overflow in Samsung S23 Radio HAL:**
Using DGIE, NASS unrolled a nested message structure requiring seven distinct deserializers. One read was an integer representing payload length. Because the interface was mapped, the fuzzer reached the vendor's dispatch logic. The service included a safety check for the payload length, but it was signed:

```cpp
int32_t payload_length = parcel->readInt32(); 

if (payload_length > MAX_MESSAGE_SIZE) {
    return ERROR_BUFFER_TOO_LARGE;
}

memcpy(internal_heap_buffer, data_ptr, payload_length);
```

The evolutionary algorithm found that a negative `payload_length` bypassed the `if` block. Since `memcpy` expects an unsigned `size_t`, the negative integer resulted in a massive, out-of-bounds copy and a heap overflow.

**Use-After-Free in Pixel 9 (CVE-2024-47040):**
NASS uncovered a critical UAF in the `ISap` service. The service maintained a linked list of active requests identified by an RPC-supplied token. When a request finished, the service freed the memory. Because NASS was generating structurally valid `Parcels`, it could explore token-handling edge cases, eventually dispatching two requests with the same token. The service failed to check for duplicates, freeing an active request while it was still being processed by another thread.

Systematically reverse-engineering interfaces and guiding execution with isolated feedback provides a way to dismantle the barriers protecting modern mobile devices.
