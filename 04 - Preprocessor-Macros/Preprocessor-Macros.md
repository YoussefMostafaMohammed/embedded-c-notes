
# **Preprocessor-Macros**

### **Table of Contents**
1. **[Macro Expansion Mechanics](#1-macro-expansion-mechanics)**
2. **[The Dangers of Function Macros](#2-the-dangers-of-function-macros)**
3. **[Stringification & Concatenation](#3-stringification--concatenation)**
4. **[Practical Macro Patterns](#4-practical-macro-patterns)**
5. **[Compiler Directives (#pragma)](#5-compiler-directives-pragma)**

---

### **1. Macro Expansion Mechanics**

**What the preprocessor does:**
```c
#define X 10
#define Y X

int z = Y;  // Expands to: int z = 10;
```

**Step-by-step expansion:**
1. **Preprocessor sees:** `Y`
2. **Looks up Y:** Expands to `X`
3. **Sees X in result:** Expands `X` to `10`
4. **Final output:** `int z = 10;`

**Infinite recursion trap:**
```c
#define X Z
#define Z X

int a = X;  // Expands: X → Z → X → Z → ... → Error!
```

**Why this fails:** Preprocessor stops after 1000 recursions (implementation limit).

---

### **2. The Dangers of Function Macros**

**The semicolon trap:**
```c
#define my_print() {printf("A"); printf("B");};

if(x == 10)
    my_print();  // Expands to: {printf...}; ; 
else             // ERROR: else without if!
    x = 5;
```

**Preprocessed output:**
```c
if(x == 10)
    {printf("A"); printf("B");}; ;  // Two statements!
else
    x = 5;  // 'else' belongs to NULL statement
```

**Solution: do-while-zero idiom:**
```c
#define my_print() do {printf("A"); printf("B");} while(0)

// Expands to:
if(x == 10)
    do {printf...} while(0);  // Single statement, safe
else
    x = 5;
```

**Why this works:** `do-while` is **one statement** that **requires a semicolon**. No empty statement created.

---

### **3. Stringification & Concatenation**

```c
#define STR(x) #x        // # = stringify
#define CONCAT(a,b) a##b // ## = concatenate

printf("%s", STR(hello));  // "hello"
int xy = CONCAT(x, y);      // xy variable
```

**Concatenation trick (two-stage):**
```c
#define B1 0
#define B2 1
#define CONCAT_HELPER(a,b) 0b##a##b
#define CONCAT(a,b) CONCAT_HELPER(a,b)  // Forces expansion

CONCAT(B1,B2)  // Expands: B1,B2 → CONCAT_HELPER(0,1) → 0b01
```

**Why two stages needed:** `##` concatenates **before** argument expansion. Without helper, gets `0bB1B2` instead of `0b01`.

---

### **4. Practical Macro Patterns**

**Safe bit manipulation:**
```c
#define SET_BIT(reg, bit)    do { (reg) |= (1U << (bit)); } while(0)
#define CLEAR_BIT(reg, bit)  do { (reg) &= ~(1U << (bit)); } while(0)

// Usage:
SET_BIT(GPIOA->ODR, 5);  // No side effects
```

**Debug logging:**
```c
#ifdef DEBUG
    #define LOG(fmt, ...) printf("[%s:%d] " fmt, __FILE__, __LINE__, ##__VA_ARGS__)
#else
    #define LOG(fmt, ...) (void)0
#endif

LOG("Value: %d", x);
// Expands to: printf("[main.c:42] Value: %d", x);
```

**__VA_ARGS__** = Variable arguments, handled by preprocessor.

---

### **5. Compiler Directives (#pragma)**

**Why #pragma is not preprocessor:**
```c
#pragma pack(1)  // Command to COMPILER, not preprocessor
// Preprocessor just passes it through
// Compiler interprets it during code generation
```

**Common pragmas:**
```c
#pragma pack(push, 1)    // Push alignment, set to 1
#pragma pack(pop)        // Restore previous alignment

#pragma GCC optimize("O0")  // Disable optimization for this file

#pragma GCC diagnostic ignored "-Wunused"  // Suppress warnings
```

**What if pragma didn't exist?** No way to control:
- Struct alignment
- Interrupt vector placement
- Memory section assignment
- Function-specific optimization
