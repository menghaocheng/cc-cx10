# Translator & Core Implementation Guide

## Overview
The **Translator** (`tango_translator_reimpl`) is the core component responsible for executing ARM32 code on an ARM64 host. It uses a combination of interpretation (fallback) and JIT compilation.

## Key Subsystems

### 1. System Call Emulation (`syscall_handler.cpp`)
- **Role**: Translates ARM32 Linux syscalls to ARM64 syscalls.
- **Critical Files**:
    - `syscall_handler.cpp`: Implementation of `handle(sc_no, ...)`.
    - `syscalls.h`: Struct definitions matching ARM32 kernel ABI.
- **Workflow for adding Syscall**:
    1. Identify missing syscall number (from logcat).
    2. Define required structs in `syscalls.h` (e.g., `kernel_stat32`). Ensure alignment matches ARM32.
    3. Add case in `SyscallHandler::handle`.
    4. Implement translation logic (marshal arguments, call host syscall, unmarshal result).

### 2. Instruction Decoding (`arm32_decoder.cpp`, `thumb_decoder.cpp`)
- **Role**: Parses raw bytes into internal `TranslationBlock` or intermediate representation.
- **Notes**:
    - Must handle both ARM (4-byte) and Thumb (2-byte/4-byte) modes.
    - Pay attention to `IT` blocks in Thumb mode.

### 3. Code Emission (`arm64_emitter.cpp`)
- **Role**: Generates native ARM64 instructions corresponding to the decoded ARM32 instructions.
- **Optimization**: Currently implements a basic 1:1 mapping where possible. Future work involves block optimization.

### 4. Memory Management
- **Guest Address Space**: Must simulate 32-bit address space behavior.
- **Signals**: `signal_handler.cpp` must catch host signals (SEGV, ILL) and inject them into the guest context or handle JIT trampolines.

## Common Issues
- **Structure Padding**: 32-bit structs often have different padding than 64-bit ones. Use `__attribute__((packed))` or manual padding carefully.
- **Pointers**: Guest pointers are 32-bit `uint32_t`. Host pointers are 64-bit.
- **Errno**: Verify if errno values need translation between ABIs (usually identical for Linux, but verify).
