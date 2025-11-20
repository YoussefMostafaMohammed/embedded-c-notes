
# **Assembly-CPU-Mechanics**

### **Table of Contents**
1. **[Core ARM Registers](#1-core-arm-registers)**
2. **[Load/Store Instructions](#2-loadstore-instructions)**
3. **[Branch and Call Instructions](#3-branch-and-call-instructions)**
4. **[Bit Manipulation Instructions](#4-bit-manipulation-instructions)**
5. **[Stack Operations](#5-stack-operations)**
6. **[Context Switch Mechanics](#6-context-switch-mechanics)**

---

### **1. Core ARM Registers**

```
Register | Name | Purpose
---------|------|--------
R0-R3    |      | Argument/scratch registers
R4-R11   |      | Callee-saved registers
R12      | IP   | Intra-procedural scratch
R13      | SP   | Stack Pointer
R14      | LR   | Link Register (return address)
R15      | PC   | Program Counter
```

**Special registers:**
- **PSR** (Program Status Register): Flags (N, Z, C, V, T)
- **CONTROL**: Thread mode, stack selection
- **PRIMASK**: Interrupt disable mask

**SP details:**
- **MSP** (Main SP): Used after reset, for OS
- **PSP** (Process SP): Used for threads
- Selected by `CONTROL[1]` bit

---

### **2. Load/Store Instructions**

**LDR (Load Register):**
```assembly
LDR R0, =0x20000000    ; Load address into R0 (32-bit)
; LDR = Load Register

LDR R1, [R0]           ; Load from memory at [R0] into R1
; [R0] means "memory addressed by R0"

LDR R2, [R0, #4]       ; Load from [R0 + 4]
; Offset addressing

LDR R3, [R0, R1, LSL #2]  ; Load from [R0 + (R1 << 2)]
; Scaled indexing
```

**STR (Store Register):**
```assembly
MOV R1, #50
STR R1, [R0]           ; Store R1 to memory at [R0]
; STR = STore Register

STRB R1, [R0]          ; Store Byte (lowest 8 bits only)
; STRB = STore Byte

STRH R1, [R0]          ; Store Halfword (lowest 16 bits)
; STRH = STore Halfword
```

**Why separate byte/halfword instructions?** Hardware buses are 32-bit wide. `STRB` generates a 32-bit write with byte-enable signals.

---

### **3. Branch and Call Instructions**

**B (Branch):**
```assembly
B .Lloop              ; PC = address of .Lloop
; B = Branch (unconditional)
```

**BL (Branch with Link):**
```assembly
BL func               ; LR = PC + 4, PC = func
; BL = Branch with Link (call function)
```

**BX (Branch eXchange):**
```assembly
BX LR                 ; PC = LR, return from function
; BX = Branch eXchange (return)
```

**BLX (Branch with Link eXchange):**
```assembly
BLX R0                ; LR = PC + 4, PC = R0
; BLX = Branch with Link eXchange
; Used for function pointer calls
```

**Instruction encoding:**
```
BL label:  F000 F804  (32-bit)
; F0 00: High part of offset
; F8 04: Low part of offset
; Offset = sign_extend(imm24 << 1)
```

---

### **4. Bit Manipulation Instructions**

**AVR-style (legacy):**
```assembly
BSET R0, #5    ; Set bit 5 in R0
; BSET = Bit SET

BCLR R0, #5    ; Clear bit 5
; BCLR = Bit CLeaR
```

**ARM Cortex-M:**
```assembly
BFC R0, #8, #4  ; Clear bits 8-11 in R0
; BFC = Bit Field Clear

BFI R0, R1, #8, #4  ; Insert R1[3:0] into R0[11:8]
; BFI = Bit Field Insert
```

**Why these exist:** Single-cycle atomic read-modify-write. Prevents race conditions.

---

### **5. Stack Operations**

**PUSH (multiple registers):**
```assembly
PUSH {R4-R7, LR}      ; Store R4,R5,R6,R7,LR onto stack
; SP = SP - (5 * 4) = SP - 20
; [SP+16] = LR
; [SP+12] = R7
; [SP+8]  = R6
; [SP+4]  = R5
; [SP+0]  = R4
```

**POP (multiple registers):**
```assembly
POP {R4-R7, PC}       ; Load from stack, return
; R4 = [SP+0]
; R5 = [SP+4]
; R6 = [SP+8]
; R7 = [SP+12]
; PC = [SP+16]  (RETURN!)
; SP = SP + 20
```

**Why PUSH/POP are special:** They operate on **multiple registers** in one instruction, saving code size.

---

### **6. Context Switch Mechanics**

**PendSV (context switch) handler:**
```assembly
PendSV_Handler:
    MRS R0, PSP          ; Get current task PSP
    STMDB R0!, {R4-R11}  ; Save callee-saved registers
    ; STMDB = STore Multiple, Decrement Before
    ; ! writes back new SP to R0

    LDR R1, =current_task
    STR R0, [R1]         ; Save task's SP to TCB

    PUSH {R14}           ; Save LR (exception return)
    BL vTaskSwitchContext  ; Choose next task
    POP {R14}

    LDR R0, [R1]         ; Get next task's SP
    LDMIA R0!, {R4-R11}  ; Restore registers
    ; LDMIA = LoaD Multiple, Increment After

    MSR PSP, R0          ; Set PSP to new task
    ORR LR, LR, #0x10    ; Return to thread mode with PSP
    BX LR                ; Exception return
```

**What happens on exception return:**
- CPU pops `R0-R3, R12, LR, PC, xPSR` from PSP stack
- **Hardware magic:** Loads next task's context automatically
- **Total switch time:** 12 cycles = **166 ns at 72MHz**
