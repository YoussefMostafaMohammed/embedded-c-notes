# **Embedded C Notes**

This is not another syntax reference. This is a **deep, hardware-rooted exploration** of why Embedded C works the way it does—created from intensive Q&A sessions where every answer was driven by the question: **"What happens if this didn't exist?"**

Most Embedded C courses teach you *how* to write code. We teach you *why* the compiler generates specific instructions, *why* memory is partitioned into exact sections, and *why* violating certain rules causes physical system failures. Every concept here is traced to **CPU logic, compiler behavior, and hardware constraints**.

## **Prerequisites**

- **Required:** Basic C syntax (variables, functions, loops)
- **Recommended:** Some exposure to microcontrollers (Arduino, STM32, etc.)
- **Helpful:** Understanding of binary/hexadecimal

**No prior assembly knowledge needed**—every instruction is explained in context.

## **How to Use This Repository**

### **For Beginners**
Read in numerical order. Each module builds on the last, starting from first principles (why C was chosen) through advanced topics (context switching, linker scripts).

### **For Experienced Developers**
Jump to any module. Each README is self-contained with its own deep-dive sections, diagrams, and real-world failure case studies.

### **For Interview Prep**
Focus on modules 3 (Keywords), 5 (Structures), 6 (Pointers), and 8 (Assembly). These cover the most frequently asked low-level questions.

---

## **Repository Structure**

### **[01 - Why Embedded C](./1%20-%20Why-Embedded-C/Why-Embedded-C.md)**
*Why C dominates embedded systems: hardware access, determinism, bit manipulation, ANSI standardization, and the safety-performance tradeoff.*

**Key takeaways:**
- How a single `STR` instruction writes directly to hardware
- Why deterministic execution is life-or-death for motor controllers
- How bit fields compile to atomic single-instruction operations

---

### **[02 - Memory Architecture](./2%20-%20Memory-Architecture/Memory-Architecture.md)**
*The complete memory map: Flash vs RAM, .text/.rodata/.data/.bss/Stack/Heap, and the "What if not" scenarios for each section.*

**Key takeaways:**
- Why .data requires dual storage (Flash copy + RAM live)
- How .bss saves 4KB of Flash per 1000 variables
- Why stack grows down (overflow detection) vs up (silent corruption)

---

### **[03 - C Keywords](./3%20-%20C-Keywords/C-Keywords.md)**
*`const`, `static`, `register`, `volatile`, `extern`: memory placement, optimization hints, and atomicity.*

**Key takeaways:**
- Why `const` locals can be "hacked" but globals can't
- How `static` gives variables permanent addresses (not SP offset)
- Why `volatile` prevents compiler optimization disasters
- The difference between declaration and definition

---

### **[04 - Preprocessor Macros](./4%20-%20Preprocessor-Macros/Preprocessor-Macros.md)**
*Macro expansion mechanics, stringification, concatenation, and why macros are evil yet necessary.*

**Key takeaways:**
- The semicolon trap and `do-while-zero` idiom
- Two-stage concatenation trick for bit patterns
- When to use `__VA_ARGS__`

---

### **[05 - Structures & Alignment](./5%20-%20Structures-Alignment/Structures-Alignment.md)**
*Struct padding, `#pragma pack()`, and why bit fields are compiler-dependent.*

**Key takeaways:**
- Why `char + int` struct is 8 bytes (not 5)
- HardFault on unaligned access (Cortex-M0)
- Why bit fields fail for hardware registers

---

### **[06 - Pointers](./6%20-%20Pointers/Pointers.md)**
*Function pointers, complex declarations, void*, and pointer arithmetic for hardware registers.*

**Key takeaways:**
- Reading right-left-right for complex types
- ISR vector tables as function pointer arrays
- Why `void*` is essential for drivers

---

### **[07 - Build Process](./7%20-%20Build-Process/Build-Process.md)**
*From source to hex: preprocessor, compiler, assembler, linker, ELF vs HEX.*

**Key takeaways:**
- Why .o files have UND symbols
- How linker resolves relocations
- What objcopy strips for production

---

### **[08 - Assembly & CPU Mechanics](./8%20-%20Assembly-CPU-Mechanics/Assembly-CPU-Mechanics.md)**
*Core ARM registers, load/store, branch, bit manipulation, and context switching.*

**Key takeaways:**
- LR (Link Register) stores return addresses
- BL instruction encoding (32-bit offset)
- PendSV context switch in 12 cycles

---

### **[09 - Stack & Heap](./9%20-%20Stack-Heap/Stack-Heap.md)**
*Stack growth, PSP/MSP dual stack, heap fragmentation, malloc corruption.*

**Key takeaways:**
- Why two stacks enable OS safety
- Fragmentation geometry: 1000 bytes free but no 50-byte block
- Why mallocted-after-init is banned in safety-critical systems

---

### **[10 - Practical Patterns](./10%20-%20Practical-Patterns/Practical-Patterns.md)**
*Lookup tables, bootloaders, memory overflow detection, PID controller failure modes.*

**Key takeaways:**
- Sine lookup is 150x faster than `sinf()`
- VTOR register for bootloader vector remapping
- Magic number `0xDEADBEEF` for stack overflow detection

---

## **License**

MIT License - use freely, credit appreciated.