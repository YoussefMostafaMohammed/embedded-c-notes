# **C-Keywords**

### **Table of Contents**
1. **[const - The Read-Only Qualifier](#1-const---the-read-only-qualifier)**
2. **[static - Internal Linkage](#2-static---internal-linkage)**
3. **[register - The Optimization Hint](#3-register---the-optimization-hint)**
4. **[volatile - The Nuclear Option](#4-volatile---the-nuclear-option)**
5. **[extern - Declaration vs Definition](#5-extern---declaration-vs-definition)**
6. **[Race Conditions & Atomicity](#6-race-conditions--atomicity)**

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

**Why volatile isn't enough for race conditions:**
```c
volatile int flag;

void broken(void) {
    flag = 0;
    if(flag == 0) {  // Read-modify-write race
        // ISR can fire here
        flag = 1;    // Overwrites ISR's change
    }
}
```

**Atomic operations required:**
```assembly
LDREX R0, [R1]       ; Load Exclusive (hardware lock)
STREX R2, R0, [R1]   ; Store Exclusive (fails if lock broken)
; LDREX/STREX = Load/Store Exclusive
```

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
