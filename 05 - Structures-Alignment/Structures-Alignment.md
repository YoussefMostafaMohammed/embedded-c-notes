
# **Structures-Alignment**

### **Table of Contents**
1. **[The Padding Problem](#1-the-padding-problem)**
2. **[Manual Alignment Control](#2-manual-alignment-control)**
3. **[Why Bit Fields Are Evil](#3-why-bit-fields-are-evil)**
4. **[Hardware Register Access Patterns](#4-hardware-register-access-patterns)**

---

### **1. The Padding Problem**

```c
struct {
    char a;  // 1 byte
    int b;   // 4 bytes
} s;
```

**Why size is 8 bytes, not 5:**
```
Memory layout:
Offset 0: [a]          (1 byte)
Offset 1: [padding]    (3 bytes - waste!)
Offset 4: [b  b  b  b] (4 bytes)
```

**Why padding exists:**
- **ARM requirement:** 32-bit access MUST be 4-byte aligned
- `LDR R0, [R1, #1]` (unaligned) → **HardFault** on Cortex-M0
- **Performance penalty:** Unaligned access takes 2 bus cycles instead of 1

**Compiler inserts padding** to ensure `&s.b` is divisible by 4.

---

### **2. Manual Alignment Control**

```c
#pragma pack(1)
struct {
    char a;
    int b;  // WARNING: b is now at offset 1 (unaligned!)
} s;
```

**What happens on ARM Cortex-M0:**
```assembly
LDR R0, =s
LDR R1, [R0, #1]     ; Unaligned load
; CPU raises HardFault exception
; System crashes
```

**What happens on Cortex-M3/M4:**
- Hardware **allows** unaligned access
- But **penalizes** with 2-cycle latency
- **Atomicity broken:** 32-bit access becomes two 16-bit accesses

**When to use pack(1):**
```c
// Only for externally-defined structures
// (USB descriptors, network packets, file formats)
typedef struct __attribute__((packed)) {
    uint8_t length;
    uint16_t language;  // Unaligned but required by USB spec
} USB_Desc;
```

**Always restore alignment:**
```c
#pragma pack(push, 1)
// packed structs here
#pragma pack(pop)  // Restore default
```

---

### **3. Why Bit Fields Are Evil**

```c
struct {
    unsigned int a:3;  // 3 bits
    unsigned int b:5;  // 5 bits
} bits;
```

**C standard leaves undefined:**
1. **Bit order:** Does `a` occupy bits [2:0] or [7:5]?
2. **Padding:** Can compiler insert bits between fields?
3. **Container:** Uses 8-bit, 16-bit, or 32-bit storage?

**Real-world failure:**
```c
// Hardware register: bit 5 = ENABLE
bits.b = 1;  // Meant to set bit 5

// GCC ARM: Sets bit [4:0] to 00001
// IAR ARM: Sets bit [7:3] to 00001
// Result: One works, other doesn't!
```

**Safe alternative:**
```c
#define REG_ENABLE (1 << 5)
*(volatile uint32_t*)0x40020000 |= REG_ENABLE;
// Single instruction, unambiguous, portable
```

---

### **4. Hardware Register Access Patterns**

```c
// UART control register bits
#define UART_CR1_UE  (1 << 13)  // Enable
#define UART_CR1_M   (1 << 12)  // Word length
#define UART_CR1_PCE (1 << 10)  // Parity control
```

**Safe manipulation macros:**
```c
#define SET_BITS(reg, mask)   do { (reg) |= (mask); } while(0)
#define CLEAR_BITS(reg, mask) do { (reg) &= ~(mask); } while(0)
#define READ_BIT(reg, bit)    (((reg) >> (bit)) & 1)

// Usage:
SET_BITS(UART1->CR1, UART_CR1_UE);      // Enable
CLEAR_BITS(UART1->CR1, UART_CR1_M);     // 8-bit mode
int parity = READ_BIT(UART1->CR1, 10);  // Read parity bit
```

**Why this is safe:**
- **No read-modify-write race:** `|=`, `&=` are atomic single instructions
- **Clear intent:** Code shows exactly which bits are changed
- **Portable:** Same macro works on all ARM chips
