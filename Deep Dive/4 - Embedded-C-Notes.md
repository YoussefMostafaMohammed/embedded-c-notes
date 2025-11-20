# 4 - Embedded-C-Notes

## **Table of Contents**

1. [Memory Analysis with objdump and Dumps](#1-memory-analysis-with-objdump-and-dumps)
   - 1.1. [Understanding `objdump -s` Output](#11-understanding-objdump--s-output)
   - 1.2. [Decoding Hexadecimal Memory Dumps](#12-decoding-hexadecimal-memory-dumps)
   - 1.3. [Real Example: Sine Lookup Table](#13-real-example-sine-lookup-table)

2. [Performance: Lookup Tables vs Runtime Calculation](#2-performance-lookup-tables-vs-runtime-calculation)
   - 2.1. [Runtime Calculation (sinf)](#21-runtime-calculation-sinf)
   - 2.2. [Lookup Table Access Assembly](#22-lookup-table-access-assembly)
   - 2.3. [Why 150x Faster Matters](#23-why-150x-faster-matters)

3. [Instruction Size and Manual Initialization Cost](#3-instruction-size-and-manual-initialization-cost)
   - 3.1. [The 12-Byte Reality](#31-the-12-byte-reality)
   - 3.2. [Startup Code memcpy Efficiency](#32-startup-code-memcpy-efficiency)
   - 3.3. [Where the 20 Bytes Come From](#33-where-the-20-bytes-come-from)

4. [Memory Sections Deep Dive](#4-memory-sections-deep-dive)
   - 4.1. [`.bss` Size Clarification](#41-bss-size-clarification)
   - 4.2. [The Economics: Why Save Flash at Cost of RAM](#42-the-economics-why-save-flash-at-cost-of-ram)
   - 4.3. [Why Zero? Characteristics of the Chosen Number](#43-why-zero-characteristics-of-the-chosen-number)

5. [Complete Memory Layout Diagram](#5-complete-memory-layout-diagram)

6. [Stack Mechanics Deep Dive](#6-stack-mechanics-deep-dive)
   - 6.1. [LR (Link Register) Deep Dive](#61-lr-link-register-deep-dive)
   - 6.2. [Function Call Mechanics](#62-function-call-mechanics)
   - 6.3. [What Happens If You Corrupt LR](#63-what-happens-if-you-corrupt-lr)

7. [Dual Stack Pointers (PSP) Context Switch](#7-dual-stack-pointers-psp-context-switch)
   - 7.1. [Creating a Task](#71-creating-a-task)
   - 7.2. [OS Schedules First Task](#72-os-schedules-first-task)
   - 7.3. [What Happens During Task Execution](#73-what-happens-during-task-execution)
   - 7.4. [When ISR Fires During Task](#74-when-isr-fires-during-task)

8. [Upward Stack Growth: The Silent Killer](#8-upward-stack-growth-the-silent-killer)

9. [Magic Numbers and Overflow Detection](#9-magic-numbers-and-overflow-detection)
   - 9.1. [Where the Magic is Set](#91-where-the-magic-is-set)
   - 9.2. [Why 0xDEADBEEF?](#92-why-0xdeadbee)

10. [Heap Corruption During ISR](#10-heap-corruption-during-isr)

11. [constexpr and Flash Storage](#11-constexpr-and-flash-storage)
    - 11.1. [The Address Rule](#111-the-address-rule)
    - 11.2. [Compiler Options](#112-compiler-options)
    - 11.3. [Solution: Use `const` and `static`](#113-solution-use-const-and-static)

12. [`extern` in Function Scope: The Truth](#12-extern-in-function-scope-the-truth)
    - 12.1. [Scoping Rules](#121-scoping-rules)
    - 12.2. [Why It's Allowed](#122-why-its-allowed)
    - 12.3. [Correct Usage](#123-correct-usage)

13. [Bootloader vs Application Vector Tables](#13-bootloader-vs-application-vector-tables)
    - 13.1. [Why Both Need Vectors](#131-why-both-need-vectors)
    - 13.2. [Bootloader's Job](#132-bootloaders-job)
    - 13.3. [Application Vector Table Layout](#133-application-vector-table-layout)
    - 13.4. [What Happens After VTOR Remap](#134-what-happens-after-vtor-remap)

14. [Symbol Table Decoding Complete Legend](#14-symbol-table-decoding-complete-legend)

15. [Executive Summary: The Why Matrix](#15-executive-summary-the-why-matrix)

---

## **1. Memory Analysis with objdump and Dumps**

### **1.1. Understanding `objdump -s` Output**

`objdump -s` extracts raw **hexadecimal bytes** from each memory section of your compiled binary.

**Your output:**
```
Contents of section .rodata:
 08001200 00001100 23003500 47005a00 6d007f00  ....#.G.Z.m...
 08001210 9100a400 b700ca00 dd00f000 03011701  ................
```

**How to decode:**
```
Address    Raw Hex Bytes         ASCII
-------    --------------        -----
08001200:  00 00 11 00 23 00 35 00 47 00 5a 00 6d 00 7f 00  ....#.G.Z.m...
08001210:  91 00 a4 00 b7 00 ca 00 dd 00 f0 00 03 01 17 01  ................
```

### **1.2. Decoding Hexadecimal Memory Dumps**

The hex bytes are displayed in **little-endian** format (ARM architecture). Each pair of hex digits represents one byte.

### **1.3. Real Example: Sine Lookup Table**

```c
const uint16_t sine[16] = {0, 17, 35, 52, 70, 87, 105, 122, 
                           140, 157, 175, 192, 210, 227, 245, 262};
```

**`objdump -s firmware.elf -j .rodata`** would show:
```
08001200  00001100 23003500 47005a00  ......G.Z.
```
**Decoded:**
- `0000` = 0 (little-endian)
- `1100` = 0x0011 = 17
- `2300` = 0x0023 = 35
- `3500` = 0x0035 = 53
- **Each entry is 2 bytes (uint16_t)**

---

## **2. Performance: Lookup Tables vs Runtime Calculation**

### **2.1. Runtime Calculation (sinf)**

```c
float angle = 45.0f * M_PI / 180.0f;
float result = sinf(angle);  // Library call
```

**Assembly for sinf():**
```assembly
BL sinf            ; Branch to floating-point library
; Inside sinf():
    VLDR S0, [R0]  ; Load angle
    VCVT.F32.F64 D0, S0  ; Convert to double
    ; ... Taylor series approximation (20+ instructions) ...
    VCVT.F64.F32 S0, D0  ; Convert back
    VSTR S0, [R1]      ; Store result
```
**Total: ~300 clock cycles** (floating point is slow on Cortex-M4 without FPU)

### **2.2. Lookup Table Access Assembly**

```c
uint16_t result = sine_lookup[45];  // Array index
```

**Assembly:**
```assembly
LDR R0, =sine_lookup  ; R0 = 0x08001200 (1 cycle)
LDRH R1, [R0, #90]    ; R1 = halfword at offset 90 (1 cycle)
```
**Total: 2 cycles**

### **2.3. Why 150x Faster Matters**

- **No math**: Just memory read
- **No conversion**: Fixed-point arithmetic
- **No function call**: Inlined code
- **No pipeline stall**: Simple load

**What if you calculated sin() in real-time?** Your motor control loop would run at **100Hz** instead of **20kHz** → drone crashes.

---

## **3. Instruction Size and Manual Initialization Cost**

### **3.1. The 12-Byte Reality**

```c
void manual_init(void) {
    config = 50;  // Variable initialization
}
```

**Generated assembly:**
```assembly
LDR R0, =config      ; Load address of config: 4 bytes
LDR R1, =50          ; Load immediate value 50: 4 bytes
STR R1, [R0]         ; Store to memory: 2 bytes
BX LR                ; Return: 2 bytes
```
**Total: 4 + 4 + 2 + 2 = 12 bytes** (wait, that's 12, not 8)

**Correction - Thumb-2 encoding:**
```
LDR R0, =label    ; 4 bytes (32-bit instruction)
LDR R1, =50       ; 4 bytes (32-bit instruction)
STR R1, [R0]      ; 2 bytes (16-bit instruction if offset small)
BX LR             ; 2 bytes (16-bit instruction)
Total: 4 + 4 + 2 + 2 = 12 bytes
```

**For 1000 variables: 1000 × 12 = 12,000 bytes (12KB) wasted**

### **3.2. Startup Code memcpy Efficiency**

```assembly
memcpy:              ; Standard library implementation
    .loop:
        LDRB R3, [R1], #1  ; 2 bytes
        STRB R3, [R0], #1  ; 2 bytes
        SUBS R2, R2, #1    ; 2 bytes
        BNE .loop          ; 2 bytes
        BX LR              ; 2 bytes
Total: ~10 bytes for the loop + 4 bytes for call = **14 bytes**

; But wait, the loop is reused for ALL variables
; So amortized cost: 14 bytes TOTAL, not per variable
```

### **3.3. Where the 20 Bytes Come From**

**The "20 bytes" includes:**
- Loading 3 registers (LDR R0, R1, R2): 12 bytes
- BL memcpy: 4 bytes
- Return: 2 bytes
- **Padding/alignment: 2 bytes**
- **Total: 20 bytes** (one-time cost)

**Where the 1000 values are stored:**
```
Flash at 0x08004000:
  32 00 00 00  ; config1 = 50
  64 00 00 00  ; config2 = 100
  96 00 00 00  ; config3 = 150
  ... 997 more ...
  (4000 bytes total)

RAM at 0x20000000:
  [Empty space reserved]
  (4000 bytes total)

Startup:
memcpy(0x20000000, 0x08004000, 4000);  // One call, copies all
```

---

## **4. Memory Sections Deep Dive**

### **4.1. `.bss` Size Clarification**

Your question: *"why 64 bytes int is 32?"*

**You are correct**: `int` is **4 bytes**, not 64.

The "64 bytes" was an **example** for a **larger structure**:
```c
struct {
    int a, b, c, d, e, f, g, h, i, j, k, l, m, n, o, p;  // 16 ints
} large_struct;  // 16 × 4 = 64 bytes
```

**For a single `int global_uninit;`:**
```
.bss uses: 4 bytes RAM
.bss flash: 0 bytes
```

### **4.2. The Economics: Why Save Flash at Cost of RAM**

#### **Resource Comparison (STM32F103C8)**
| Resource | **Size** | **Cost Impact** | **Why It Matters** |
|----------|----------|----------------|-------------------|
| **Flash** | 64KB | **$0.15 per KB** × 1M units = $150K | Bigger MCU = higher cost |
| **RAM** | 20KB | **$0.05 per KB** × 1M units = $50K | Cheaper to add RAM |
| **CPU cycles** | 72MHz | **Battery life** - each cycle costs power | Longer runtime = better product |

**Scenario: You have 1,000 uninitialized globals**

**Option A: Put in .bss (RAM only)**
- Flash cost: 0 bytes
- RAM cost: 4,000 bytes
- **Total: $0.20 per unit**

**Option B: Put in .data (RAM + Flash)**
- Flash cost: 4,000 bytes
- RAM cost: 4,000 bytes
- **Total: $0.80 per unit**

**For 1 million units: $200K vs $800K savings**

**But RAM is smaller - why is this okay?**
- Most globals are **pointers, counters, flags** (small)
- Only **10-20%** of globals need initialization
- **Typical project:** 10KB .bss, 2KB .data
- **RAM savings bigger than Flash cost**

**What if you have 50KB .bss and 10KB .data?**
- You need **bigger MCU** with more RAM (e.g., STM32F103RE: 512K Flash, 64K RAM)
- Cost: $3.50 vs $2.50 per unit
- **Still cheaper than storing zeros in Flash**

### **4.3. Why Zero? Characteristics of the Chosen Number**

**Why not 0xDEADBEEF or 0xFFFFFFFF?**

```c
int x;  // .bss
```

**Zero is special because:**
1. **Math identity:** `x + 0 = x`, `x * 0 = 0`
2. **Pointer null:** `(void*)0` is NULL pointer
3. **Boolean false:** `if(x)` works correctly
4. **String terminator:** `'\0'` is 0
5. **Hardware reset state:** Many peripherals power up as 0

**If .bss was initialized to 0xDEADBEEF:**
```c
int x;  // x = -559038737 (signed)
if(x) { } // Always true - surprising!
char *p; // p = 0xDEADBEEF -> invalid address
```

**The "neutral value" concept:**
- **Zero is the only number** that behaves correctly for all data types
- **Any other number** requires special-case handling

---

## **5. Complete Memory Layout Diagram**

```
╔═══════════════════════════════════════════════════════════════╗
║                    FLASH (512KB)                              ║
║  0x08000000 ─┬─ .isr_vector (512 bytes)                      ║
║              │   - Initial SP value                          ║
║              │   - Reset_Handler address                     ║
║              │   - 60+ ISR addresses                         ║
║              ├─ .text (150KB)                                ║
║              │   - Reset_Handler code                        ║
║              │   - main() code                               ║
║              │   - All functions                             ║
║              ├─ .rodata (20KB)                               ║
║              │   - const tables                              ║
║              │   - String literals                           ║
║              ├─ .data_init (4KB)                             ║
║              │   - Copy of .data initial values              ║
║              └─ (Unused Flash space)                         ║
╠═══════════════════════════════════════════════════════════════╣
║                    RAM (64KB)                                 ║
║  0x20000000 ─┬─ .data (2KB)                                  ║
║              │   - global_init = 50                          ║
║              │   - static int x = 10                         ║
║              ├─ .bss (10KB)                                  ║
║              │   - int global_uninit                         ║
║              │   - static int y                              ║
║              ├─ Heap (grows UP) (40KB)                       ║
║              │   - malloc() area                             ║
║              │   - Managed by heap pointer                   ║
║              ├─ (Unused gap) (4KB)                           ║
║              │   - Magic number: 0xDEADBEEF at 0x2000C000    ║
║              └─ Stack (grows DOWN) (8KB)                    ║
║                  - SP starts at 0x2000FFFC                    ║
║                  - Local variables                            ║
║                  - Return addresses                           ║
╚═══════════════════════════════════════════════════════════════╝

Addresses:
- Stack top:    0x2000FFFC
- Stack bottom: 0x2000E000 (8KB stack)
- Heap start:   0x20002800 (after .bss)
- Heap end:     0x2000BFFF (before magic)
- Magic:        0x2000C000 (overflow detection)
- .bss start:   0x20000800
- .bss end:     0x200027FF (10KB)
- .data start:  0x20000000
- .data end:    0x200007FF (2KB)
```

---

## **6. Stack Mechanics Deep Dive**

### **6.1. LR (Link Register) Deep Dive**

**What is LR?** 
- **ARM register R14**
- **Purpose:** Stores **return address** from function call
- **Special behavior:** **Not a general-purpose register** (has hardware semantics)

### **6.2. Function Call Mechanics**

```c
void caller(void) {
    int x = 5;
    callee();
    x = 10;  // <- Must return here
}
```

**Assembly step-by-step:**
```assembly
caller:
    PUSH {R4, LR}       ; Save LR (4 bytes on stack)
    SUB SP, SP, #4      ; Allocate x
    MOV R0, #5
    STR R0, [SP]        ; x = 5
    
    BL callee           ; Branch with Link:
                        ; 1. LR = 0x08000124 (address of next instruction)
                        ; 2. PC = address of callee
                        ; 3. Start executing callee
    
    ; After callee returns:
    LDR R0, [SP]        ; x = 10
    MOV R0, #10
    STR R0, [SP]
    ADD SP, SP, #4      ; Deallocate x
    POP {R4, PC}        ; Pop saved LR into PC (return)
```

**Why LR is special:**
- **Hardware automatically sets LR** on `BL` instruction
- **Hardware uses LR on `BX LR`** to return
- **Cannot be used as temp register** inside functions (or you lose return address)

### **6.3. What Happens If You Corrupt LR**

```c
void bad(void) {
    asm("MOV LR, #0");  // LR = 0
    return;             // BX LR tries to jump to 0x0
                        // HardFault!
}
```

---

## **7. Dual Stack Pointers (PSP) Context Switch**

### **7.1. Creating a Task**

```c
TaskHandle_t xTaskCreate(void (*task)(void*), void *params) {
    // Allocate task control block
    TCB_t *tcb = malloc(sizeof(TCB_t));
    
    // Allocate stack for task
    tcb->stack = malloc(1024);  // PSP will point here
    
    // Set up initial stack frame
    uint32_t *stack = tcb->stack + 1024;  // Stack grows down
    
    *--stack = 0x01000000;  // xPSR (thumb bit)
    *--stack = (uint32_t)task;  // PC (entry point)
    *--stack = 0xFFFFFFFD;  // LR (return to thread mode with PSP)
    *--stack = 0;  // R12
    *--stack = 0;  // R3
    *--stack = 0;  // R2
    *--stack = 0;  // R1
    *--stack = (uint32_t)params;  // R0 (parameter)
    
    tcb->sp = stack;  // PSP will be set to this
    return tcb;
}
```

### **7.2. OS Schedules First Task**

```c
void vTaskStartScheduler(void) {
    // Load first task's TCB
    TCB_t *task = ready_list[0];
    
    // Switch to PSP (Process Stack Pointer)
    __asm volatile("MRS R0, PSP");  // Get current PSP
    __asm volatile("MSR PSP, %0" : : "r"(task->sp));  // Set new PSP
    
    // Switch to thread mode (use PSP instead of MSP)
    __asm volatile("MOV R0, #0x02");  : Set CONTROL[1] = 1
    __asm volatile("MSR CONTROL, R0");
    
    // Return from exception - this loads PC from task stack!
    __asm volatile("BX LR");
    
    // CPU hardware pops R0-R12, LR, PC from PSP stack
    // Starts executing task at its entry point
}
```

### **7.3. What Happens During Task Execution**

```c
void my_task(void *p) {
    int local = 5;  // On PSP stack at 0x20002FFC
    // ...
}

// PSP is now 0x20003000 - 8 = 0x20002FF8
// [0x20002FF8] = 5 (local variable)
// [0x20002FFC] = 0x20003000 (previous SP value)
```

### **7.4. When ISR Fires During Task**

```assembly
SysTick_Handler:
    ; CPU automatically switches to MSP!
    ; MSP is still 0x2000FFFC (safe, separate)
    ; PSP is frozen at 0x20002FF8 (task stack)
    
    ; Save task context:
    MRS R0, PSP
    STMDB R0!, {R4-R11}  ; Save task's R4-R11 on its stack
    
    ; Update TCB:
    LDR R1, =current_task
    STR R0, [R1]         ; Save new PSP value
    
    ; Choose next task:
    BL vTaskSwitchContext
    
    ; Load next task stack:
    LDR R0, [R1]
    LDMIA R0!, {R4-R11}  ; Restore R4-R11
    
    ; Return to task:
    MSR PSP, R0          ; Set PSP to new task
    BX LR                ; CPU pops R0-R12, PC from new PSP
```

**Why two stacks save you:**
- **MSP overflow:** Corrupts OS → system crash (detectable)
- **PSP overflow:** Corrupts task → OS can kill task, system survives
- **Isolation:** Task bug can't bring down OS

---

## **8. Upward Stack Growth: The Silent Killer**

### **Hypothetical Upward Growth**
```
RAM Layout:
0x2000FFFF ─┐
            │
            │ (Unused)
            │
0x20002000 ─┼─ Stack Limit
            │ Stack grows UP
0x20001000 ─┼─ Stack Start (SP)
            │
            │ .bss and .data (globals)
0x20000000 ─┤
```

**What happens on overflow:**
```c
void deep_recursion(void) {
    char buffer[8192];  // 8KB
    // Stack is at 0x20001800
    // Buffer allocated from 0x20001800 to 0x20003800
    // **OVERFLOWS past 0x20002000**
    // Overwrites global variables at 0x20000000+
}
```

**Result:**
```c
// Before overflow:
int global_x = 50;  // At 0x20000000, value is 50

// After overflow:
// Stack wrote 0x20002000 bytes of garbage
// global_x now contains 0x12345678 (corrupted)

// Your program keeps running!
// But global variables are wrong → silent data corruption
// This is **untraceable** because crash happens much later
```

**Why downward growth is better:**
```
With downward growth, overflow hits:
0x2000F000 ─┼─ Stack Limit
            │ Stack grows DOWN
0x2000EFFF ─┤
            │ MAGIC NUMBER 0xDEADBEEF
            │
            │ Heap (or unused)
            │
0x20000000 ─┤
```

**Overflow writes to 0x2000EFFC → overwrites magic number**
```c
void check_overflow(void) {
    if (*(uint32_t*)0x2000EFFC != 0xDEADBEEF) {
        // Overflow detected immediately!
        // You know exactly what happened
        reboot();
    }
}
```

---

## **9. Magic Numbers and Overflow Detection**

### **9.1. Where the Magic is Set**

```c
#define STACK_CHECK_MAGIC 0xDEADBEEF

void main(void) {
    // Place magic number at bottom of stack gap
    *(uint32_t*)0x2000EFFC = STACK_CHECK_MAGIC;
    
    // Run program
    while(1) {
        check_overflow();
        do_work();
    }
}
```

**Alternative: Linker script defines it**
```ld
/* In linker.ld */
_stack_check = ORIGIN(RAM) + LENGTH(RAM) - 0x1000 - 4;
_stack_check_value = 0xDEADBEEF;
```

### **9.2. Why 0xDEADBEEF?**

- **Unlikely value:** Not a valid pointer or normal data
- **Easy to spot in debugger:** Memory window shows DEADBEEF
- **ASCII:** DEAD BEEF (memorable)

---

## **10. Heap Corruption During ISR**

### **Timeline of Disaster**

**Initial state:**
```
Free list: head → [100 free] → [200 free] → [500 free]
```

**Main thread:**
```c
void main(void) {
    void *p = malloc(150);  // Walking the list...
    // Inside malloc:
    current = free_list_head;  // current = 0x20001000 (first block)
    // Found block [200 free] at 0x20001100
    // About to split it...
}
```

**CPU context right before split:**
```
R0 = 0x20001100  (block to split)
R1 = 150         (size needed)
R2 = 0x20001100  (where to write new header)
```

**ISR fires:**
```assembly
SysTick_Handler:
    PUSH {R0-R3, LR}  ; Save context (R0=0x20001100)
    ; ISR calls malloc(50):
    void *p = malloc(50);  // Corrupts free_list_head!
    ; malloc finds [100 free] block
    ; Splits it, writes new header at 0x20001000
    ; free_list_head = 0x20001000 (updated)
    POP {R0-R3, LR}   ; Restore context (R0=0x20001100, but list changed!)
    BX LR             ; Return to main
```

**Back in main:**
```c
    // malloc continues where it left off:
    // R0 = 0x20001100 (still)
    // BUT free_list_head is now 0x20001000!
    // Splits block at WRONG address
    // Writes header that corrupts heap metadata
    // Next malloc() crashes
```

**Result:** **Heap metadata corruption** → `malloc()` returns pointer to already-used memory → double-free or hard fault.

---

## **11. constexpr and Flash Storage**

### **11.1. The Address Rule**

```cpp
constexpr int x = 42;  // C++: Compile-time constant

// What compiler generates:
// If you NEVER take address:
int y = x * 2;  // Compiler replaces with int y = 84;
                // No memory allocated anywhere!

// If you take address:
const int *p = &x;  // OOPS!
```

**Compiler options:**
1. **Create RAM copy** (most compilers):
```assembly
.data:
    x: .word 42   ; RAM at 0x20000000
    
.text:
    LDR R0, =x    ; Load address of RAM copy
```

2. **Create Flash address** (optimized):
```assembly
.rodata:
    x: .word 42   ; Flash at 0x08001000
    
.text:
    LDR R0, =x    ; Load address of Flash location
```

**The standard doesn't guarantee Flash placement** - it's implementation-defined.

### **11.2. Compiler Options**

```cpp
// Option 1: RAM copy (most compilers)
.data:
    x: .word 42   ; RAM at 0x20000000
    
.text:
    LDR R0, =x    ; Load address of RAM copy

// Option 2: Flash address (optimized)
.rodata:
    x: .word 42   ; Flash at 0x08001000
    
.text:
    LDR R0, =x    ; Load address of Flash location
```

### **11.3. Solution: Use `const` and `static`**

```cpp
static const int x = 42;  // FORCE Flash placement
// static: internal linkage (can't be extern'd)
// const: read-only data → goes to .rodata
```

**Or in C:**
```c
const int x = 42;  // Global const → .rodata
// Don't take address, or take address and cast:
#define X_ADDR ((const int*)0x08001000)
```

---

## **12. `extern` in Function Scope: The Truth**

```c
void func(void) {
    extern int x;  // Declares global 'x' with external linkage
    x = 10;        // Modifies the SAME 'x' as file scope
}

// Equivalent to:
extern int x;      // File scope declaration
void func(void) {
    x = 10;
}
```

### **12.1. Scoping Rules**

```c
// file1.c
int global_x = 5;  // Definition

// file2.c
void func(void) {
    extern int global_x;  // Declaration - refers to file1.c's variable
    global_x = 10;        // Works!
}

// file3.c
void another(void) {
    // global_x is visible here too (global scope)
}
```

### **12.2. Why It's Allowed**

What the standard says:
- `extern` at block scope declares a name with **external linkage**
- It **refers to** the same entity as file-scope declaration
- It's **bad style** because it hides the global nature

### **12.3. Correct Usage**

```c
// globals.h
extern int global_x;  // Declaration (in header)

// globals.c
int global_x = 5;     // Definition (in one .c file)

// any_file.c
#include "globals.h"  // Brings declaration into scope
void func(void) {
    global_x = 10;    // No extern needed here
}
```

---

## **13. Bootloader vs Application Vector Tables**

### **13.1. Why Both Need Vectors**

**CPU hardware requirement:**
- After **any** reset, CPU reads SP/PC from **address 0x00000000**
- That's **always bootloader's vector table**

### **13.2. Bootloader's Job**

```c
void bootloader(void) {
    // 1. Check if button pressed → stay in bootloader
    if (button) {
        // Run bootloader commands
    } else {
        // 2. Jump to application
        jump_to_app(0x08008000);
    }
}
```

### **13.3. Application Vector Table Layout**

```
0x08008000:  0x20018000  (app's own stack pointer)
0x08008004:  0x08008041  (app's Reset_Handler address)
0x08008008:  0x08008081  (app's NMI_Handler address)
...
0x080080C0:  0x0800A001  (app's USART1_IRQ address)
```

### **13.4. What Happens After VTOR Remap**

```c
// After SCB->VTOR = 0x08008000:
CPU interrupt logic:
- Systick fires
- CPU reads Systick vector from 0x0800803C (new table)
- Jumps to application's handler
- Application works correctly
```

---

## **14. Symbol Table Decoding Complete Legend**

```c
SYMBOL TABLE:
00000000 l    df *ABS*  00000000 main.c
00000000 l    d  .text  00000000 .text
00000000 g     F .text  0000001c main
00000000 g     O .data  00000004 global_var
00000000         *UND*  00000000 printf
```

**Column breakdown:**

| Column | **Value** | **Meaning** | **Example** |
|--------|-----------|-------------|-------------|
| **Address** | `00000000` | Symbol location (relative in .o) | Where in file |
| **Bind** | `l` or `g` | **l** = local, **g** = global | Visibility |
| **Type** | `df`, `d`, `F`, `O` | **d** = section, **F** = function, **O** = object | What it is |
| **Section** | `*ABS*`, `.text`, `.data` | Where symbol lives | Memory region |
| **Size** | `0000001c` | Symbol size in hex (1c = 28 bytes) | How big |
| **Name** | `main` | Symbol name | Identifier |

**Bind column (`l` vs `g`):**
-  ** `l` (local)**  : Symbol only visible in this .o file (like `static`)
  ```c
  static int x;  // Shows as 'l' in symbol table
  ```
-  ** `g` (global)**  : Symbol visible to linker (can be extern'd)
  ```c
  int x;  // Shows as 'g' in symbol table
  ```

**Type column:**
-  **`F` (function)**  : Executable code
  ```c
  void func(void);  // F type
  ```
-  **`O` (object)**  : Data variable
  ```c
  int x;  // O type
  ```
-  **`d` (section)**  : Section marker (not a real symbol)
  ```c
  .text:  // d type
  ```
-  **`df` (file)**  : Source file name

**Section column:**
-  **`*UND*`**  : Undefined (needs linker resolution)
  ```c
  extern int x;  // UND until link time
  ```

**What UND means to linker:**
```
printf *UND* → Search libc.a → Found in printf.o → 
Replace all UND references with actual printf address
```

---

## **15. Executive Summary: The Why Matrix**

| Question | **Why This Way** | **What If Wrong** | **Cost** |
|----------|------------------|-------------------|----------|
| **Lookup tables** | Runtime math too slow | System too slow | Flash for table |
| **.data section** | Bulk init saves Flash | Code bloat | 20 bytes startup |
| **.bss zero** | Flash savings + standard | Unpredictable globals | RAM usage |
| **Downward stack** | Detect overflow early | Silent corruption | Gap needed |
| **Two stack pointers** | OS safety | No multitasking | Hardware feature |
| **Magic number** | Detect overflow | Untraceable bugs | 4 bytes RAM |
| **EXTERN scope** | Linker visibility | Multiple definition | None |
| **Vector tables** | Hardware requirement | Can't boot | 512 bytes Flash |

**Every mechanism is a tradeoff between resource cost and failure mode severity.**