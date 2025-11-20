# 2 - Embedded-C-Notes

### **Table of Contents**
1. **[Core Embedded C Concepts](#1-core-embedded-c-concepts)**
   - 1.1. [Compile-Time vs Runtime: Two Different Worlds](#11-compile-time-vs-runtime-two-different-worlds)
   - 1.2. [`volatile`: The Nuclear Option (Compiler vs Hardware)](#12-volatile-the-nuclear-option-compiler-vs-hardware)
   - 1.3. [Determinism: The Embedded Holy Grail](#13-determinism-the-embedded-holy-grail)
   - 1.4. [What is Deterministic Code?](#14-what-is-deterministic-code)

2. **[Assembly Instructions & Hardware](#2-assembly-instructions--hardware)**
   - 2.1. [STR (Store Register): Direct Hardware Access](#21-str-store-register-direct-hardware-access)
   - 2.2. [Bit Manipulation Instructions (BSET/BCLR/BFC/BFI)](#22-bit-manipulation-instructions-bsetbclrbfcbfi)
   - 2.3. [Why BSET/BCLR Are Atomic](#23-why-bsetbclr-are-atomic)

3. **[Compilers & Toolchain](#3-compilers--toolchain)**
   - 3.1. [Keil MDK vs IAR EW vs GCC](#31-keil-mdk-vs-iar-ew-vs-gcc)
   - 3.2. [`#pragma`: Compiler Directives](#32-pragma-compiler-directives)
   - 3.3. [`#pragma once` vs Include Guards](#33-pragma-once-vs-include-guards)

4. **[Memory Architecture Deep Dive](#4-memory-architecture-deep-dive)**
   - 4.1. [Complete Memory Map (Flash & RAM)](#41-complete-memory-map-flash--ram)
   - 4.2. [.text: Executable Code](#42-text-executable-code)
   - 4.3. [.rodata: Read-Only Data](#43-rodata-read-only-data)
   - 4.4. [.data: Initialized Globals](#44-data-initialized-globals)
   - 4.5. [.bss: Zero-Initialized Globals](#45-bss-zero-initialized-globals)
   - 4.6. [Stack vs .data/.bss: Three Different Beasts](#46-stack-vs-databss-three-different-beasts)
   - 4.7. [Stack Growth: The PDP-11 Legacy](#47-stack-growth-the-pdp11-legacy)

5. **[Stack & Context Management](#5-stack--context-management)**
   - 5.1. [SP Register: The Stack Pointer](#51-sp-register-the-stack-pointer)
   - 5.2. [Why Modifying SP is Instant Death](#52-why-modifying-sp-is-instant-death)
   - 5.3. [Context Switch Mechanism](#53-context-switch-mechanism)
   - 5.4. [Why SP Makes Context Switching Fast](#54-why-sp-makes-context-switching-fast)

6. **[Dynamic Memory Allocation](#6-dynamic-memory-allocation)**
   - 6.1. [How Malloc Works & Fragmentation](#61-how-malloc-works--fragmentation)
   - 6.2. [Why Malloc Time Varies](#62-why-malloc-time-varies)
   - 6.3. [Heap Fragmentation Deep Dive](#63-heap-fragmentation-deep-dive)
   - 6.4. [Why Heap is Banned in Critical Systems](#64-why-heap-is-banned-in-critical-systems)
   - 6.5. [Malloc Corruption Scenarios](#65-malloc-corruption-scenarios)

7. **[Language Features & Pitfalls](#7-language-features--pitfalls)**
   - 7.1. [`const` Local vs Global: Why One Can Be Hacked](#71-const-local-vs-global-why-one-can-be-hacked)
   - 7.2. [Function Pointers: Reading Complex Declarations](#72-function-pointers-reading-complex-declarations)
   - 7.3. [The "Right-Left-Right" Rule](#73-the-right-left-right-rule)
   - 7.4. [Generic Pointer Types](#74-generic-pointer-types)
   - 7.5. [Far vs Near Pointers: Legacy Concepts](#75-far-vs-near-pointers-legacy-concepts)

8. **[Real-World Applications & Failure Modes](#8-real-world-applications--failure-modes)**
   - 8.1. [Lookup Tables in .rodata](#81-lookup-tables-in-rodata)
   - 8.2. [DSP Loop: Digital Signal Processing](#82-dsp-loop-digital-signal-processing)
   - 8.3. [PID Loop: Motor Control](#83-pid-loop-motor-control)
   - 8.4. [Race Conditions: Step-by-Step Breakdown](#84-race-conditions-step-by-step-breakdown)
   - 8.5. [Atomic Operations: Why They Matter](#85-atomic-operations-why-they-matter)

9. **[Macro Expansion & Preprocessor](#9-macro-expansion--preprocessor)**
   - 9.1. [Macro Expansion Failure: If/Else Mystery](#91-macro-expansion-failure-ifelse-mystery)
   - 9.2. [Do-While-Zero Idiom](#92-dowhilezero-idiom)
   - 9.3. [Concatenation Flow & Helper Macros](#93-concatenation-flow--helper-macros)

10. **[Summary & Best Practices](#10-summary--best-practices)**
    - 10.1. [Memory Section Characteristics Matrix](#101-memory-section-characteristics-matrix)
    - 10.2. [The Embedded C Mantra](#102-the-embedded-c-mantra)

---

## **1. Core Embedded C Concepts**

### **1.1. Compile-Time vs Runtime: Two Different Worlds**

| Aspect | **Compile Time** | **Runtime** |
|--------|------------------|-------------|
| **Location** | On your PC, in compiler | On MCU, in hardware |
| **Process** | Text → Assembly → Object code | Instructions execute, memory changes |
| **Agent** | Compiler (GCC, Clang, Keil) | CPU core (ARM Cortex-M) |
| **Error Detection** | Syntax errors, type mismatches | Hard faults, watchdog resets |
| **Key Tools** | Preprocessor, optimizer, linker | CPU, peripherals, memory |
| **Example** | `#define MAX 100` becomes `100` everywhere | `int x = MAX;` allocates RAM and stores value |

**Analogy:** Compile time = blueprint review; Runtime = actual construction.

**What if `volatile` didn't exist?**
```c
int *reg = (int*)0x40020000;
while(*reg & 0x01) { /* wait */ }  // Compiler optimizes to infinite loop!
```
**Result:** System hangs because compiler assumes memory is stable (false in embedded).

---

### **1.2. `volatile`: Compiler vs Hardware Deep Dive**

`volatile` is a **compile-time directive** that forces **runtime memory access**.

```c
volatile int *reg = (int*)0x40020000;
*reg = 0x5A;
```

**Compile-Time Action:** Compiler adds variable to "do not optimize" list and generates memory-access instructions.

**Runtime Action:** Hardware executes instruction and modifies register.

**The Core Problem: Compiler Assumptions**
- Compiler assumes memory is passive and only changes when *your code* writes to it.
- This allows register caching, dead store elimination, and instruction reordering.

**These assumptions are FALSE for:**
- **Hardware registers**: Change autonomously (ADC conversion complete, UART byte received)
- **ISR-shared variables**: Modified by interrupt context
- **Multi-core memory**: Changed by another CPU core

**Real Disaster Without `volatile`:**
```c
#define DMA_DONE (*(uint32_t*)0x40020000)
void start_dma(void) {
    DMA_CTRL = 0x01;  // Start transfer
    while(!DMA_DONE); // Wait for completion
}
```
**Without `volatile`, compiler sees `*reg` doesn't change inside loop → generates assembly that reads once → infinite loop → system hangs → watchdog reset → product fails in field.**

**Correct Assembly:**
```assembly
loop:
    LDR R1, [R0]      // Re-read from hardware EVERY iteration
    TST R1, #1
    BNE loop          // Hardware-responsive
```

**Critical Distinction: `volatile` vs Atomic**
```c
volatile int flag = 0;
void main(void) {
    flag = 0;
    while(flag == 0) { } // Race condition possible!
}
```
**Why `volatile` isn't enough:** It ensures the read happens, but the read is in a CPU register. An ISR can't modify that register. **Atomic operations (LDREX/STREX) are required for true safety.**

---

### **1.3. Determinism: The Embedded Holy Grail**

**Deterministic** = **Guaranteed execution time, every time, under all conditions.**

```c
// Deterministic:
void isr_handler(void) {
    GPIOA->ODR |= 0x01;  // Always 12 cycles on STM32F103
}

// NON-Deterministic:
void risky(void) {
    int *p = malloc(100);  // Could take 5μs or 5ms!
    free(p);
}
```

**Why determinism is mandatory:**
- Motor control loop must run every **100μs exactly**. A 1μs jitter causes audible noise.
- If `malloc()` takes 5ms due to fragmentation, motor crashes into limit switch.
- **Result:** Physical damage, safety hazard.

**What if code wasn't deterministic?** Your quadcopter's PID controller runs late → erratic flight → crash. This is why safety standards **ban dynamic allocation**.

---

### **1.4. What is Deterministic Code?**

```c
// Deterministic code: ALWAYS 250 cycles, EVERY time
int good_dsp_loop(void) {
    static int16_t buffer[128];  // .bss, pre-allocated
    output = fixed_sqrt(sample) * gain >> 16;  // Fixed-point sqrt = 45 cycles
    PWM_TIMER->CCR1 = output;  // Direct register write (1 cycle)
}
```

**The Three Rules:**
1. **Is execution time constant?** `NO` → Jitter → Control loop instability
2. **Is memory access deterministic?** `NO` → Cache miss/DMA conflict → Deadline miss
3. **Can I count cycles by reading assembly?** `NO` → Hidden costs → Budget overflow

---

## **2. Assembly Instructions & Hardware**

### **2.1. STR (Store Register): Direct Hardware Access**

```assembly
STR R0, [R1, #0x18]  // Store R0 to address (R1 + 0x18)
```
**What it does:** Writes 32-bit value from CPU register R0 to memory address calculated by R1 + 0x18.

**On ARM Cortex-M:** Single instruction (1 cycle if cached). Directly corresponds to: `*(volatile uint32_t*)(R1 + 0x18) = R0;`

---

### **2.2. Bit Manipulation Instructions**

| Instruction | Stands For | What It Does | **Why It Matters** |
|-------------|------------|--------------|-------------------|
| **BSET** | Bit Set | `BSET reg, #5` sets bit 5 to 1 | **Atomic operation** - prevents race conditions |
| **BCLR** | Bit Clear | `BCLR reg, #5` clears bit 5 | Same as above |
| **BFC** | Bit Field Clear | `BFC R0, #8, #4` clears bits 8-11 | ARM-specific, clears range atomically |
| **BFI** | Bit Field Insert | `BFI R0, R1, #8, #4` inserts R1[3:0] into R0[11:8] | ARM-specific, inserts bits atomically |

**What if these didn't exist?**
```c
// Without BSET: 5 cycles
uint32_t temp = *REG;      // Read (2 cycles)
temp |= (1 << 5);          // Modify (1 cycle)
*REG = temp;               // Write (2 cycles)

// With BSET: 1 cycle
*REG |= (1 << 5);          // Single instruction, atomic
```
**Why atomic matters:** If ISR fires between read and write, it could modify the same register. Your write would overwrite its change. **BSET/BCLR prevents this.**

---

### **2.3. Why BSET/BCLR Are Atomic**

**Atomic** = **Single, indivisible instruction** that cannot be interrupted.

```c
reg |= (1 << 5);  // Non-atomic: LDR-ORR-STR (3 instructions)
```
**Race window:** ISR can fire between LDR and STR → lost change.

```c
BSET reg, #5;  // Atomic: Single instruction
```
**Race window:** **None.** Hardware guarantees completion before interrupt recognition.

---

## **3. Compilers & Toolchain**

### **3.1. Keil MDK vs IAR EW vs GCC**

| Tool | **What It Is** | **Why Use It** | **Cost** | **Best For** |
|------|----------------|----------------|----------|--------------|
| **Keil MDK** | IDE + ARM Compiler | Optimized for ARM, excellent debugger | $5000/year | ARM Cortex-M development |
| **IAR EW** | IDE + IAR Compiler | Best optimization, safety certified | $3000/year | Automotive/medical (MISRA) |
| **GCC** | Open-source compiler | Free, customizable, industry standard | Free | Linux, open-source projects |

**Why Keil/IAR matter:**
- Generate **10-20% tighter code** than GCC
- Have **certified compilers** for safety (provable correctness)
- Provide **hardware debugging integration** (Cortex-M trace)

---

### **3.2. `#pragma`: Compiler Directives**

**What is `#pragma`?**
- **NOT** a preprocessor directive (despite `#`)
- **Direct command to the compiler** (bypasses C standard)
- **Implementation-defined**: Each compiler interprets it differently

```c
#pragma pack(1)       // GCC: "Pack structs tight"
__packed struct { }   // ARM Compiler: "Pack structs tight"
```

**What if `#pragma` didn't exist?** You couldn't control:
- Memory alignment (`#pragma pack`)
- Interrupt vector placement (`#pragma vector=0x08`)
- Function priority (`#pragma startup`)
- Section placement (`#pragma section .fastcode`)

**The compiler would use defaults that might not match your hardware.**

---

### **3.3. `#pragma once` vs Include Guards**

```c
#pragma once  // Include guard alternative
```
**What it does:** Tells compiler "Only process this header file once per compilation."

**vs `#ifndef HEADER_H`?**:
- `#pragma once`: Faster (compiler tracks file path)
- `#ifndef`: Portable (works everywhere)

**Why both exist?** Historical. `#pragma once` is newer but not standard C. Most compilers support it now.

---

## **4. Memory Architecture Deep Dive**

### **4.1. Complete Memory Map (Flash & RAM)**

```
╔═══════════════════════════════════════════════════════════════╗
║                    FLASH (ROM) - 512KB @ 0x08000000           ║
║  0x08000000 ─┬─ .isr_vector (512 bytes)                      ║
║              │   - Initial SP value                          ║
║              │   - Reset handler address                     ║
║              ├─ .text (150KB)                                ║
║              │   - All executable code                       ║
║              │   - Literal pools                             ║
║              ├─ .rodata (20KB)                               ║
║              │   - const tables, strings                     ║
║              └─ .data_init (4KB)                             ║
║                  - Initial values for .data                  ║
╠═══════════════════════════════════════════════════════════════╣
║                    RAM - 64KB @ 0x20000000                    ║
║  0x20000000 ─┬─ .data (2KB)                                  ║
║              │   - global_init = 50 (copied from Flash)     ║
║              ├─ .bss (10KB)                                   ║
║              │   - global_uninit (zeroed)                    ║
║              ├─ Heap (40KB, grows UP)                         ║
║              │   - malloc() area                             ║
║              ├─ Stack Gap (4KB)                               ║
║              │   - Magic: 0xDEADBEEF at 0x2000C000           ║
║              └─ Stack (8KB, grows DOWN)                       ║
║                  - Local variables                            ║
╚═══════════════════════════════════════════════════════════════╝
```

---

### **4.2. .text: Executable Code**

**What's inside .text:**
```assembly
08000100 <Reset_Handler>:
 08000100:   4b02        ldr r3, [pc, #8]    ; Load __data_init address
 08000102:   4a03        ldr r2, [pc, #12]   ; Load __data_start address
 08000104:   4c04        ldr r4, [pc, #16]   ; Load __data_size
 08000106:   f000 f804   bl 0x08001112       ; Call memcpy
```
**Characteristics:**
- **Read-only** (hardware fault on write)
- **Execute-only** on modern MCUs
- **Simultaneous fetch** on Harvard architecture (code & data buses separate)

---

### **4.3. .rodata: Read-Only Data**

```c
const int16_t sine_lookup[360] = {0, 17, 35, ...};  // 720 bytes
const char *msg = "Error: Overvoltage";
```

**Why 720 bytes for 360 entries?** The example uses `int16_t` (2 bytes) to save Flash. Standard `int` would be 1,440 bytes.

**What if .rodata didn't exist?**
```c
int *sine_lookup = malloc(720);  // Runtime allocation
// Problems: Wastes RAM, non-deterministic, requires init code
```

---

### **4.4. .data: Initialized Globals**

```c
int global_init = 50;  // Goes to .data
```

**Dual-storage mechanism:**
```
Flash (source): 0x08004000:  32 00 00 00  ; The value 50
RAM (dest):     0x20000000:  ?? ?? ?? ??  ; Uninitialized on power-on
```

**Startup code copies:**
```assembly
LDR R0, =0x20000000  ; Destination: __data_start
LDR R1, =0x08004000  ; Source: __data_init
LDR R2, =4000        ; Size: __data_size (1000 vars × 4 bytes)
BL memcpy            ; Copies from Flash to RAM
```

**What if .data didn't exist?**
```c
int global_init;  // In .bss (zero)
void init(void) {
    global_init = 50;  // Generates 12 bytes per variable
}
// For 1000 variables: 12,000 bytes of Flash wasted on init code!
```

---

### **4.5. .bss: Zero-Initialized Globals**

```c
int global_uninit;      // In .bss
static int file_static; // In .bss
```

**Why zero?**
1. Flash savings: Don't store 1000 zeros in Flash
2. Standard requirement: C mandates static storage starts zeroed
3. Determinism: No random values on startup

**Startup zeroing:**
```assembly
LDR R0, =0x20000400  ; __bss_start
LDR R1, =0x20002800  ; __bss_end
MOV R2, #0
.Lloop:
    STR R2, [R0]      ; Write zero
    ADD R0, R0, #4    ; Next word
    CMP R0, R1
    BNE .Lloop        ; Loop
// Total: ~10 bytes of code, regardless of .bss size
```

---

### **4.6. Stack vs .data/.bss: Three Different Beasts**

| Feature | **Stack** | **.data/.bss** |
|---------|-----------|----------------|
| **Lifetime** | Function call only | Entire program |
| **Allocation** | Runtime (PUSH/POP) | Link time (fixed addresses) |
| **Addressing** | Relative to SP | Absolute (0x20000100) |
| **Content** | Locals, return addresses | Global/static variables |
| **Speed** | 1 cycle (PUSH/POP) | 2-3 cycles (LDR/STR) |
| **Size** | Limited (4-64KB) | Can be large |

**What if stack grew upward?**
```
Stack starts at 0x20000000 (low)
Heap starts at 0x20001000 (high)
Stack overflow corrupts .bss/.data globals SILENTLY
```

---

### **4.7. Stack Growth: The PDP-11 Legacy**

**What is PDP-11?**
- **1970 minicomputer** by DEC
- **First popular system** with downward-growing stack
- **Calling convention:** `JSR` pushed return address, decremented SP
- **Why it stuck:** Became UNIX's platform, influenced x86, ARM, MIPS

**ARM Cortex-M:**
```assembly
PUSH {R0, R1}   // SP = SP - 8, then store
POP {R0, R1}    // Load, then SP = SP + 8
```
**Hardware is hardwired** for this direction.

---

## **5. Stack & Context Management**

### **5.1. SP Register: The Stack Pointer**

**MSP vs PSP:**
```c
__set_MSP(0x20010000);  // Main Stack Pointer (OS kernel)
__set_PSP(0x20009000);  // Process Stack Pointer (user tasks)
```

**CONTROL register:**
```
Bit 1 (SPSEL): 0 = MSP, 1 = PSP
Bit 0 (nPRIV): 0 = Privileged, 1 = Unprivileged
```

**Why PSP matters:** Fault isolation. Task overflow corrupts its own stack, not OS.

---

### **5.2. Why Modifying SP is Instant Death**

```c
void dangerous(void) {
    __asm volatile("MOV SP, #0x12345678");  // BOOM!
}
```
**What happens:**
1. SP points to invalid address
2. Next `PUSH` or `BL` triggers bus fault
3. **Return address is lost** → cannot recover

**Safe debug method:**
```c
register uint32_t original_sp __asm("r0");
__asm volatile("MOV %0, SP" : "=r"(original_sp));  // Copy SP
__asm volatile("ADD SP, SP, #100");  // Temp change
__asm volatile("MOV SP, %0" : : "r"(original_sp));  // Restore
```

---

### **5.3. Context Switch Mechanism**

**PendSV ISR (FreeRTOS):**
```assembly
PendSV_Handler:
    MRS R0, PSP          // Get current task PSP
    STMDB R0!, {R4-R11}  // Save callee-saved registers
    LDR R1, =current_task
    STR R0, [R1]         // Save task's SP to TCB
    BL vTaskSwitchContext  // Choose next task
    LDR R0, [R1]         // Get next task's SP
    LDMIA R0!, {R4-R11}  // Restore registers
    MSR PSP, R0          // Set PSP to new task
    ORR LR, LR, #0x10    // Return to thread mode with PSP
    BX LR                // Exception return
```
**Total switch time:** 12 cycles = **166 ns at 72MHz**.

---

### **5.4. Why SP Makes Context Switching Fast**

**Without hardware SP:**
```c
// Manual context save: 30 instructions
void my_isr(void) {
    __asm volatile("STR R0, [some_temp]");
    __asm volatile("STR R1, [some_temp+4]");
    // ... save all registers ...
    // 30 cycles wasted before ISR logic!
}
```

**With hardware SP: 12 cycles total.** ISR runs immediately.

**Why critical:** Motor control ISR must finish in **5μs**. At 72MHz, that's 360 cycles. 30-cycle overhead = **8% budget wasted**.

---

## **6. Dynamic Memory Allocation**

### **6.1. How Malloc Works & Fragmentation**

```c
// Heap linked list:
struct free_block { size_t size; struct free_block *next; };
```

**Allocation process:**
```c
void *malloc(size_t size) {
    for(block = free_list; block; block = block->next) {
        if(block->size >= size) {
            if(block->size > size + sizeof(struct free_block)) {
                // Split block
                new_block = (char*)block + size;
                new_block->size = block->size - size;
                new_block->next = block->next;
            }
            return block;
        }
    }
    return NULL;  // Out of memory!
}
```

---

### **6.2. Why Malloc Time Varies**

- **Best case:** 1μs (first block fits)
- **Worst case:** 50μs (searches 1000 fragments, splits blocks)
- **Fragmentation killer:** After 10,000 cycles, `malloc(1000)` might take 5ms

**Why banned in deterministic systems:** Cannot guarantee worst-case execution time.

---

### **6.3. Heap Fragmentation Deep Dive**

```
Initial: [Free: 61440]

After malloc(100), malloc(200), free(100):
[Free:100][Used:200][Free:61140]

After malloc(150):
[Free:100][Used:150][Free:50][Used:200][Free:60940]
               ↑ Splits block, creates 50-byte fragment

After many cycles: Memory Swiss cheese. Total free = 1000 bytes, but no block > 100 bytes.
```

---

### **6.4. Why Heap is Banned in Critical Systems**

```c
void loop(void) {
    uint8_t *buffer = malloc(256);  // DANGER!
    free(buffer);  // Fragmentation builds up
}
```

**Why never in loops:** After 1 hour, malloc() time increases from 5μs to 500μs → violates timing.

**Why never in ISRs:** Non-reentrant, unbounded time (5ms vs 5μs).

**MISRA C Rule 21.3:** **"The memory allocation functions shall not be used."**

**Historical failure:** Therac-25 radiation machine used dynamic allocation → race condition → **6 patients overdosed**.

---

### **6.5. Malloc Corruption Scenarios**

**Main thread interrupted by ISR:**
```c
void main(void) {
    p = malloc(100);  // Finds block at 0x20001000
    // About to split it...
    // ← ISR fires here!
}

void SysTick_Handler(void) {
    q = malloc(50);   // Finds SAME block at 0x20001000
    // Corrupts malloc's internal free_list!
}
```
**Result:** malloc state corrupted → system crash on next allocation.

---

## **7. Language Features & Pitfalls**

### **7.1. `const` Local vs Global: Why One Can Be Hacked**

```c
void func(void) {
    const int secret = 42;  // Stack at SP+4
    int *hacker = (int*)&secret;
    *hacker = 0;  // Works! secret is now 0
}
```
**Why:** `const` is a compile-time promise, not hardware enforcement. Secret lives in **RAM**.

```c
const int secret = 42;  // .rodata in Flash
int *hacker = (int*)&secret;
*hacker = 0;  // **HARDFAULT!**
```
**Why:** Flash write requires unlock sequence. Hardware raises bus fault.

---

### **7.2. Function Pointers: Reading Complex Declarations**

```c
int (*func_ptr)(int, int);  // Pointer to function
func_ptr = &add;
int result = func_ptr(5, 3);  // Calls add(5, 3)
```

**Assembly:**
```assembly
LDR R0, =add
MOV R1, #5
MOV R2, #3
BLX R0              // Branch and Link Exchange
```

---

### **7.3. The "Right-Left-Right" Rule**

```c
int (*func_array[10])(int, float);
```
**Parse:**
1. `func_array` → identifier
2. `[10]` → array of 10
3. `*` → pointers
4. `(int, float)` → functions taking int and float
5. `int` → returning int

**Final:** Array of 10 pointers to functions that take (int, float) and return int.

---

### **7.4. Generic Pointer Types**

| Type | Purpose | Valid Operations |
|------|---------|------------------|
| `void*` | Generic pointer | Cast, assignment |
| `int*` | Typed pointer | Dereference, arithmetic |
| `uintptr_t` | Integer address | Bit manipulation |
| `far pointer` | 32-bit segment:offset | **Legacy x86 only, obsolete in ARM** |
| `near pointer` | 16-bit offset only | **Legacy x86 only, obsolete in ARM** |

---

### **7.5. Far vs Near Pointers: Legacy Concepts**

**Historical context:** Intel 8086 had 16-bit registers but 1MB memory (20-bit address) → **segmentation**.

**Near pointer (16-bit):**
```c
char near *p = (char near*)0x1234;  // Offset within current 64KB segment
```
- Size: 2 bytes, Range: 64KB only

**Far pointer (32-bit):**
```c
char far *p = (char far*)0x12340005;  // segment:offset
```
- Size: 4 bytes, Range: 1MB, **Physical = (Segment << 4) + Offset**

**ARM Cortex-M (32-bit flat):**
```c
uint32_t *p = (uint32_t*)0x20000000;  // Flat address
```
- **No far/near distinction:** Every pointer accesses full 4GB space
- **Modern practice:** **Ignore them completely** in ARM embedded development. They are obsolete.

---

## **8. Real-World Applications & Failure Modes**

### **8.1. Lookup Tables in .rodata**

```c
// Temperature: ADC → Celsius
const int16_t temp_lookup[4096] = { -400, -399, /* ... */ 850 };

// FIR filter coefficients
const float filter_taps[64] = { 0.0123, 0.0456, /* ... */ };

// Font bitmap
const uint8_t font_A[8] = {0x3C, 0x66, 0x42, 0x42, 0x66, 0x3C, 0x00, 0x00};
```
**Why .rodata:** Never changes, large size (8KB), fast access, compile-time generated.

---

### **8.2. DSP Loop: Digital Signal Processing**

```c
void fir_filter(const int16_t *input, int16_t *output, size_t n) {
    for(size_t i = 0; i < n; i++) {
        int32_t sum = 0;
        for(size_t j = 0; j < TAPS; j++) {
            sum += input[i - j] * coeffs[j];  // MAC operation
        }
        output[i] = sum >> 15;
    }
}
```

**Why `volatile` would kill performance:**
```c
volatile int32_t sum = 0;  // DON'T!
// Each iteration: 4 cycles (LDR-ADD-STR)
// Without volatile: 2 cycles (keep sum in register)
// Result: 2x slower filter → can't process 48kHz audio
```

---

### **8.3. PID Loop: Motor Control**

```c
void pid_update(void) {
    error = setpoint - actual;
    integral += error;
    derivative = error - last_error;
    output = Kp*error + Ki*integral + Kd*derivative;
    TIM1->CCR1 = output;  // Update PWM
}
```

**Must run every 100μs ±1μs.** Jitter causes motor oscillation, overheating, mechanical failure.

---

### **8.4. Race Conditions: Step-by-Step Breakdown**

```c
int x;  // Shared between main and ISR

void main(void) {
    x = 0;
    while(x == 0) { }  // 3-instruction window
}
```

**Timeline:**
```
Time 0: main executes LDR R2, [R1]    → R2 = 0
Time 1: main executes CMP R2, #0
Time 2: ISR fires                     → ISR sets x = 10 in RAM
Time 3: ISR returns
Time 4: main executes BNE loop        → uses stale R2 = 0
Time 5: **Infinite loop!**
```

**Why it happens:** ISR modifies memory, but main compares **register** (stale value).

---

### **8.5. Atomic Operations: Why They Matter**

```c
// Non-atomic:
reg |= (1 << 5);  // LDR-ORR-STR (3 instructions)

// Atomic:
BSET reg, #5;  // Single instruction
```

**What atomic means:** **Single, indivisible instruction** that cannot be interrupted. ARM provides `BSET`/`BCLR` for bits and `LDREX`/`STREX` for memory with lock.

---

## **9. Macro Expansion & Preprocessor**

### **9.1. Macro Expansion Failure: If/Else Mystery**

```c
#define my_print() {printf("A"); printf("B");};

if(i == 10)
    my_print();  // Expands to: {printf...;}; ; 
else             // ERROR: "else without if"
    x = 5;
```

**Preprocessed output:**
```c
if(i == 10)
    {printf("A"); printf("B");}; ;  // Two statements!
else
    x = 5;  // 'else' belongs to NULL statement
```

---

### **9.2. Do-While-Zero Idiom**

```c
#define my_print() do {printf("A"); printf("B");} while(0)

// Expands to:
if(i == 10)
    do {printf...} while(0);  // Single statement, safe
else
    x = 5;
```
**Why it works:** `do-while` is **one statement** that **requires a semicolon**. No empty statement created.

---

### **9.3. Concatenation Flow & Helper Macros**

```c
#define B3 0
#define B2 1
#define CONCAT(a,b) 0b##a##b  // Fails: produces 0bB3B2

// Two-stage:
#define CONCAT_HELPER(a,b) 0b##a##b
#define CONCAT(a,b) CONCAT_HELPER(a,b)  // Forces expansion first

CONCAT(B3,B2)  // Expands: B3,B2 → CONCAT_HELPER(0,1) → 0b01
```

**Why two stages needed:** `##` concatenates **before** argument expansion. Helper macro forces evaluation first.

---

## **10. Summary & Best Practices**

### **10.1. Memory Section Characteristics Matrix**

| Section | **Purpose** | **Lifetime** | **Speed** | **What If Wrong** |
|---------|-------------|--------------|-----------|-------------------|
| **.text** | Code | Permanent | Fast | If in RAM: firmware lost on power-off |
| **.rodata** | Constants | Permanent | Fast | If in RAM: wastes RAM, can be modified |
| **.data** | Initialized globals | Permanent | Medium | Manual init wastes Flash & cycles |
| **.bss** | Zeroed globals | Permanent | Medium | If in .data: wastes Flash storing zeros |
| **Stack** | Locals | Temporary | **Very Fast** | If in .data: permanent RAM waste |
| **Heap** | Dynamic | Variable | **Slow & Variable** | Non-deterministic, fragmentation |

---

### **10.2. The Embedded C Mantra**

Every decision must answer **YES** to all three:
1. **Is execution time constant?**
2. **Is memory access deterministic?**
3. **Can I count cycles by reading assembly?**

**When in doubt, ask:**
- "How does this affect Flash size, RAM usage, execution time, and safety?"
- "Can this be interrupted? What happens if it is?"
- "What is the worst-case behavior?"

**Final rule:** *"If you can't guarantee it completes in 95% of your sample period, don't put it in the critical path."*

---

## **Critical Info**

**Added Definitions for Clarity:**

- **ISR (Interrupt Service Routine):** Hardware-triggered function that runs immediately when peripherals need attention (UART received byte, timer overflow). **Must be fast (5-50μs).**

- **DMA (Direct Memory Access):** Hardware controller that moves data without CPU (ADC samples → RAM, RAM → DAC). **Conflicts with CPU for bus access.**

- **HardFault:** ARM exception #3 triggered by illegal operations (bad memory access, divide-by-zero). **System crashes unless handler recovers.**

- **Watchdog:** Timer that resets MCU if not fed periodically. **Catches stuck code.** In critical systems, watchdog feed is **only allowed in main loop**.

- **PID (Proportional-Integral-Derivative):** Control algorithm for motor/temperature control. **Must run at fixed frequency** (e.g., 1000Hz). Each term reacts to different error characteristics.

- **DSP (Digital Signal Processing):** Math-intensive algorithms (filters, FFT) processing real-time data. **Must be optimized** (fixed-point, SIMD, loop unroll).

- **MISRA C:** Safety standard banning dangerous practices (malloc, goto, bit fields). **Required for automotive/medical certification.**

- **Linker Script:** `.ld` file telling linker where to put sections. **Without it, chaos.**

- **Startup Code:** Runs before `main()`. **Copies .data, zeros .bss, calls SystemInit().** Usually in `startup_stm32f103.s`.

- **Flash Wait States:** Flash is slower than CPU. At 72MHz, CPU must insert **5 wait cycles** per Flash read. **Code in RAM runs 5x faster.**

- **Bus Fault:** Exception when accessing invalid address (0x00000000, unmapped peripheral). **Usually means bug in pointer arithmetic.**

- **Alignment Fault:** Accessing 32-bit value at non-4-byte address. **Cortex-M0 crashes, M3/M4 is slow.**

- **Atomic Operation:** Single instruction that cannot be interrupted. **Foundation of mutexes, semaphores, hardware control.**

- **Race Condition:** When two contexts (main + ISR) access same variable and result depends on timing. **Heisenbug: disappears when you debug.**

- **Critical Section:** Code between `__disable_irq()` and `__enable_irq()`. **Must be short (< 10μs)** or system becomes unresponsive.

- **Context Switch:** Saving one task's state (registers, SP) and restoring another's. **RTOS does this 1000x/second.**

- **Stack Overflow:** Writing past stack bottom. **Corrupts globals, causes HardFault.** Detect with canary values (0xDEADBEEF).

- **Memory Leak:** Forgetting to `free()` → heap exhaustion. **In embedded, permanent crash.**

- **Segmentation Fault:** x86 term for accessing memory outside your segment. **Doesn't exist in ARM flat model.**

- **Literal Pool:** Constants embedded in code stream. **Accessed via PC-relative load: `LDR R0, [PC, #offset]`.**

- **Thumb-2 Instruction Set:** Mixed 16/32-bit instructions. **Denser code than ARM32, 25-30% smaller.**

- **Pipeline Stall:** CPU waiting for memory or previous instruction. **Causes hidden cycle cost.**

- **Cache Miss:** Data not in cache → must fetch from RAM (10-20 cycles). **Kills determinism.**

- **Write-Back Cache:** CPU writes to cache, not RAM immediately. **Confusing when debugging hardware registers.**

- **Memory Barrier:** Assembly instruction that forces all memory operations to complete. **Needed after reconfiguring MPU or cache.**

---

**Final addition:** **Toolchain Flow Diagram**

```
main.c → (cpp) → main.i → (cc1) → main.s → (as) → main.o → (ld) → firmware.elf → (objcopy) → firmware.hex
   ↓        ↓        ↓        ↓         ↓         ↓           ↓            ↓
Preproc  Compiler  Assembler  Linker   Strip    ELF to HEX
(5000 lines)                    ↳ Linker Script (.ld) → Memory layout
```

**What each step does:**
- **cpp:** Handles `#define`, `#include` (creates 5000-line expanded source)
- **cc1:** Parses C, generates assembly
- **as:** Converts assembly to machine code with symbols
- **ld:** Combines .o files, resolves symbols, uses linker script
- **objcopy:** Strips symbols, converts to Intel HEX for programming
