# Chapter 3: Kernel Fuzzing: The DIFUZE Approach

Early efforts to apply automated fuzzing to Android's structured interfaces focused on the open-source parts, specifically the Linux kernel. Introduced in 2017, **DIFUZE** (Interface Aware Fuzzing for Kernel Drivers) [1] established a way to recover `ioctl` interfaces directly from source code. 

## 3.1 The `ioctl` Problem

The `ioctl` interface is a huge roadblock for traditional fuzzers. It relies on arbitrary command IDs and untyped data pointers. A handler uses the command ID (`cmd`) to select a subroutine. These subroutines then cast the untyped data pointer (`arg`) to a specific structure and copy data into kernel memory.

A custom audio driver shows the difficulty. The command ID might be `AUDIO_SET_CONFIG`, and the driver expects a pointer to an `audio_config` structure. This structure could contain primitive integers—like sample rate—alongside a nested memory pointer for an equalizer preset array. 

If a fuzzer sends a random integer for `cmd`, the handler just rejects it. Even if you guess the right command, sending a pointer to random memory causes the driver to process garbage. If the driver then reads the equalizer preset by dereferencing that nested pointer—which is currently just random bytes—it attempts an invalid memory access and crashes the kernel. An effective fuzzer needs to know valid `cmd` values and the exact layout of those structures.

## 3.2 How DIFUZE Works

DIFUZE breaks its workflow into three stages. First, it **recovers the interface** through static analysis. **Structure generation** follows, using heuristics to fill fields. Finally, **execution** happens on the actual device.

## 3.3 Analyzing the Kernel

The heaviest lifting happens here. It all starts with getting the kernel into a format that a program can actually analyze.

### 3.3.1 Compiling to Bitcode

Android kernels are usually compiled with GCC, but DIFUZE needs the LLVM framework. A custom utility intercepts GCC commands during the build and translates them into LLVM commands, producing a consolidated bitcode file for each driver. These individual bitcode files are then merged and linked to enable whole-program analysis before the handler-finding step begins. Bitcode is a platform-agnostic representation that allows for more consistent analysis. 

Also, it is kind of ironic that the conversion breaks most often on the vendor kernels DIFUZE was meant to analyze. That is exactly where the source code is needed most, but the build customizations often make the conversion impossible.

### 3.3.2 Finding the Handlers

The analysis first has to find the `ioctl` entry points. Linux drivers register handlers by filling fields in standard structures. For example, they might assign a function pointer to the `unlocked_ioctl` field inside a `file_operations` struct. 

The harder part is scanning the bitcode for instructions that store function pointers into these fields. Once a handler is found, the analysis traces the code back to find where the device was registered. By inspecting those arguments, it extracts the path for the device node (like `/dev/nve`), which the fuzzer has to `open()` later.

### 3.3.3 Recovering Commands

Once a handler is pinpointed, DIFUZE has to figure out which integers the `cmd` argument accepts. It traces execution paths and collects equality checks. A `switch(cmd)` block is a very clear constraint. A Range Analysis algorithm then resolves these into the specific numbers needed to unlock the driver.

### 3.3.4 Identifying Argument Types

Linking command IDs to their data structures is probably the most complex step in the analysis pipeline. DIFUZE traces every path from the handler that ends in a `copy_from_user` call. It ignores calls that don't handle the `arg` pointer. For the valid calls, it identifies the type of the destination variable. If the kernel copies data into a `struct audio_config`, that's what the fuzzer should generate.

Nested functions and pointer casts can obscure these types. DIFUZE propagates type information across boundaries to avoid losing track. Finally, it links command IDs to the structures being copied and outputs the blueprints as XML.

## 3.4 Generating Structures

With the interface mapped, DIFUZE generates instances of the recovered structures. It doesn't just use random noise. Integers are often powers of two or right next to them, like 128 or 255. Nested structures are generated independently and packaged for later.

## 3.5 The Pointer Fixup

Executing `ioctl` calls on a real device requires instantiating structures with valid pointers. You can't just provide a random number as a memory address. You need a virtual address pointing to memory the fuzzer actually controls. 

DIFUZE handles this with a fixup step. It allocates anonymous memory for nested child structures, populates it with fuzzed data, and then injects that real virtual address into the parent structure's pointer field. It's a simple trick, but it's very effective—it keeps the driver from crashing on a trivial access violation.

## 3.6 Real-World Cases

The effectiveness of this method is shown by the bugs it actually found.

**The `qseecom` Bug (CVE-2017-0612):**
The exploit chain on the Google Pixel is interesting:
1. The fuzzer sends a huge `in_buf_size`.
2. `PAGE_ALIGN` overflows, making the value zero.
3. The driver allocates a zero-byte buffer and keeps going.
4. `copy_from_user` is called with that zero-size buffer. 

This bug is unreachable without a valid embedded pointer. That is the whole point of the fixup step.

**The `nve` Logic Flaw:**
On the Huawei Honor 8, DIFUZE found a logic flaw in the `nve` driver. The interface allowed userspace to change bootloader variables without any permission checks. An unprivileged app could just overwrite the device serial number. 

The `nve` flaw is actually more troubling than a memory corruption. There's no bug to patch; the interface works exactly as designed. The attack surface here isn't a mistake—it's an architectural decision about what to expose to the user. That is a much harder problem to solve.
