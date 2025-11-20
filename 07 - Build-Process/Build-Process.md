

# **Build-Process**

### **Table of Contents**
1. **[From Source to Hex: The Pipeline](#1-from-source-to-hex-the-pipeline)**
2. **[Object Files and Symbols](#2-object-files-and-symbols)**
3. **[The Linker's Job](#3-the-linkers-job)**
4. **[Linker Scripts Decoded](#4-linker-scripts-decoded)**
5. **[ELF vs HEX Files](#5-elf-vs-hex-files)**

---

### **1. From Source to Hex: The Pipeline**

```bash
main.c → main.i → main.s → main.o → firmware.elf → firmware.hex
   ↓        ↓        ↓        ↓         ↓           ↓
  cpp     cc1       as       ld      objcopy     programmer
(5000 lines)

- **cpp** (Preprocessor)
  - Handles #define, #include
  - Output: Expanded source code

- **cc1** (C Compiler)
  - Parses C, generates assembly
  - Output: main.s (human-readable assembly)

- **as** (Assembler)
  - Converts assembly to machine code
  - Output: main.o (object file with symbols)

- **ld** (Linker)
  - Combines .o files, resolves symbols
  - Uses linker script for memory layout
  - Output: firmware.elf (executable with debug info)

- **objcopy**
  - Strips symbols, converts to plain binary/hex
  - Output: firmware.hex (Intel HEX format)
```

---

### **2. Object Files and Symbols**

**Inspect an object file:**
```bash
arm-none-eabi-objdump -x main.o
```

**Output:**
```
Sections:
Idx Name    Size      VMA      LMA      File off  Algn
  0 .text   00000034 00000000 00000000 00000034  2**1
  1 .data   00000004 00000000 00000000 00000068  2**2
  2 .bss    00000004 00000000 00000000 0000006c  2**2

SYMBOL TABLE:
00000000 l    df *ABS*  00000000 main.c
00000000 l    d  .text  00000000 .text
00000000 g     F .text  0000001c main
00000000 g     O .data  00000004 global_var
00000000         *UND*  00000000 printf
```

**Column meanings:**
- **l/g**: Local/Global binding
- **F/O**: Function/Object type
- **UND**: Undefined (needs linking)
- **`*ABS*`** : Absolute symbol (not in section)

**Relocation table:**
```bash
arm-none-eabi-objdump -r main.o
```
```
OFFSET   TYPE              VALUE
00000008 R_ARM_ABS32       printf
```
**R_ARM_ABS32**: 32-bit absolute address relocation. Linker will replace offset 8 with actual `printf` address.

---

### **3. The Linker's Job**

**Linking process:**
1. **Collect all .o files:** main.o, startup.o, libc.a
2. **Build symbol map:**
   ```
   Symbol: __data_start
   - Defined in main.o at 0x20000000
   
   Symbol: printf
   - Undefined in main.o
   - Found in libc.a at 0x08001200
   ```
3. **Resolve relocations:**
   ```
   In main.o at offset 0x08: CALL printf
   - Replace with: CALL 0x08001200
   ```
4. **Assign final addresses:**
   ```
   .text starts at 0x08000100
   .data starts at 0x20000000
   ```
5. **Generate ELF segments:**
   - **LOAD segment**: Flash content (.text, .rodata, .data_init)
   - **LOAD segment**: RAM content (.data, .bss zeroed)

---

### **4. Linker Scripts Decoded**

**STM32F103.ld:**
```ld
ENTRY(Reset_Handler)  /* First instruction */

MEMORY
{
    FLASH (rx) : ORIGIN = 0x08000000, LENGTH = 512K
    RAM   (xrw): ORIGIN = 0x20000000, LENGTH = 64K
}

SECTIONS
{
    .isr_vector : { *(.isr_vector) } >FLASH
    
    .text : { *(.text) } >FLASH  /* All code */
    
    .data : 
    {
        _sdata = .;              /* RAM start */
        *(.data)                 /* Initialized data */
        _edata = .;              /* RAM end */
    } >RAM AT> FLASH             /* AT> = initial values in Flash */
    
    .bss :
    {
        _sbss = .;               /* RAM start */
        *(.bss)                  /* Zeroed data */
        _ebss = .;               /* RAM end */
    } >RAM
}
```

**Key symbols:**
-  **`_sdata`**  : Start of .data in RAM (set by linker)
-  **`_edata`**  : End of .data in RAM
-  **`_sbss`**  : Start of .bss in RAM
-  **`_ebss`**  : End of .bss in RAM

**How startup uses these:**
```c
// In C code, reference linker symbols:
extern char _sbss, _ebss;

void clear_bss(void) {
    memset(&_sbss, 0, &_ebss - &_sbss);
}
```

---

### **5. ELF vs HEX Files**

**ELF file (firmware.elf)**
```bash
$ arm-none-eabi-readelf -h firmware.elf
ELF Header:
  Class: ELF32
  Type:  EXEC (Executable file)
  Machine: ARM
  Entry: 0x08000100 (Reset_Handler)
  Sections: 12

$ nm firmware.elf
08000100 T Reset_Handler
20000000 D __data_start
20000400 B __bss_start
```
**Contains:** Debug symbols, section addresses, everything needed for debugging with GDB.

**HEX file (firmware.hex)**
```
:020000020800F2  ; Record type 02, address 0x0800
:1000000000200010200100084101000841010008A0  ; 16 bytes at 0x08000000
:100010004101000841010008410100084101000890  ; 16 bytes at 0x08000010
:0400000508000100EF  ; Entry point record
:00000001FF          ; End of file
```

**Contains:** Only raw bytes and addresses. **No symbols.** Used for production programming.

**Why both formats needed:**
- **ELF**: For developers (debugging)
- **HEX**: For manufacturing (programming tools)
