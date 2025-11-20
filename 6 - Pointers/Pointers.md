
# **6-Pointers**

### **Table of Contents**
1. **[Function Pointers Demystified](#1-function-pointers-demystified)**
2. **[Reading Complex Declarations](#2-reading-complex-declarations)**
3. **[Generic Pointer Types](#3-generic-pointer-types)**
4. **[Pointer Arithmetic in Embedded](#4-pointer-arithmetic-in-embedded)**

---

### **1. Function Pointers Demystified**

```c
// Basic syntax
int (*func_ptr)(int, int);  // Pointer to function taking two ints, returning int

// Usage
int add(int a, int b) { return a + b; }
func_ptr = &add;  // Or just func_ptr = add;
int result = func_ptr(5, 3);  // Calls add(5, 3)
```

**Generated assembly:**
```assembly
LDR R0, =add         ; R0 = address of add function
MOV R1, #5
MOV R2, #3
BLX R0               ; Branch and Link Exchange to R0
; BLX = Branch with Link eXchange (can branch to any address)
; Result in R0
```

**Real-world: ISR vector table**
```c
// Vector table in startup file
__attribute__((section(".isr_vector")))
void (* const g_pfnVectors[46])(void) = {
    (void*)&_estack,    // Initial SP
    Reset_Handler,      // Reset
    NMI_Handler,        // Non-maskable interrupt
    HardFault_Handler,  // Hard fault
    // ... 42 more handlers
};

// CPU sees this as array of addresses in Flash:
// 0x08000000: 20010000  (stack top)
// 0x08000004: 08000120  (Reset_Handler address)
// 0x08000008: 08000140  (NMI_Handler address)
```

---

### **2. Reading Complex Declarations**

**The "Right-Left-Right" rule:**
```c
int (*func_array[10])(int, float);
```

**Parsing steps:**
1. **Start at identifier:** `func_array`
2. **Go right:** `[10]` → "array of 10"
3. **Go left:** `*` → "pointers"
4. **Go right:** `(int, float)` → "functions taking int and float"
5. **Go left:** `int` → "returning int"

**Result:** `func_array` is an array of 10 pointers to functions that take `(int, float)` and return `int`.

**The ultimate monster:**
```c
int (*(*(*func)(double))[5])(char*);
```
**Parse:** `func` is a pointer to function taking `double`, returning pointer to array of 5 pointers to functions taking `char*`, returning `int`.

**When used:** Never in practice. This is interview torture material.

---

### **3. Generic Pointer Types**

| Pointer Type | **Purpose** | **Valid Operations** |
|--------------|-------------|---------------------|
| `void*`      | Generic pointer (can cast to any) | Cast, assignment, pass as argument |
| `int*`       | Typed pointer | Dereference, arithmetic (`p++` adds sizeof(int)) |
| `uintptr_t`  | Integer holding address | Bit manipulation, NOT dereferencing |
| `far pointer`| 32-bit pointer to any segment | Legacy, x86 only |
| `near pointer`| 16-bit pointer to same segment | Legacy, x86 only |

**Void pointer usage:**
```c
void my_memcpy(void *dst, const void *src, size_t n) {
    uint8_t *d = (uint8_t*)dst;      // Cast to byte pointer
    const uint8_t *s = (const uint8_t*)src;
    while(n--) *d++ = *s++;
}

// Can copy any type:
int arr1[10];
int arr2[10];
my_memcpy(arr1, arr2, sizeof(arr1));
```

**Why void* is essential:** Hardware drivers need to handle any data type (UART, SPI, DMA).

---

### **4. Pointer Arithmetic in Embedded**

```c
int arr[10];
int *p = arr;

p++;  // R1 = R1 + 4  (sizeof(int) = 4)
p--;  // R1 = R1 - 4

char *c = (char*)arr;
c++;  // R0 = R0 + 1  (sizeof(char) = 1)
```

**Why this matters for hardware:**
```c
// Incrementing through peripheral registers
#define USART1_BASE 0x40013800
USART_TypeDef *usart = (USART_TypeDef*)USART1_BASE;

usart->CR1 = 0x01;       // Offset 0x00
usart->BRR = 0x0D;       // Offset 0x04 (pointer + 4)
usart->SR  = 0x00;       // Offset 0x08 (pointer + 8)
```

**Structure layout magic:**
```c
typedef struct {
    volatile uint32_t CR1;  // 0x00
    volatile uint32_t BRR;  // 0x04
    volatile uint32_t SR;   // 0x08
} USART_TypeDef;

USART_TypeDef *usart = (USART_TypeDef*)0x40013800;
usart->BRR = 9600;  // Compiler generates: STR R0, [R1, #4]
```
