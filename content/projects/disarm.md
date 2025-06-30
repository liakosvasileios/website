+++
title = "DISARM"
date = 2025-05-20
github = "https://github.com/liakosvasileios/DISARM"
tags = ["malware", "operating systems", "windows", "antivirus evasion"]
description = "DISARM: Detection Inhibition via Self-Adaptive Runtime Mutation"
+++

## Overview

[DISARM](https://github.com/liakosvasileios/DISARM) (Detection Inhibition via Self-Adaptive Runtime Mutation) is a cutting-edge metamorphic engine designed specifically for 64-bit x86_64 binaries. Unlike traditional polymorphic techniques that rely on encryption or simple obfuscation, DISARM dynamically rewrites machine code at runtime while preserving the program's original functionality.

This project fills a major gap in existing tools, which mostly target outdated 32-bit architectures, by supporting modern 64-bit instruction sets and advanced obfuscation techniques.

---

## Key Features

- **Custom x86_64 Decoder:** Converts variable-length instructions into a normalized intermediate representation (IR) for flexible transformation.
- **Powerful Mutation Pipeline:** Uses Mixed Boolean Arithmetic (MBA), instruction substitution, and control flow rewriting to change code structure while maintaining behavior.
- **Safe Reassembler:** Generates valid, fully functional x86_64 machine code after transformations.
- **JIT Dispatch Obfuscation:** Implements a Just-In-Time obfuscation technique inspired by WebAssembly's virtual calls, using SIMD registers (XMM/YMM) to hide function call targets dynamically.
- **Round-Trip Testing:** Ensures semantic equivalence and binary integrity by simulating and verifying transformed code behavior.

---

## Why DISARM?

As attackers evolve and detection techniques improve, simple obfuscation and static encryption are no longer enough. DISARM’s approach focuses on:

- Continuous runtime mutation to evade signature-based detection.
- Obscuring control flow to resist static and symbolic analysis.
- Maintaining full compatibility with existing binaries without requiring source code or recompilation.
- Leveraging modern CPU features like AVX instructions and SIMD registers for innovative protection methods.

---

## Technical Highlights

- Supports complex, reversible structural mutations while preserving program semantics.
- Uses SIMD registers for encrypted virtual call tables, complicating control flow analysis.
- Modular design allows easy extension with new transformation passes or integration with runtime packers and virtualization.
- Operates entirely at the binary level, making it applicable to legacy and modern software alike.

---

## Applications

- Advanced software protection against reverse engineering and malware analysis.
- Offensive security research into evasion techniques.
- Platform for developing new metamorphic obfuscation and JIT compilation methods.

---

For more technical details or to contribute, feel free to reach out or check the project repository.
[*DISARM*](https://github.com/liakosvasileios/DISARM)
