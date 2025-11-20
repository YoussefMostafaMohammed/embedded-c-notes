# **Memory-Architecture**

### **Table of Contents**
1. **[The Complete Memory Map](#1-the-complete-memory-map)**
2. **[Flash vs RAM Characteristics](#2-flash-vs-ram-characteristics)**
3. **[Text Segment (.text)](#3-text-segment-text)**
   - 3.1. [What's Inside .text?](#31-whats-inside-text)
4. **[Read-Only Data (.rodata)](#4-read-only-data-rodata)**
   - 4.1. [Why 720 Bytes for 360 Entries?](#41-why-720-bytes-for-360-entries)
5. **[Initialized Data (.data)](#5-initialized-data-data)**
   - 5.1. [The 12-Byte Cost of Manual Init](#51-the-12-byte-cost-of-manual-init)
6. **[Zero Section (.bss)](#6-zero-section-bss)**
7. **[Stack Segment](#7-stack-segment)**
8. **[Heap Segment](#8-heap-segment)**
9. **[What If Not Scenarios](#9-what-if-not-scenarios)**
   - 9.4. [What If No PSP? (Deep Dive)](#94-what-if-no-psp-deep-dive)

---

### **1. The Complete Memory Map**

```
╔═══════════════════════════════════════════════════════════════╗
║                    FLASH (512KB @ 0x08000000)                 ║
║  0x08000000 ─┬─ .isr_vector (512 bytes)                      ║
║              │   - Initial SP value                          ║
║              │   - Reset handler address                     ║
║              ├─ .text (150KB)                                ║
║              │   - All executable code                       ║
║              ├─ .rodata (20KB)                               ║
║              │   - const tables, strings                     ║
║              └─ .data_init (4KB)                             ║
║                  - Initial values for .data                    ║
╠═══════════════════════════════════════════════════════════════╣
║                    RAM (64KB @ 0x20000000)                    ║
║  0x20000000 ─┬─ .data (2KB)                                  ║
║              │   - global_init = 50                          ║
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

### **2. Flash vs RAM Characteristics**

| Property | **FLASH** | **RAM** | **Why It Matters** |
|----------|-----------|---------|-------------------|
| **Persistence** | Retained on power loss | Lost on power loss | Code/constants must live in Flash |
| **Write cycles** | 10,000-100,000 | Unlimited | Can't modify Flash frequently |
| **Write speed** | 20μs per 16-bit word | 1-2 cycles (ns) | RAM for variables, Flash for code |
| **Cost** | $0.15/KB | $0.05/KB | Saving Flash saves money |
| **Size** | 512KB typical | 64KB typical | RAM is precious |

---

### **3. Text Segment (.text)**

**What lives here:**
```assembly
08000100 <Reset_Handler>:
 08000100:   4b02        ldr r3, [pc, #8]    ; Load __data_init address
 08000102:   4a03        ldr r2, [pc, #12]   ; Load __data_start address
 08000104:   4c04        ldr r4, [pc, #16]   ; Load __data_size
 08000106:   f000 f804   bl  0x08001112      ; Call memcpy
 ; ... more code ...
```

**Characteristics:**
- **Read-only** (hardware enforced - write triggers fault)
- **Execute-only** on modern MCUs (can't read as data)
- **Shared** between processes in OS environment

**Why separate from data?** Harvard architecture CPUs have separate buses for code vs data. Placing `.text` in Flash allows **simultaneous fetch and data access**.

#### **3.1. What's Inside .text?**

The `.text` section contains pure machine code—your compiled functions. But it's more than just instructions:

- **Function bodies**: Every compiled C function becomes binary opcodes
- **Compiler-generated code**: Startup routines (`__main`), exception handlers
- **Constant data disguised as code**: Small immediate values embedded directly in instructions (e.g., `MOV R0, #42` stores 42 inside the opcode itself)
- **Literal pools**: Groups of constants (addresses, large numbers) embedded in the code stream for quick access via PC-relative loads

**Example of literal pool:**
```assembly
08000100 <func>:
 08000100:   f04f 3342   mov.w   r3, #12850      ; Load immediate
 08000104:   f2ad 4298   sub.w   r2, sp, #4000   ; Large offset
 08000108:   4b01        ldr     r3, [pc, #4]    ; Load from literal pool
0800010c:   4770        bx      lr
0800010e:   bf00        nop
08000110:   12345678    .word   0x12345678        ; Literal pool constant
```

The constant `0x12345678` is stored **inside .text** at address `0x08000110`, not in `.rodata`, because it's quicker to load via PC-relative addressing.

---

### **4. Read-Only Data (.rodata)**

```c
const int sine_lookup[360] = {0, 17, 35, ...};
const char *msg = "Error: Overvoltage";
```

**Memory dump:**
```
08001200:  0000 1100 2300 3500 4700 5a00 6d00 7f00  ....#.G.Z.m...
08001210:  9100 a400 b700 ca00 dd00 f000 0301 1701  ...............
```

**Why not in RAM?** 
- 360-entry lookup table = 720 bytes
- RAM is 64KB total
- **1% of RAM** saved by using Flash

**What if .rodata didn't exist?**
```c
// Without .rodata, you'd have:
int *sine_lookup = malloc(720);  // Runtime allocation
// Problems:
// 1. Wastes RAM (720 bytes)
// 2. Requires initialization code
// 3. Non-deterministic (malloc may fail)
```

#### **4.1. Why 720 Bytes for 360 Entries?**

You're right to question this! A standard `int` is 4 bytes, so 360 entries would be **1,440 bytes**. The example uses `int16_t` or `short` (2 bytes per entry), likely because the sine values fit within a 16-bit range (e.g., Q15 fixed-point format). This is a **conscious optimization**: using `int16_t` instead of `int` **saves 720 bytes of Flash** without losing precision for the application.

**Corrected calculation:**
```c
const int16_t sine_lookup[360] = {0, 17, 35, ...};  // 360 × 2 = 720 bytes
```

---

### **5. Initialized Data (.data)**

```c
int global_init = 50;  // Goes to .data
```

**The dual-storage mechanism:**
```
Flash (source):
0x08004000:  32 00 00 00  ; The value 50

RAM (destination):
0x20000000:  ?? ?? ?? ??  ; Uninitialized on power-on
```

**Startup code copies:**
```assembly
; In Reset_Handler
LDR R0, =0x20000000  ; Destination: __data_start
LDR R1, =0x08004000  ; Source: __data_init
LDR R2, =4000        ; Size: __data_size = 1000 variables × 4 bytes
BL memcpy            ; Copies from Flash to RAM
```

**What if .data didn't exist?**
```c
// Manual init:
int global_init;  // In .bss (zero)

void init(void) {
    global_init = 50;  // Generates 12 bytes per variable
}
// For 1000 variables: 12,000 bytes of Flash wasted
```

#### **5.1. The 12-Byte Cost of Manual Init**

The line `global_init = 50;` generates three 32-bit (4-byte) instructions in Flash:

```assembly
LDR R0, =0x20000000  ; Load address of global_init (4 bytes)
LDR R1, =50          ; Load immediate value 50 (4 bytes)
STR R1, [R0]         ; Store value to RAM address (4 bytes)
```

**Total: 12 bytes per variable.**  
With `.data`, the startup code uses a **single `memcpy` loop** (about 10 bytes total) to initialize all variables, regardless of count. For 1,000 variables, `.data` saves **11,990 bytes of Flash**.

---

### **6. Zero Section (.bss)**

```c
int global_uninit;      // In .bss
static int file_static; // In .bss
```

**Why zero?**
1. **Flash savings:** Don't store 1000 zeros in Flash
2. **Standard requirement:** C mandates static storage starts zeroed
3. **Determinism:** No random values on startup

**Startup zeroing:**
```assembly
LDR R0, =0x20000400  ; __bss_start (after .data)
LDR R1, =0x20002800  ; __bss_end
MOV R2, #0
.Lloop:
    STR R2, [R0]      ; Write zero
    ADD R0, R0, #4    ; Next word
    CMP R0, R1
    BNE .Lloop        ; Loop until end
; Total: ~10 bytes of code, regardless of .bss size
```

**What if .bss didn't exist?**
```c
int global_uninit = 0;  // In .data
// Flash stores: 00 00 00 00 (4 bytes wasted per variable)
// For 1000 variables: 4KB of Flash wasted storing zeros
```

---

### **7. Stack Segment**

**Stack operations:**
```assembly
PUSH {R0, R1}       ; SP = SP - 8, store R0,R1 at [SP]
; PUSH = PuSH registers onto stack
; Decrements SP first, then stores

POP {R0, R1}        ; Load R0,R1 from [SP], SP = SP + 8
; POP = POP registers from stack
; Loads first, then increments SP
```

**Why grow downward?**
```
RAM:
0x2000FFFC:  Stack top (SP starts here)
            ↓ Grows downward (decrement)
0x2000F000:  Stack limit
0x2000EFFC:  Magic number for overflow detection
0x2000E000:  Heap starts
            ↑ Grows upward
0x20000000:  RAM bottom
```

**Visual stack frame:**
```c
void func(void) {
    int a = 10;        // SP-4
    int b = 20;        // SP-8
    char c[4];         // SP-12
    // SP points to 0x2000FFF4
}
```

```
Stack after func() called:
0x2000FFFC:  (return address) LR value
0x2000FFF8:  (saved R4)
0x2000FFF4:  [a = 10]        ← SP points here
0x2000FFF0:  [b = 20]
0x2000FFEC:  [c[0] c[1] c[2] c[3]]
```

---

### **8. Heap Segment**

**Fragmentation visualization:**
```
Initial: [Free: 61440]

After p1 = malloc(100):
[Used:100][Free:61340]

After p2 = malloc(200):
[Used:100][Used:200][Free:61140]

After free(p1):
[Free:100][Used:200][Free:61140]

After p3 = malloc(150):
[Free:100][Used:150][Free:50][Used:200][Free:60940]
            ↑ Splits block, creates small fragment
```

**Why malloc time varies:**
- **Best case (no fragmentation):** 1μs (first block fits)
- **Worst case (high fragmentation):** 5ms (searches 1000 blocks, merges fragments)
- **Deterministic systems:** **Ban malloc() after initialization**

---

### **9. What If Not Scenarios**

#### **What if .data didn't exist?**
```c
// Without .data (manual init):
int config = 50;  // Becomes .bss (zero)

void init_all(void) {
    config = 50;   // 12 bytes of code
    // ... repeat for 1000 variables ...
}
// Result: 12KB Flash waste, 1ms longer startup
```

#### **What if .bss didn't exist?**
```c
// All variables in .data:
int global_uninit = 0;  // Flash stores 0x00000000
// For 1000 variables: 4KB Flash waste
// MCU choice: Must buy 128KB Flash instead of 64KB = $0.50 more per unit
```

#### **What if .rodata didn't exist?**
```c
// Strings in RAM:
const char *msg = "Error";  // Pointer in RAM, string in Flash
// Waste: 4 bytes RAM per string pointer
// For 100 strings: 400 bytes RAM wasted
```

#### **What if stack grew upward?**
```
RAM:
0x20000000:  Stack start
            ↑ Grows up
0x20001000:  .bss and .data globals
0x20002000:  Stack limit

Overflow: Overwrites globals silently → untraceable bugs
```

#### **What if no PSP?**
```c
// Task overflows stack:
void task(void) {
    char buf[10000];  // Too big
}

// Result: Corrupts MSP (OS stack) → HardFault → System crash
// No way to recover or kill just the task
```

#### **9.4. What If No PSP? (Deep Dive)**

**PSP (Process Stack Pointer)** is a second stack pointer in ARM Cortex-M that enables **task isolation** in RTOS environments.

**Without PSP (single MSP only):**
```
Memory Layout with ONE stack:
0x20010000:  MSP top (OS + all tasks share this)
            ↓ Grows down
            [Task1 variables]
            [Task2 variables]  ← Overflow here corrupts OS
            [OS kernel data]
0x20000000:  MSP bottom

When Task2 overflows: Corrupts OS heap → HardFault → Total system crash
No way to identify which task failed
```

**With PSP (dual stacks):**
```
Memory Layout with PSP:
0x20010000:  MSP top (OS kernel only, protected)
            ↓ Grows down
            [OS kernel data]
0x20008000:  MSP limit (HW-checked)
            ↑ Task area
            [Task1 PSP stack] (separate SP)
            [Task2 PSP stack] (separate SP)
0x20000000:  RAM bottom

When Task2 overflows: Corrupts Task2's own data
OS can detect PSP fault → Kill ONLY Task2 → System keeps running
```

**Why PSP matters:**
- **Fault isolation**: A buggy user task can't crash the OS
- **Deterministic context switch**: `PendSV` uses PSP automatically
- **Safety**: Enables watchdog per-task recovery

**Real-world consequence:**
> Without PSP, a memory leak in a low-priority sensor task will eventually crash the entire drone's flight controller. With PSP, the OS can restart just that task and log the error while the drone keeps flying.

