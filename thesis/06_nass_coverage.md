# Chapter 6: NASS and Coverage in Multi-Threaded Daemons

Dynamic interface extraction overcomes the initial structural barrier, but constructing valid `Parcels` is not enough to find logical vulnerabilities. Evolutionary guidance is required to navigate complex stateful logic. The fuzzer must see which code paths its inputs execute so it can reward mutations that reach deeper states. 

This feedback loop depends on accurate code coverage—which is straightforward in principle, but very expensive in practice when concurrent daemons are involved.

## 6.1 Daemons are Not Clean Fuzzing Targets

Traditional grey-box fuzzers are built around the idea that one input goes to one target. Android daemons break that completely — they are always doing multiple things at once. A daemon such as `cameraserver` or a vendor's HAL is a multi-threaded process constantly running in the background. At any moment, it might be handling hardware interrupts or internal maintenance.

Naive coverage tracking on these processes results in too much noise. The fuzzer cannot distinguish which blocks were executed because of its payload and which were triggered by background traffic. This pollution breaks the link between mutations and paths, making the coverage guidance ineffective.

## 6.2 Thread-Localized Tracing

NASS isolates its footprint from background noise by tying instrumentation to the Single Entry Point (Si) principle. When an IPC request triggers `onTransact`, NASS intercepts the execution and retrieves the PID of the client. If the calling PID matches the NASS client, the hook activates Frida Stalker exclusively on the current thread. Unrelated transactions just proceed normally.

When `onTransact` returns, NASS deactivates Stalker and ships the coverage bitmap back to the fuzzer. This guarantees that the trace is deterministic and represents only the logic triggered by the fuzzer's input. 

The reported 30–400 executions/second range is worth pausing on. NASS reports this as a performance range, but it's really an admission that service complexity varies so much that a single number is meaningless. It is worth pausing on this—it makes the results harder to compare. We can't tell if the slowest services are actually the most complex from the paper.

## 6.3 The Fuzzing Feedback Loop

With interface definitions and isolated coverage, NASS implements an interface-aware mutator. Rather than mutating a `Parcel` as a raw array of bytes, it targets the specific types found during DGIE. NASS applies mutations tailored to the 23 supported Binder argument types:

1.  **Integers and floats** are mutated with boundary values—`MAX_INT`, zero, and negative values.
2.  **Strings** get length expansions and special character insertions.
3.  **Vectors** undergo element-count mutations while preserving the outer `Parcel` layout. Elements can be added or removed dynamically without breaking the structure.
4.  **Special objects** require more work. For arguments that need file descriptors, NASS writes fuzzed data to a temporary file and serializes the descriptor.

The fuzzer dispatches a mutated `Parcel`, collects the coverage, and analyzes the results. If a mutation bypasses a check to execute a new block, it is added to the corpus.

## 6.4 Real-World Results

The proof of this approach is in the empirical findings. NASS discovered 12 memory-corruption bugs across 316 services on five devices, leading to five CVEs [3]. 

**Heap Buffer Overflow in Samsung S23 Radio HAL:**
Using DGIE, NASS unrolled a nested structure requiring seven distinct deserializers. One read was an integer representing payload length. Because the interface was mapped, the fuzzer reached the vendor's logic. The service included a safety check, but it was signed:

```cpp
int32_t payload_length = parcel->readInt32(); 

if (payload_length > MAX_MESSAGE_SIZE) {
    return ERROR_BUFFER_TOO_LARGE;
}

memcpy(internal_heap_buffer, data_ptr, payload_length);
```

The evolutionary algorithm found that a negative `payload_length` bypassed the `if` block. Since `memcpy` expects an unsigned `size_t`, the negative integer resulted in a massive, out-of-bounds copy.

**Use-After-Free in Pixel 9 (CVE-2024-47040):**
NASS also found a critical UAF in the `ISap` service. The service maintained a linked list of active requests. When a request finished, the memory was freed. Because NASS was generating valid `Parcels`, it could explore token-handling edge cases, eventually sending two requests with the same token. The service failed to check for duplicates, freeing an active request while it was still being processed by another thread.

In practice, this means a crash found at hour three might not be reproducible at hour four because the hardware state changed. But systematically reverse-engineering the interface and guiding execution with isolated feedback finally gives us a way to dismantle these barriers.
