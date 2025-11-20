# **C-Keywords**

### **Table of Contents**
1. **[const - The Read-Only Qualifier](#1-const---the-read-only-qualifier)**
2. **[static - Internal Linkage](#2-static---internal-linkage)**
3. **[register - The Optimization Hint](#3-register---the-optimization-hint)**
4. **[volatile - The Nuclear Option](#4-volatile---the-nuclear-option)**
   - 4.1. [volatile Deep Dive: Compiler vs Hardware](#41-volatile-deep-dive-compiler-vs-hardware)
5. **[extern - Declaration vs Definition](#5-extern---declaration-vs-definition)**
6. **[Race Conditions & Atomicity](#6-race-conditions--atomicity)**
   - 6.1. [Step-by-Step Race Condition Breakdown](#61-step-by-step-race-condition-breakdown)

---

### **1. const - The Read-Only Qualifier**

```c
// Global const → TRUE constant in Flash
const int MAX_SPEED = 100;  // .rodata at 0x08001200

// Local const → Stack (can be "hacked")
void func(void) {
    const int secret = 42;    // Stack at SP-4
    int *hacker = (int*)&secret;
    *hacker = 0;              // Works! secret is now 0
}
```

**Why the difference?**
```assembly
; Global const (hard fault on write)
LDR R0, =0x08001200  ; Flash address
LDR R1, [R0]         ; Read OK
STR R1, [R0]         ; HardFault! Flash is read-only

; Local const (RAM write allowed)
LDR R0, [SP, #4]     ; Stack address
LDR R1, [R0]         ; Read OK
STR R1, [R0]         ; Write OK (no hardware fault)
```

**Why not put locals in .rodata?**
```c
void func(void) {
    const int x = 10;  // If in Flash at 0x08001200
    // But you call func() 100 times
    // Each call would need its own copy at different address
    // IMPOSSIBLE - stack solves this with reuse
}
```

---

### **2. static - Internal Linkage**

```c
// Static global: file-local, permanent
static int file_counter = 0;  // .data, visible only in this .c file

// Static local: function-local, permanent
void counter(void) {
    static int call_count = 0;  // .data, retains value
    return ++call_count;
}
```

**Generated code for static local:**
```assembly
counter:
    LDR R0, =0x20000100  ; Fixed address in .data
    LDR R1, [R0]         ; Load current value
    ADD R1, R1, #1       ; Increment
    STR R1, [R0]         ; Store back
    BX LR                ; Return
```
**Key:** Static locals **don't use SP offset**. They have **fixed addresses** in .data/.bss like globals.

**Why not use globals?**
```c
// file1.c
static int temp;  // Symbol 'temp' not exported

// file2.c
static int temp;  // Different variable, no linker error
```
**Information hiding**: Prevents name collisions in large projects.

---

### **3. register - The Optimization Hint**

```c
void fast_loop(void) {
    register int i;  // Request: store i in CPU register
    for(i = 0; i < 1000; i++) {
        // Tight loop
    }
}
```

**What happens:**
- Compiler tries to allocate `i` to R0-R12
- If no register free, uses stack (auto variable)
- **Cannot take address**: `&i` produces compile error

**Why registers are faster:**
```assembly
; With register (1 cycle per increment)
ADD R5, R5, #1         ; Single register add

; With stack (3 cycles per increment)
LDR R5, [SP, #4]       ; 1. Load from stack
ADD R5, R5, #1         ; 2. Increment
STR R5, [SP, #4]       ; 3. Store back
```

**Why modern compilers ignore `register`:**
- Register allocators are smarter than manual hints
- At -O2, `int i` often becomes register anyway
- **Better to use `restrict` for performance**

**Global register variables (DANGEROUS):**
```c
register int foo asm("r5");  // Force R5 usage
void func1(void) { foo = 10; }
void func2(void) { /* uses R5 for other purpose */ }
```
**Result:** `func2` corrupts `func1`'s data. **Never use this.**

---

### **4. volatile - The Nuclear Option**

**When mandatory:**
```c
volatile uint32_t *status_reg = (uint32_t*)0x40020000;

void wait_for_flag(void) {
    while(*status_reg & 0x01) {
        // Wait for hardware to clear bit
    }
}
```

**What `volatile` prevents:**
```assembly
// WITHOUT volatile (compiler optimizes away):
wait_for_flag:
    LDR R0, =0x40020000
    LDR R1, [R0]         ; Read ONCE
loop:
    TST R1, #1
    BNE loop             ; Infinite loop if bit was set!
    ; Never re-reads hardware register!

// WITH volatile (re-reads every time):
wait_for_flag:
    LDR R0, =0x40020000
loop:
    LDR R1, [R0]         ; Read EVERY iteration
    TST R1, #1
    BNE loop             ; Correctly responds to hardware
```

#### **4.1. volatile Deep Dive: Compiler vs Hardware**

**The Core Problem: Compiler Assumptions**

The compiler assumes memory is passive—it only changes when *your code* writes to it. This allows aggressive optimizations:

- **Register caching**: Store variable in CPU register, avoid memory reads
- **Dead store elimination**: Remove "useless" writes
- **Instruction reordering**: Reorder for pipeline efficiency

**These assumptions are FALSE for:**
- **Hardware registers**: Change autonomously (ADC conversion complete, UART byte received)
- **ISR-shared variables**: Modified by interrupt context
- **Multi-core memory**: Changed by another CPU core

**What `volatile` Forces: Memory Access Every Time**

`volatile` is a **compiler directive**, not a hardware feature. It tells the compiler: **"Never optimize away accesses to this location."**

**Real disaster without volatile:**
```c
// DMA status register
#define DMA_DONE (*(uint32_t*)0x40020000)

void start_dma(void) {
    DMA_CTRL = 0x01;  // Start transfer
    while(!DMA_DONE); // Wait for completion
}
```

**Compiler's view:**
1. `DMA_CTRL = 0x01;` - Write to memory
2. `while(!DMA_DONE);` - Read memory once, loop forever
3. "No one writes to DMA_DONE in this function, so its value never changes"

**Optimized assembly (BROKEN):**
```assembly
LDR R0, =0x40020004
MOV R1, #1
STR R1, [R0]          ; Start DMA
LDR R0, =0x40020000
LDR R1, [R0]          ; Read DMA_DONE ONCE
loop:
    CMP R1, #0
    BEQ loop          ; Infinite loop! Never checks hardware again
```

**DMA completes, sets bit, but CPU never sees it.** Your system locks up.

**With volatile:**
```c
volatile uint32_t *dma_done = (uint32_t*)0x40020000;
while(!(*dma_done));  // Compiler generates LDR every iteration
```

**Correct assembly:**
```assembly
loop:
    LDR R1, [R0]      ; Re-read from hardware EVERY time
    CMP R1, #0
    BEQ loop          ; Hardware-responsive
```

**volatile vs. Atomic: The Deadly Confusion**

```c
volatile int counter = 0;

// ISR increments counter
void ISR_Handler(void) {
    counter++;  // ISR: Read-modify-write (3 instructions)
}

// Main reads counter
void main(void) {
    if(counter == 10) {  // Main: Read (1 instruction)
        // Do something
    }
}
```

**The trap**: `volatile` ensures `main()` reads the latest `counter` value from memory, not a cached register. **But the increment itself is NOT atomic.**

**Without atomic increment:**
```assembly
; ISR fires in the middle of main's read-modify-write
main:
    LDR R0, =counter
    LDR R1, [R0]      ; Read counter (R1 = 10)
                      ; ← ISR INTERRUPTS HERE
    ADD R1, R1, #1    ; R1 = 11
    STR R1, [R0]      ; Write back (counter = 11)
                      ; ISR returns
                      ; main continues with R1 = 11
                      ; But ISR also incremented!
                      ; counter is now 12, but main thinks it's 11

ISR:
    LDR R0, =counter
    LDR R1, [R0]      ; Read counter (R1 = 10)
    ADD R1, R1, #1    ; R1 = 11
    STR R1, [R0]      ; Write back (counter = 11)
    BX LR             ; Return
```

**Result**: Lost increment. `counter` should be 12, but is 11.

**Key lesson**: `volatile` prevents **compiler caching** but does NOT prevent **hardware race conditions**. For atomicity, you need hardware support (LDREX/STREX) or disable interrupts.

---

### **5. extern - Declaration vs Definition**

**The One Definition Rule:**
```c
// file1.c - DEFINITION (memory allocated)
int shared_var = 10;  // Symbol: D shared_var

// file2.c - DECLARATION (no memory)
extern int shared_var; // Symbol: U shared_var

// Linker resolves U to point to D
```

**The linker process:**
1. **Compile file1.o:** Symbol table contains `00000000 D shared_var`
2. **Compile file2.o:** Symbol table contains `         U shared_var`
3. **Link:** Replace all `U` references with address of `D`

**Why multiple declarations allowed:**
```c
// Every .c file that uses shared_var can declare it
// Linker ensures only ONE definition exists
// This is how headers work
```

**What if you forget `extern`?**
```c
// file2.c
int shared_var;  // WRONG! Creates new definition
// Linker error: multiple definition of 'shared_var'
```

---

### **6. Race Conditions & Atomicity**

**Race condition example:**
```c
int x;  // Shared between main and ISR

void main(void) {
    x = 0;
    while(x == 0) {  // 3-instruction window
        // ISR fires here
    }
}

void ISR(void) {
    x = 10;  // Corrupts main's logic
}
```

**Assembly showing the race:**
```assembly
main:
    LDR R1, =x
    STR R0, [R1]          ; x = 0
    
loop:                     ; ISR can fire HERE
    LDR R2, [R1]          ; Read x into R2
    CMP R2, #0            ; Compare
    BEQ loop              ; Branch if zero

ISR:
    STR R0, [R1]          ; Write x = 10 (in memory)
    BX LR                 ; Return

; Problem: R2 still contains 0 (stale register value)
; Loop continues forever even though x is 10 in memory!
```

#### **6.1. Step-by-Step Race Condition Breakdown**

Let's trace **exactly** what happens at the instruction level:

| Time | Main Thread | ISR Thread | CPU State |
|------|-------------|------------|-----------|
| **t0** | `LDR R1, =x` (R1 = 0x20000000) | (idle) | x = 0 in RAM |
| **t1** | `STR R0, [R1]` (x = 0) | (idle) | x = 0 in RAM |
| **t2** | `LDR R2, [R1]` (R2 = 0) | (idle) | **R2 = 0** (cached) |
| **t3** | **CMP R2, #0** → EQUAL | **ISR TRIGGERS!** | Main context saved to stack |
| **t4** | (suspended) | `LDR R1, =x` | ISR gets R1 = 0x20000000 |
| **t5** | (suspended) | `STR R0, [R1]` → **x = 10** in RAM | x is NOW 10 in memory |
| **t6** | (suspended) | `BX LR` (return) | ISR exits, restores main context |
| **t7** | `BEQ loop` (branch taken) | (idle) | **Main uses STALE R2 = 0!** |
| **t8** | Back to loop... | (idle) | **INFINITE LOOP** |

**The 3-instruction vulnerability window:**

```assembly
loop:
    LDR R2, [R1]          ; Instruction 1: Read
    ; ← ISR CAN FIRE HERE (after read, before compare)
    CMP R2, #0            ; Instruction 2: Compare
    ; ← OR HERE (after compare, before branch)
    BEQ loop              ; Instruction 3: Branch
    ; ← OR HERE (after branch, before loop restart)
```

**Why it's deadly:** The ISR correctly writes `x = 10` to RAM, but **main's R2 register** still holds the old value (0). The `CMP` instruction compares **registers**, not memory. Main never re-reads memory, so it never sees the ISR's change.

**Atomic solution:**
```c
// Critical section
__disable_irq();          ; CPSID I (disable interrupts)
int temp = x;             ; Atomic read
if(temp == 0) {
    x = 1;                ; Atomic write
}
__enable_irq();           ; CPSIE I (enable interrupts)
```

**What atomic means:** **Single, indivisible instruction** that cannot be interrupted. ARM provides:
- `BSET`/`BCLR` for single bits
- `LDREX`/`STREX` for memory with lock
