
# **Stack-Heap**

### **Table of Contents**
1. **[Stack Growth Mechanics](#1-stack-growth-mechanics)**
2. **[PSP vs MSP Dual Stack](#2-psp-vs-msp-dual-stack)**
3. **[Heap Fragmentation Deep Dive](#3-heap-fragmentation-deep-dive)**
4. **[Malloc Corruption Scenarios](#4-malloc-corruption-scenarios)**

---

### **1. Stack Growth Mechanics**

**Function prologue/epilogue:**
```c
void func(int a) {
    int x = 5;
}
```

```assembly
func:
    PUSH {R4, LR}      ; Prologue: save registers
    SUB SP, SP, #4     ; Allocate space for 'x'
    ; ... function body ...
    ADD SP, SP, #4     ; Epilogue: deallocate
    POP {R4, PC}       ; Restore and return
```

**Visual stack evolution:**
```
Before call:
SP = 0x2000FFFC

After PUSH {R4, LR}:
[0x2000FFF8] = R4 (saved)
[0x2000FFF4] = LR (saved)
SP = 0x2000FFF4

After SUB SP, #4:
[0x2000FFF0] = x (local variable)
SP = 0x2000FFF0

Before return:
SP = 0x2000FFF0

After ADD SP, #4:
SP = 0x2000FFF4

After POP {R4, PC}:
R4 = [0x2000FFF8]
PC = [0x2000FFF4] (return address)
SP = 0x2000FFFC
```

---

### **2. PSP vs MSP Dual Stack**

**Why two stacks exist:**
```c
// MSP (Main Stack Pointer)
// - Used by OS kernel
// - Used by interrupts
// - Always valid after boot
__set_MSP(0x20010000);

// PSP (Process Stack Pointer)
// - Used by user threads
// - Optional, enabled by OS
// - Each thread has its own PSP
__set_PSP(0x20009000);
```

**CONTROL register:**
```
Bit 1: SPSEL
  0 = Use MSP (default after reset)
  1 = Use PSP (in thread mode)

Bit 0: nPRIV
  0 = Privileged (full access)
  1 = Unprivileged (restricted access)
```

**Context switch in FreeRTOS:**
```c
void vTaskSwitchContext(void) {
    // Save current context
    current_task->sp = __get_PSP();
    
    // Choose next task
    next_task = scheduler();
    
    // Load next context
    __set_PSP(next_task->sp);
}
```

---

### **3. Heap Fragmentation Deep Dive**

**malloc internal structures:**
```c
struct free_block {
    size_t size;
    struct free_block *next;
};

// Heap maintains linked list:
free_list → [16 bytes] → [128 bytes] → [60000 bytes] → NULL
```

**Allocation algorithm:**
```c
void* malloc(size_t size) {
    // Find first block >= size
    for(block = free_list; block; block = block->next) {
        if(block->size >= size) {
            // Split block if too big
            if(block->size > size + sizeof(struct free_block)) {
                new_block = (char*)block + size;
                new_block->size = block->size - size;
                new_block->next = block->next;
                // Return original block
                return block;
            }
            // Remove block from list
            // Return block
        }
    }
}
```

**Fragmentation example:**
```
After malloc(100), malloc(200), free(100):
[100 FREE] → [200 USED] → [60000 FREE]

malloc(150):
[100 FREE] (can't use - too small!)
[150 USED] → [50 FREE] → [60000 FREE]

Result: 100 bytes free but unusable for 150-byte request
```

---

### **4. Malloc Corruption Scenarios**

**Main thread interrupted by ISR:**
```c
void main(void) {
    p = malloc(100);  // Finds block at 0x20001000
    // About to split it...
    // ISR fires here!
}

void SysTick_Handler(void) {
    q = malloc(50);   // Finds SAME block at 0x20001000
    // Splits it, writes metadata
    // Returns to main
    // Main's metadata is now corrupted!
}
```

**Why this happens:** `malloc` uses **global variables** (free_list head). ISR interruption corrupts internal state.

**Solutions:**
1. **Disable interrupts around malloc:**
```c
__disable_irq();
p = malloc(100);
__enable_irq();
```
2. **Use thread-safe malloc** (FreeRTOS pvPortMalloc)
3. **Ban malloc after init** (MISRA C Rule 21.3)
