# 3 - Embedded-C-Notes

## **Table of Contents**

1. [Why Determinism is Life-or-Death in Embedded Systems](#1-why-determinism-is-life-or-death-in-embedded-systems)
2. [Memory Initialization and Storage Strategies](#2-memory-initialization-and-storage-strategies)
   - 2.1. [Lookup Tables and Flash Storage](#21-lookup-tables-and-flash-storage)
   - 2.2. [The "Auto" Storage Class](#22-the-auto-storage-class)
   - 2.3. [The Hidden Cost of Manual Initialization](#23-the-hidden-cost-of-manual-initialization)
   - 2.4. [`.bss` Zero-Initialization: The Free Lunch](#24-bss-zero-initialization-the-free-lunch)
   - 2.5. [Direct Memory Access Patterns](#25-direct-memory-access-patterns)
3. [Stack Architecture and Protection](#3-stack-architecture-and-protection)
   - 3.1. [Stack Overflow Detection Mechanisms](#31-stack-overflow-detection-mechanisms)
   - 3.2. [Return Addresses and Call Frames](#32-return-addresses-and-call-frames)
   - 3.3. [Dual Stack Pointers (MSP and PSP)](#33-dual-stack-pointers-msp-and-psp)
4. [Heap Allocation and Fragmentation](#4-heap-allocation-and-fragmentation)
   - 4.1. [Understanding Heap Layout](#41-understanding-heap-layout)
   - 4.2. [Malloc's Global State and Reentrancy](#42-mallocs-global-state-and-reentrancy)
5. [The Build Pipeline: From Source to Silicon](#5-the-build-pipeline-from-source-to-silicon)
   - 5.1. [Compiler to Linker Flow](#51-compiler-to-linker-flow)
   - 5.2. [Linker Scripts: The Memory Blueprint](#52-linker-scripts-the-memory-blueprint)
   - 5.3. [Vector Tables and Startup Code](#53-vector-tables-and-startup-code)
6. [Language Features and Edge Cases](#6-language-features-and-edge-cases)
   - 6.1. [`constexpr` in Modern C++](#61-constexpr-in-modern-c)
   - 6.2. [`extern` in Function Scope](#62-extern-in-function-scope)
   - 6.3. [`volatile` and Race Conditions (Revisited)](#63-volatile-and-race-conditions-revisited)
7. [When to Modify Core System Components](#7-when-to-modify-core-system-components)
8. [Executive Summary: What If It All Goes Wrong?](#8-executive-summary-what-if-it-all-goes-wrong)

---

## **1. Why Determinism is Life-or-Death in Embedded Systems**

A **PID controller** calculates motor speeds 1000 times per second to keep a drone stable. The formula is:
```c
output = Kp * error + Ki * integral + Kd * derivative;
```

**"Runs late" means:** The calculation finishes at 1.2ms instead of 1.0ms.
- **Result:** Drone over-corrects, starts oscillating
- **Eventually:** Oscillation grows until drone flips and crashes

**Why determinism matters:**
- The control loop has **5ms total budget** (200Hz update rate)
- If `malloc()` sometimes takes 2ms, your PID calculation has only 3ms left
- **Sometimes it works, sometimes it crashes** → unpredictable product
- **Solution:** No dynamic allocation, fixed-point math, no function calls in loop

---

## **2. Memory Initialization and Storage Strategies**

### **2.1. Lookup Tables and Flash Storage**

```c
// Example: Sine wave for motor PWM smoothing
const uint16_t sine_lookup[360] = {
    0, 17, 35, 52, 70, 87, 105, 122, 140, 157, 175, 192, 210, 227, 245, 262,
    // ... 345 more values calculated by Python script
    65535, 65518, 65500, 65483, 65465, 65448, 65430, 65413, 65395, 65378, 65360, 65343
};

// Usage in code:
uint16_t pwm_value = sine_lookup[angle] >> 2;  // Fast, no floating point
```

**Memory Dump (objdump -s firmware.elf):**
```
Contents of section .rodata:
 08001200 00001100 23003500 47005a00 6d007f00  ....#.G.Z.m...
 08001210 9100a400 b700ca00 dd00f000 03011701  ................
```

**Storage layout:**
- **Address**: `0x08001200` (in Flash)
- **Size**: `360 * 2 = 720 bytes`
- **Access pattern**: `LDRH R0, [R1, R2, LSL #1]` (2-cycle read)
- **Why not calculate at runtime?** `sin()` takes 300 cycles, lookup takes 2 cycles = **150x faster**

### **2.2. The "Auto" Storage Class**

```c
void func(void) {
    int x;        // Equivalent to: auto int x;
    auto int y;   // Explicit but obsolete
}
```

**What "auto" means:**
- **Automatic storage duration** (created when function called, destroyed on return)
- **Default for all local variables** since 1978 (K&R C)
- **Never write it** - it's implied and makes you look like a time traveler

**C++11 hijacked the keyword:**
```cpp
auto x = 10;    // C++: type inference (x is int)
auto y = 3.14;  // C++: y is double
```
**This is completely unrelated to C's storage class.** In C, `auto` does nothing useful.

### **2.3. The Hidden Cost of Manual Initialization**

```c
// Without .data (manual init):
int config;  // In .bss (zero by default)

void my_init(void) {
    config = 50;  // This line generates code!
}
```

**Assembly generated:**
```assembly
my_init:
    LDR R0, =0x20000000  ; Address of config
    LDR R1, =50          ; Value 50
    STR R1, [R0]         ; Store instruction (4 bytes Flash)
    BX LR
```

**Flash waste per variable:**
- **Instructions**: `LDR LDR STR BX` = **8 bytes Flash**
- **For 1000 variables**: 1000 × 8 = **8000 bytes wasted**

**With .data:**
```assembly
startup:
    ; One loop for ALL variables
    LDR R0, =__data_start
    LDR R1, =__data_init
    LDR R2, =__data_size
    SUB R2, R2, R0        ; Calculate size
    BL memcpy             ; Single call, 20 bytes total code
```
**Savings**: 8000 bytes → 20 bytes = **400x reduction**

### **2.4. `.bss` Zero-Initialization: The Free Lunch**

```c
int global_uninit;  // Goes to .bss
```

**Why "no storage" but "stored zeros"?**

The **.bss section itself occupies RAM space** (64 bytes for that variable).
But **no Flash space** is used to store the value 0.

**Startup code pseudocode:**
```c
void startup(void) {
    extern char __bss_start, __bss_end;
    memset(&__bss_start, 0, &__bss_end - &__bss_start);
    // Equivalent to: for(p = 0x20000000; p < 0x20000040; p++) *p = 0;
}
```

**Tradeoff:**
- **RAM**: Yes, uses 4 bytes (must have space to store the variable)
- **Flash**: No, uses 0 bytes (doesn't store the literal `0x00000000`)
- **CPU**: 1 cycle per 4 bytes to zero

**What if .bss didn't exist and we used .data?**
```c
int global_uninit = 0;  // In .data
```
**Flash would contain**: `00 00 00 00` at some address → **4 bytes wasted** (times 1000 variables = 4KB wasted)

**Why not random number?** C standard mandates static storage has **deterministic** initial state. Zero is the logical neutral value.

### **2.5. Direct Memory Access Patterns**

```c
const char *msg = "Error";
```

**Diagram: Two Cases**

#### **Case 1: msg in .data (BAD)**
```
Flash at 0x08001234:
  45 72 72 6F 72 00  ; "Error\0" stored here

RAM at 0x20000000:
  08 00 12 34        ; msg pointer lives here (4 bytes RAM)

Code:
  LDR R0, =msg       ; R0 = 0x20000000 (load pointer from RAM)
  LDR R0, [R0]       ; R0 = 0x08001234 (dereference)
  BL printf
```
**2 loads, 2 cycles wasted**

#### **Case 2: msg in .rodata (GOOD)**
```
Flash at 0x08001234:
  45 72 72 6F 72 00  ; "Error\0" stored here
  ; No pointer stored - address is IMMEDIATE

Code:
  LDR R0, =0x08001234  ; R0 = immediate address (literal pool)
  BL printf
```
**1 load, 1 cycle**

**How the address is immediate:**
The compiler places the address in a **literal pool** right next to the code:
```assembly
.text:
  LDR R0, [PC, #8]  ; Load from PC+8 (relative)
  BL printf
  NOP
.literal_pool:
  .word 0x08001234  ; The address itself, in Flash
```

**The "pointer" is in the instruction stream, not RAM.**

---

## **3. Stack Architecture and Protection**

### **3.1. Stack Overflow Detection Mechanisms**

#### **Visual Detection Mechanism (Downward Growth)**

**Downward Growth (ARM):**
```
RAM Layout:
0x2000FFFC ─┐
            │ Stack starts here (SP)
            │ Grows downward
0x2000F000 ─┼─ Stack Limit
            │
            │ Unused (gap)
            │
            │ Heap starts here
0x20001000 ─┼─ Heap Start (with magic number 0xDEADBEEF)
            │ Grows upward
0x20000000 ─┤
```

**Detection code:**
```c
void check_overflow(void) {
    if (*(uint32_t*)0x20001000 != 0xDEADBEEF) {
        // Stack overflowed and overwrote magic number!
        while(1); // Halt system
    }
}
```
**Why easier to detect:** Stack and heap approach each other. You can place a **sentinel value** in between. When they collide, the sentinel changes.

**Upward Growth Hypothetical:**
```
0x2000FFFC ─┐
            │
            │ Heap (grows up)
            │
0x20001000 ─┼─ Stack starts here (grows UP)
            │
0x20000000 ─┤ Stack Limit
```
**Problem:** Stack overflow **immediately corrupts .bss and .data** below it. No gap, no sentinel. Your global variables silently change value. **Bug is untraceable.**

### **3.2. Return Addresses and Call Frames**

```c
void caller(void) {
    callee();  // Address: 0x08000120
    // Next line: 0x08000124
}

void callee(void) {
    return;  // Must jump back to 0x08000124
}
```

**What happens: The Hardware Stack**

```assembly
caller:
    BL callee      ; Branch with Link
                   ; 1. PC = 0x08000124 (next instruction)
                   ; 2. LR = PC (save return address)
                   ; 3. PC = address of callee
                   ; 4. Push LR onto stack (SP -= 4, [SP] = LR)

callee:
    PUSH {R4-R7}   ; Save registers
    POP {R4-R7}    ; Restore registers
    BX LR          ; Branch indirect to LR
                   ; 1. Pop LR from stack (if pushed)
                   ; 2. PC = LR
                   ; 3. Execution continues at 0x08000124
```

**The return address is the instruction address right after the `BL` call.** It's pushed on stack so nested calls work:
```c
main() calls → func1() calls → func2()
Stack: [main_ret] [func1_ret] [func2_ret]
       SP→                                    (grows down)
```

**If stack overflows and overwrites return address:**
```c
func2 returns → pops corrupted address (0xDEADBEEF) → PC = 0xDEADBEEF → HardFault
```

### **3.3. Dual Stack Pointers (MSP and PSP)**

ARM Cortex-M has **two stack pointers** for **operating system support**:

| Register | **Stands For** | **When Used** | **Why Two Exist** |
|----------|----------------|---------------|-------------------|
| **MSP** | Main Stack Pointer | - Reset handler<br>- NMI/HardFault<br>- Privileged code | **Never fails** - always valid for exceptions |
| **PSP** | Process Stack Pointer | - User threads<br>- Unprivileged code | **Isolation** - thread stack corruption can't crash OS |

**Boot sequence:**
```c
Reset_Handler:
    ; CPU starts with MSP = value from vector table[0]
    ; MSP = 0x20010000 (top of RAM)
    BL main                ; Use MSP for main() and ISRs
```

**With RTOS (FreeRTOS):**
```c
void vTask(void *pvParams) {
    // OS switches to PSP for this task
    // Stack is task->stack (separate from MSP)
    // If task overflows its PSP stack, MSP stack is still safe
    // OS can catch the fault and kill the task
}
```

**What if only one stack pointer?**
- **No OS support:** User code could overflow stack and corrupt kernel
- **No isolation:** One bug in thread crashes entire system
- **No safety:** Can't implement memory protection

---

## **4. Heap Allocation and Fragmentation**

### **4.1. Understanding Heap Layout**

```c
// Malloc's Internal Free List

// Malloc maintains a linked list of free blocks
struct free_block {
    size_t size;
    struct free_block *next;
};

// After p1=malloc(100), p2=malloc(200), free(p1):
Free list: head → [100 bytes free] → [61140 bytes free]
           (p1's freed block)      (remaining heap)

// Why malloc(150) doesn't reuse p1's block:
// - p1 is only 100 bytes (too small)
// - Malloc can't easily "grow" a free block
// - So it **splits** the next large block:

// Before allocating p3:
// [Used:100][Used:200][Free:61140]

// Malloc finds first fit: 61140 byte block
// Splits it: [150 used] + [61140-150-8(header) free]
// [Used:100][Used:200][Used:150][Free:60982]
```

**Your suggested layout is impossible** because:
- `malloc()` cannot move existing allocated blocks (`p2` is at fixed address)
- It can only carve pieces from remaining **contiguous** free space
- **Fragmentation is geometric:** After 100 cycles, you have 100 small holes

**What malloc can't do:**
```c
// You CANNOT compact heap like disk defragmentation
// Because pointers would become invalid:
int *p = malloc(4);  // p = 0x20001000
// If malloc moved data, p would point to garbage
```

### **4.2. Malloc's Global State and Reentrancy**

```c
// Malloc's global variables (simplified):
static struct free_block *free_list_head = NULL;
static size_t heap_remaining = 61440;

void *malloc(size_t size) {
    // Critical section start
    struct free_block *current = free_list_head;
    // ... search list, split blocks ...
    free_list_head = new_head;
    heap_remaining -= size;
    // Critical section end
}
```

**What "global state" means:** Any data **outside the function's stack frame** that persists between calls.

**Race condition:**
```c
// Main code:
malloc(100);  // Halfway through, just found a block

// ISR fires:
malloc(50);   // Corrupts the same free_list_head!
// Returns to main: free_list is now garbage
```

**Why non-reentrant:** Malloc doesn't disable interrupts during its critical section (would be too slow). It assumes it's in a single-threaded environment. RTOS replaces malloc with **thread-safe version** using mutexes.

---

## **5. The Build Pipeline: From Source to Silicon**

### **5.1. Compiler to Linker Flow**

```bash
main.c (source)
  ↓
cpp (preprocessor)
  ↓
main.i (preprocessed source - 5000 lines)
  ↓
cc1 (compiler proper)
  ↓
main.s (assembly code - human readable)
  ↓
as (assembler)
  ↓
main.o (relocatable object file - binary)
    - Contains: .text, .data, .bss sections (empty)
    - Symbol table: _main T 0x0, _my_var D 0x0
    - Relocation table: "call printf needs fixing"
  ↓
ld (linker) + linker script
  ↓
firmware.elf (linked executable)
    - Absolute addresses resolved
    - All sections merged
    - Debug symbols included
  ↓
objcopy
  ↓
firmware.hex (Intel HEX format)
    - :020000020800F2 (record type, address, data, checksum)
    - Plain text, no symbols
  ↓
STM32 Flash Programmer
    - Parses HEX
    - Writes Flash at addresses
```

**Object File Contents (objdump -x main.o)**
```
Sections:
Idx Name          Size      VMA       LMA       File off  Algn
  0 .text         00000034  00000000  00000000  00000034  2**1
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, CODE
  1 .data         00000004  00000000  00000000  00000068  2**2
                  CONTENTS, ALLOC, LOAD, DATA
  2 .bss          00000004  00000000  00000000  0000006c  2**2
                  ALLOC

SYMBOL TABLE:
00000000 l    df *ABS*  00000000 main.c
00000000 l    d  .text  00000000 .text
00000000 g     F .text  0000001c main
00000000 g     O .data  00000004 global_var
00000000         *UND*  00000000 printf  <<<<< UNDEFINED!
```

**The `printf` is UNDEFINED** - linker must find it in libc.a

### **5.2. Linker Scripts: The Memory Blueprint**

```ld
/* memory.ld - Bare metal ARM */
ENTRY(Reset_Handler)  /* First instruction to execute */

MEMORY
{
    /* Flash: 512KB at 0x08000000, readable and executable */
    FLASH (rx)  : ORIGIN = 0x08000000, LENGTH = 512K
    
    /* RAM: 64KB at 0x20000000, readable, writable, executable */
    RAM   (xrw) : ORIGIN = 0x20000000, LENGTH = 64K
    
    /* CCMRAM: 32KB core-coupled RAM (faster) */
    CCM   (rw)  : ORIGIN = 0x10000000, LENGTH = 32K
}

SECTIONS
{
    /* Vector table MUST be first */
    .isr_vector :
    {
        . = ALIGN(4);
        KEEP(*(.isr_vector))
    } >FLASH

    /* Startup code follows */
    .text :
    {
        . = ALIGN(4);
        *(.text)           /* Startup */
        *(.text*)          /* All other code */
        *(.rodata)         /* Read-only data */
        . = ALIGN(4);
    } >FLASH

    /* .data initial values (Flash copy) */
    _sidata = LOADADDR(.data);  /* Address in Flash */

    /* .data (RAM live copy) */
    .data : 
    {
        . = ALIGN(4);
        _sdata = .;         /* RAM start address */
        *(.data)
        _edata = .;         /* RAM end address */
    } >RAM AT> FLASH        /* AT> means "initial values stored in Flash" */

    /* .bss (zeroed RAM) */
    .bss :
    {
        _sbss = .;
        *(.bss)
        _ebss = .;
    } >RAM

    /* Heap (explicitly positioned) */
    _heap_start = _ebss;
    _heap_end = ORIGIN(RAM) + LENGTH(RAM) - 0x100; /* Keep 256B for stack */

    /* Stack at very end */
    _stack_start = ORIGIN(RAM) + LENGTH(RAM);
    _stack_end = _stack_start - 0x1000; /* 4KB stack */
}
```

**How Linker Uses Symbols**
```c
// In C code, you can reference linker symbols:
extern char _sbss;
extern char _ebss;

void my_clear_bss(void) {
    memset(&_sbss, 0, &_ebss - &_sbss);
}
```

**What linker does:**
1. **Resolves all symbols:** `_sbss = 0x20000100`, `_ebss = 0x20000400`
2. **Patch instructions:** `memset` call gets actual addresses
3. **Generate ELF segments:** Marks which parts go to Flash vs RAM

### **5.3. Vector Tables and Startup Code**

#### **What Happens When You Press Reset**

```
CPU Reset Line Asserted
    ↓
CPU loads SP from address 0x00000000
    ↓
CPU loads PC from address 0x00000004 (Reset vector)
    ↓
CPU starts executing at Reset_Handler
    ↓
Reset_Handler runs startup code
    ↓
startup code calls main()
    ↓
main() runs your program
```

#### **Vector Table Structure (ARM Cortex-M)**

```c
// From STM32 startup file (startup_stm32f103xb.s)
__Vectors:
    .word  _estack                    // 0x00: Initial SP value
    .word  Reset_Handler              // 0x04: Reset vector
    .word  NMI_Handler                // 0x08: Non-maskable interrupt
    .word  HardFault_Handler          // 0x0C: Hard fault
    .word  MemManage_Handler          // 0x10: Memory management fault
    .word  BusFault_Handler           // 0x14: Bus fault
    .word  UsageFault_Handler         // 0x18: Usage fault
    .word  0                          // 0x1C: Reserved
    .word  0                          // 0x20: Reserved
    .word  0                          // 0x24: Reserved
    .word  0                          // 0x28: Reserved
    .word  SVC_Handler                // 0x2C: SVCall
    .word  DebugMon_Handler           // 0x30: Debug monitor
    .word  0                          // 0x34: Reserved
    .word  PendSV_Handler             // 0x38: PendSV (OS context switch)
    .word  SysTick_Handler            // 0x3C: SysTick timer
    .word  WWDG_IRQHandler            // 0x40: Window watchdog
    .word  PVD_IRQHandler             // 0x44: PVD through EXTI detect
    // ... 60 more peripheral ISR vectors ...
```

**Memory location:** **Flash at 0x08000000** (or 0x00000000 if remapped)

#### **Startup Code: What It Actually Does**

```assembly
// From real STM32 startup file
Reset_Handler:
    ; Copy .data from Flash to RAM
    LDR R0, =_sidata      ; Flash source (initial values)
    LDR R1, =_sdata       ; RAM destination
    LDR R2, =_edata
    SUB R2, R2, R1        ; Size in bytes
    BL memcpy             ; Call memcpy (__ARM_memcpy)

    ; Zero .bss
    LDR R1, =_sbss        ; Start of .bss
    LDR R2, =_ebss        ; End of .bss
    MOV R0, #0            ; Fill value = 0
    BL memset             ; Call memset

    ; Call static constructors (C++ only)
    BL __libc_init_array

    ; Call main
    BL main

    ; If main returns, trap here
    B .
```

**How to See Them**
```bash
# 1. See vector table addresses
arm-none-eabi-objdump -s -j .isr_vector firmware.elf

# Output:
# 08000000 00000000 09000000 09020000 09020000
# 08000010 09020000 09020000 00000000 00000000

# 2. See startup code disassembly
arm-none-eabi-objdump -D -j .text firmware.elf | grep -A 20 Reset_Handler

# 3. See where sections end up
arm-none-eabi-nm firmware.elf | grep _sidata
# 08004000 D _sidata  (Flash address of initial values)

arm-none-eabi-nm firmware.elf | grep _sdata
# 20000000 D _sdata   (RAM address where they're copied)
```

---

## **6. Language Features and Edge Cases**

### **6.1. `constexpr` in Modern C++**

```cpp
constexpr int table[360] = { /* sine values */ };

void func() {
    constexpr int x = table[45];  // x is computed at compile time
    // x is literal 7071, not a memory access
}
```

**Does it save Flash?** **Only if you never take its address:**
```cpp
constexpr int x = 42;  // In Flash, no RAM

void bad() {
    const int *p = &x;   // OOPS! Now x must have an address
    // Compiler creates RAM copy (or Flash address)
}

// To guarantee Flash storage:
constinit int x = 42;  // C++20: FORCE constant initialization
```

**Misconception:** `constexpr` guarantees compile-time evaluation, not Flash placement. **C++ object model is complex** - it can create temporary objects in RAM.

### **6.2. `extern` in Function Scope (Legal but Stupid)**

```c
void func(void) {
    extern int global_x;  // Legal! Declares global variable
    global_x = 10;
}

// Same as:
extern int global_x;  // Usually at file scope
void func(void) {
    global_x = 10;
}
```

**Why it's allowed:** C grammar permits `extern` anywhere. It declares a name with **external linkage**.

**Why it's bad practice:**
- **Surprise factor:** Reader thinks it's local variable
- **Scope confusion:** Variable has global lifetime but block scope visibility
- **Maintenance nightmare:** Can't find declarations easily

**Rule:** Only use `extern` at **file scope** in header files.

### **6.3. `volatile` and Race Conditions (Revisited)**

You said:
> "write flag=0, read flag=0, isr fire and set flag=10, then isr finish, compare enter loob as the value read is zero at the start of the new loob it will reread the flag again and it will be 10 so it will works or what?"

#### **You Are Right - That Scenario Works**

```c
volatile int flag = 0;

void main(void) {
    flag = 0;
    while(flag == 0) {  // Each iteration RE-READS flag
        // Work
    }
}
```

**Assembly (with -O2):**
```assembly
loop:
    LDR R2, [R1]      ; Read flag (volatile forces this)
    CMP R2, #0        ; Compare
    BEQ loop          ; Branch if zero
```

**Timeline:**
```
Iteration 1: Read flag=0 → Compare 0==0 → Loop
ISR: flag=10
Iteration 2: Read flag=10 → Compare 10==0 → Exit
```
**This works correctly.** The **volatile** keyword ensures the read happens every time.

#### **The Race I Described (Different Code)**

```c
volatile int flag;

void broken(void) {
    flag = 0;
    while(1) {
        if(flag) break;  // Miscompiled!
    }
}
```

**Miscompiled assembly (buggy compiler):**
```assembly
broken:
    LDR R1, =flag
    MOV R0, #0
    STR R0, [R1]      ; flag = 0
    
    LDR R2, [R1]      ; Read ONCE (into R2)
loop:
    CMP R2, #0        ; Compare STALE VALUE in R2
    BEQ loop          ; Never re-reads memory!
```
**This is a compiler bug** that `volatile` prevents. But your example is **correctly compiled** and works as you described.

**Key takeaway:** Your mental model is correct. The race occurs only if the compiler optimizes away the read **inside the loop**, which `volatile` prevents.

---

## **7. When to Modify Core System Components**

### **Bootloader Development**

```c
// Bootloader at 0x08000000 with its own vectors
// Application at 0x08008000 with its own vectors

void boot_jump(uint32_t app_addr) {
    // 1. Disable interrupts
    __disable_irq();
    
    // 2. Remap vector table to application
    SCB->VTOR = app_addr;  // Now ISR addresses come from app's table
    
    // 3. Set new stack pointer
    __set_MSP(*(uint32_t*)app_addr);
    
    // 4. Jump to app's Reset_Handler
    void (*app_reset)(void) = (void*)(*(uint32_t*)(app_addr + 4));
    app_reset();
    
    // 5. Never returns
}

// After this, your application owns the MCU
```

### **OS Development (FreeRTOS Modification)**

```c
// FreeRTOS needs SysTick and PendSV vectors
// Original startup file points them to Default_Handler

// In FreeRTOSConfig.h:
#define vPortSVCHandler    SVC_Handler
#define xPortPendSVHandler PendSV_Handler
#define xPortSysTickHandler SysTick_Handler

// You must edit vector table in startup file:
.word vPortSVCHandler    // Was: Default_Handler
.word xPortPendSVHandler // Was: Default_Handler
.word xPortSysTickHandler // Was: Default_Handler
```

**If you don't change vectors correctly:**
- OS context switch never triggers → tasks freeze
- SysTick interrupt crashes to Default_Handler → system reset
- **Result:** OS doesn't work

---

## **8. Executive Summary: What If It All Goes Wrong?**

| Component | **What If Missing** | **Consequence** |
|-----------|---------------------|-----------------|
| **Vector table** | CPU doesn't know where to start | HardFault at boot |
| **Startup code** | .data not copied, .bss not zeroed | Globals contain garbage → instant bugs |
| **Linker script** | Linker doesn't know memory addresses | All addresses are 0x0, binary doesn't fit |
| **Flash .data copy** | In RAM only: values lost on power cycle | Settings revert, system forgets state |
| **Two stack pointers** | No OS support | Can't run multithreaded apps |
| **Atomic instructions** | Race conditions everywhere | Data corruption, crashes |
| **Alignment rules** | Unaligned access penalties | 50% performance loss, random hard faults |
| **Volatile** | Compiler erases hardware access | System hangs waiting for flags |

**Every piece exists because without it, embedded systems physically fail.**