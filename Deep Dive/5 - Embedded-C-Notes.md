# 5 - Embedded-C-Notes

## **Table of Contents**
1. [Memory Map for Our Example](#memory-map)
2. [Single Instruction Deep Dive: `my_init()` Function](#my-init-deep-dive)
   - [Instruction 1: `LDR R0, =config`](#instruction-1-ldr-r0-config)
   - [Instruction 2: `LDR R1, =50`](#instruction-2-ldr-r1-50)
   - [Instruction 3: `STR R1, [R0]`](#instruction-3-str-r1-r0)
   - [Instruction 4: `BX LR`](#instruction-4-bx-lr)
3. [Function Call Mechanism: The `memcpy` Example](#memcpy-example)
   - [Calling `memcpy` from Startup](#calling-memcpy)
   - [`BL` Instruction Deep Dive](#bl-instruction)
   - [Inside `memcpy` Function](#inside-memcpy)
4. [Performance Analysis: Manual vs Bulk Initialization](#performance-analysis)
   - [Byte Count Summary Table](#byte-count-table)
   - [Address Calculation for 1000 Variables](#address-calculation)
5. [Key Takeaway: The Cost of Abstraction](#key-takeaway)

---

## **<a name="memory-map"></a>Memory Map for Our Example**

```
FLASH (Code Storage)
0x08000000:  __Vectors (vector table)
0x08000100:  Reset_Handler (startup code)
0x08000200:  main() function
0x08000300:  my_init() function
0x08000310:  Literal pool (constants)
0x08000320:  memcpy() function (from libc)

RAM (Variable Storage)
0x20000000:  config (4 bytes)
0x20000004:  (next variable)
```

---

## **<a name="my-init-deep-dive"></a>Single Instruction Deep Dive: `my_init()` Function**

```c
// C code
void my_init(void) {
    extern int config;
    config = 50;
}

// Generated assembly (Thumb-2)
my_init:
    LDR R0, =config      ; @ 0x08000300
    LDR R1, =50          ; @ 0x08000302
    STR R1, [R0]         ; @ 0x08000304
    BX LR                ; @ 0x08000306
```

### **<a name="instruction-1-ldr-r0-config"></a>Instruction 1: `LDR R0, =config`**

**What it does:** Load the **address** of `config` into register R0.

**Encoding (4 bytes):**
```
Address: 0x08000300
Hex:     0x4802        ; This is the instruction
Binary:  01001000 00000010  (Thumb-2 encoding)

Instruction breakdown:
01 001 0 0 0 0 0 0 0 0 0 0 0 0 1 0
|  |  | | | | | | | | | | | | | |
|  |  | | | | | | | | | | | | | └─ Immediate offset (2)
|  |  | | | | | | | | | | | | └─── Reserved
|  |  | | | | | | | | | | | └───── PC-relative (yes)
|  |  | | | | | | | | | | └──────── Load to register
|  |  | | | | | | | | | └────────── Register destination (R0)
|  |  | | | | | | | | └──────────── Literal pool access
|  |  | | | | | | | └──────────────── Load (not store)
|  |  | | | | | | └────────────────── 32-bit (not 16)
|  |  | | | | | └──────────────────── Thumb mode
|  |  | | | | └────────────────────── Condition (always)
|  |  | | | └───────────────────────── Instruction group
|  |  | | └─────────────────────────── Opcode
|  |  | └───────────────────────────── Prefix
|  |  └──────────────────────────────── Prefix
|  └────────────────────────────────── Thumb marker
└───────────────────────────────────── Thumb marker
```

**Execution flow:**
1. **CPU fetches instruction** from 0x08000300-0x08000301 (2 bytes)
2. **CPU decodes:** "Load from PC + offset"
3. **CPU calculates source address:**
   ```
   PC = 0x08000300 + 4 = 0x08000304 (pipelined)
   Offset = 2 * 4 = 8
   Literal pool address = 0x08000304 + 8 = 0x0800030C
   ```
4. **CPU reads Flash** at 0x0800030C:
   ```
   Flash[0x0800030C] = 0x20000000  (address of config)
   ```
5. **CPU stores value** into R0: `R0 = 0x20000000`

**Visual register/memory:**
```
Flash:
[0x08000300] = 0x4802 (instruction)
[0x0800030C] = 0x20000000 (literal pool constant)

CPU Registers:
R0 = 0x20000000 (after execution)
PC = 0x08000302 (next instruction)
```

---

### **<a name="instruction-2-ldr-r1-50"></a>Instruction 2: `LDR R1, =50`**

**What it does:** Load the **value 50** into register R1.

**Encoding (4 bytes):**
```
Address: 0x08000302
Hex:     0x4903        ; Instruction encoding
```

**Execution flow:**
1. **CPU fetches** from 0x08000302-0x08000303
2. **PC = 0x08000302 + 4 = 0x08000306**
3. **Offset = 3 * 4 = 12**
4. **Source address = 0x08000306 + 12 = 0x08000312**
5. **CPU reads Flash** at 0x08000312:
   ```
   Flash[0x08000312] = 0x00000032  (50 in decimal = 0x32)
   ```
6. **CPU stores value** into R1: `R1 = 50`

**Final state:**
```
Flash:
[0x08000312] = 0x00000032

Registers:
R0 = 0x20000000
R1 = 0x00000032 (50)
PC = 0x08000304
```

---

### **<a name="instruction-3-str-r1-r0"></a>Instruction 3: `STR R1, [R0]`**

**What it does:** Store the **value in R1** to the **memory address in R0**.

**Encoding (2 bytes):**
```
Address: 0x08000304
Hex:     0x6001        ; 16-bit Thumb instruction
```

**Execution flow:**
1. **CPU fetches** 2-byte instruction from 0x08000304-0x08000305
2. **CPU decodes:** "Store R1 to address in R0"
3. **CPU performs bus write cycle:**
   ```
   Address bus = R0 = 0x20000000
   Data bus = R1 = 0x00000032
   Write enable = 1
   ```
4. **RAM at 0x20000000 receives**: `0x00000032`
5. **CPU updates**: `RAM[0x20000000] = 50`

**Visual memory change:**
```
BEFORE:
RAM: 0x20000000 = 0x00000000  (uninitialized)

AFTER:
RAM: 0x20000000 = 0x00000032  (now contains 50)
```

---

### **<a name="instruction-4-bx-lr"></a>Instruction 4: `BX LR`**

**What it does:** Branch to address stored in **Link Register (R14)** and **return from function**.

**Encoding (2 bytes):**
```
Address: 0x08000306
Hex:     0x4770        ; 16-bit instruction
```

**Execution flow:**
1. **CPU fetches** from 0x08000306-0x08000307
2. **CPU reads LR register:** `LR = 0x08000250` (return address)
3. **CPU loads PC:** `PC = LR = 0x08000250`
4. **Execution continues** at address 0x08000250 (in main())

**Where LR comes from:**
```assembly
// Whoever called my_init:
main:
    BL my_init   ; Branch with Link does:
                 ; 1. LR = address of next instruction (0x08000250)
                 ; 2. PC = my_init (0x08000300)
                 ; 3. my_init executes
                 ; 4. BX LR returns to 0x08000250
```

**Visual call stack:**
```
Main code:
0x0800024C:  BL my_init      ; LR set to 0x08000250
0x08000250:  <return here>   ; PC after BX LR
```

---

## **<a name="memcpy-example"></a>Function Call Mechanism: The `memcpy` Example**

### **<a name="calling-memcpy"></a>Calling `memcpy` from Startup**

```assembly
startup:
    LDR R0, =__data_start     ; 4 bytes @ 0x08000100
    LDR R1, =__data_init      ; 4 bytes @ 0x08000102
    LDR R2, =__data_size      ; 4 bytes @ 0x08000104
    BL memcpy                 ; 4 bytes @ 0x08000106
    ; ... continue ...
```

### **<a name="bl-instruction"></a>`BL` Instruction Deep Dive**

**Encoding:**
```
Address: 0x08000106
Hex:     0xF000F804  ; 32-bit Thumb-2 instruction
```

**What BL does:**
1. **Saves return address:**
   ```
   LR = PC + 4 = (0x08000106 + 4) = 0x0800010A
   ```
2. **Calculates jump address:**
   ```
   Offset = sign_extend(0x804 << 1) = 0x1008
   Target = PC + 4 + 0x1008 = 0x0800010A + 0x1008 = 0x08001112
   ```
3. **Jumps to memcpy:**
   ```
   PC = 0x08001112  (memcpy function address)
   ```

**Visual flow:**
```
Before BL:
PC = 0x08000106
LR = (unchanged)

After BL:
PC = 0x08001112  (inside memcpy)
LR = 0x0800010A  (address of next instruction)
```

### **<a name="inside-memcpy"></a>Inside `memcpy` Function**

```assembly
memcpy:  @ Located at 0x08001112
.Lloop:
    LDRB R3, [R1], #1    ; 2 bytes: Load byte from src, increment R1
    STRB R3, [R0], #1    ; 2 bytes: Store byte to dst, increment R0
    SUBS R2, R2, #1      ; 2 bytes: Decrement count
    BNE  .Lloop          ; 2 bytes: Branch if not zero
    BX   LR              ; 2 bytes: Return to caller
```

**How it uses our registers:**
```
Entry:
R0 = 0x20000000  (__data_start)
R1 = 0x08004000  (__data_init, Flash)
R2 = 0x00001000  (__data_size = 4096 bytes)

Loop:
1. LDRB R3, [R1]    ; R3 = Flash[0x08004000] = 50
   R1 becomes 0x08004001

2. STRB R3, [R0]    ; RAM[0x20000000] = 50
   R0 becomes 0x20000001

3. SUBS R2, R2, #1  ; R2 = 4095

4. BNE .Lloop       ; Repeat...

After 4096 iterations:
R0 = 0x20001000
R1 = 0x08005000
R2 = 0

BX LR returns to 0x0800010A (startup code)
```

---

## **<a name="performance-analysis"></a>Performance Analysis: Manual vs Bulk Initialization**

### **<a name="byte-count-table"></a>Byte Count Summary Table**

| Component | **Bytes** | **What It Is** | **Where Stored** |
|-----------|-----------|----------------|------------------|
| `LDR R0, =config` | 4 | Load address instruction | Flash |
| `LDR R1, =50` | 4 | Load value instruction | Flash |
| `STR R1, [R0]` | 2 | Store instruction | Flash |
| `BX LR` | 2 | Return instruction | Flash |
| **Per-variable manual init** | **12** | Total for one variable | Flash |
| **For 1000 variables** | **12,000** | 1000 × 12 | Flash (wasted) |
| **Memcpy loop** | ~10 | Copy loop code | Flash (reusable) |
| `BL memcpy` | 4 | Call instruction | Flash |
| **Startup code total** | **14** | One-time cost | Flash |
| **Savings** | **11,986** | 12,000 - 14 | Flash (saved) |

### **<a name="address-calculation"></a>Address Calculation for 1000 Variables**

**For 1000 variables:**
```
Flash storage of values:
0x08004000:  32 00 00 00  (var1 = 50)
0x08004004:  64 00 00 00  (var2 = 100)
0x08004008:  96 00 00 00  (var3 = 150)
...
0x08004FBC:  E8 03 00 00  (var1000 = 1000)

RAM destination:
0x20000000:  var1
0x20000004:  var2
0x20000008:  var3
...
0x20000FB8:  var1000

memcpy(0x20000000, 0x08004000, 4000);
// Copies 4000 bytes in one operation
```

---

## **<a name="key-takeaway"></a>Key Takeaway: The Cost of Abstraction**

### **Cost Comparison**

**Manual init (12 bytes per variable):**
```assembly
LDR R0, =var1
LDR R1, =50
STR R1, [R0]
; ... repeated 1000 times ...
```
**Total: 12,000 bytes** of **code** (instructions)

**Bulk init (14 bytes total):**
```assembly
LDR R0, =dest
LDR R1, =src
LDR R2, =4000
BL memcpy   ; Loop inside memcpy runs 1000 times
```
**Total: 14 bytes** of **code** + **4000 bytes** of **data** (initial values)

### **Core Philosophy**
- **Instructions are expensive:** Each one costs Flash space
- **Data is cheap:** Can be bulk-copied efficiently
- **Loops are reusable:** One loop handles any size

**This is why `.data` section exists:** **Data compression** through bulk operations.