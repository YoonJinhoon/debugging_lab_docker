# CSAPP Chapter 3: Machine-Level Representation of Programs

A study of machine-level execution through the lens of formal logic: translating **Human Desire** (software abstraction) into the **Physical Axioms** of x86_64 hardware.

---

## The 5 Invariant Physical Axioms

1. **Axiom 1 (Flat Byte-Addressable Memory):**
   Memory is a linear mapping $\mathcal{M}: [0, 2^{64}-1] \to \{0, 1\}^8$. Memory cells have no types, boundaries, or variable names; only numerical indices (addresses).
2. **Axiom 2 (Page Protection & The Zero Page):**
   Memory is mapped in 4 KB pages. The interval $[0, 4095]$ is unmapped ($\text{PROT} = \emptyset$). Dereferencing any pointer in this range triggers an immediate hardware trap (`SIGSEGV`).
3. **Axiom 3 (Registers as State Storage):**
   Arithmetic and logical operations cannot directly connect two arbitrary memory locations. Data flows through discrete registers ($\mathcal{R}$).
4. **Axiom 4 (Monotonic Execution & Condition Codes):**
   The program counter ($\text{RIP}$) moves monotonically forward unless redirected by jumps governed by condition flags ($\mathcal{C} = \langle \text{ZF}, \text{SF}, \text{CF}, \text{OF} \rangle$).
5. **Axiom 5 (The LIFO Stack Frontier):**
   The call stack grows downward toward lower memory addresses, tracked by $\text{RSP}$. Activation records manage nested procedure lifetimes.

---

## Experiments & Verification

### Section 3.4 & 3.5: Data Movement (`movq`) vs. Address Arithmetic (`leaq`)

**Source (`test3_4.c`):**
```c
long move_example(long x) {
    long y = x + 4;
    return y;
}
### The Contrast: Mathematical Elimination of Redundancy

Compare the two versions side-by-side:[cite: 1]

-------------------------------------------------------------------------------------
 `-O0` (Unoptimized: 10 instructions) | `-O2` (Optimized: 2 functional instructions)
-------------------------------------------------------------------------------------
 pushq %rbp                           | no stack setup)
 movq %rsp, %rbp                      | no base pointer frame
 movq %rdi, -24(%rbp)                 | no memory write for `x`
 movq -24(%rbp), %rax                 | no memory read)
 addq $4, %rax                        | leaq 4(%rdi), %rax` 
 ovq %rax, -8(%rbp)                   | no memory write for `y`
 movq -8(%rbp), %rax                  | no memory read
 popq %rbp                            | no stack teardown
 ret                                  | ret
-------------------------------------------------------------------------------------
---

## Section 3.6: Control Flow, Condition Codes, and Inversion

### 1. Condition Codes Register ($\mathcal{C}$)

Arithmetic operations update single-bit status flags in `%rflags`:


-------------------------------------------------------------------------------------
Flag   	Name       	Mathematical Definition	          Hardware Meaning
-------------------------------------------------------------------------------------
ZF	   Zero Flag	      Result==0                    	The operation produced a zero (e.g., a−b=0, so a==b).
SF	   Sign Flag	      Result<0	                     The most significant bit (MSB) of the result is 1 (negative).
OF	   Overflow Flag  (a>0,b>0,Res<0)∨(a<0,b<0,Res>0)	Two's-complement signed overflow occurred.
CF	   Carry Flag	      Unsigned Overflow	            An unsigned addition carried out of the MSB, or a borrow occurred.
-------------------------------------------------------------------------------------


### 2. Source Code & Disassembly (`test3_6.c`)

```c
long max(long a, long b) {
    if (a > b) {
        return a;
    } else {
        return b;
    }
}
+-----------------------------------+
                  |           Function Entry          |
                  |  pushq   %rbp                     |
                  |  movq    %rsp, %rbp               |
                  |  movq    %rdi, -8(%rbp)   (save a)|
                  |  movq    %rsi, -16(%rbp)  (save b)|
                  +-----------------------------------+
                                    |
                                    v
                  +-----------------------------------+
                  |             Condition             |
                  |  movq    -8(%rbp), %rax   (%rax=a)|
                  |  cmpq    -16(%rbp), %rax  (a - b) |
                  +-----------------------------------+
                                    |
                            jle .L2 (a <= b)
                           /                 \
                 [ True ] /                   \ [ False ]
                         /                     \
                        v                       v
      +----------------------------+  +----------------------------+
      |      .L2 (Else Block)      |  |      Then-Fallthrough      |
      |  movq  -16(%rbp), %rax     |  |  movq  -8(%rbp), %rax      |
      |        (%rax = b)          |  |        (%rax = a)          |
      +----------------------------+  |  jmp   .L3                 |
                    |                 +----------------------------+
                    \                               /
                     \                             /
                      ----->        .L3       <----
                                     |
                                     v
                  +-----------------------------------+
                  |             Function Exit         |
                  |  popq    %rbp                     |
                  |  ret                              |
                  +-----------------------------------+
### Control Flow Execution Path (`-O2`)

Unlike the branching diamond of `-O0`, `-O2` uses conditional moves (`cmovge`) to keep execution strictly linear and monotonic:

```text
               +----------------------------------------+
               |              Function Entry            |
               |  (No stack setup, no memory writes)    |
               |  Arguments already in registers:       |
               |      %rdi = a,  %rsi = b               |
               +----------------------------------------+
                                   |
                                   v  [Monotonic Flow: %rip advances]
               +----------------------------------------+
               |             1. Comparison              |
               |  cmpq   %rsi, %rdi                     |
               |  Computes: (%rdi - %rsi) = (a - b)     |
               |  Sets flags: CF, ZF, SF, OF in %rflags |
               +----------------------------------------+
                                   |
                                   v  [Monotonic Flow: %rip advances]
               +----------------------------------------+
               |        2. Default Assignment           |
               |  movq   %rsi, %rax                     |
               |  State: %rax = b                       |
               +----------------------------------------+
                                   |
                                   v  [Monotonic Flow: %rip advances]
               +----------------------------------------+
               |         3. Conditional Move            |
               |  cmovge %rdi, %rax                     |
               |  Condition: (SF ^ OF) == 0 (i.e. a>=b) |
               |                                        |
               |  [ a >= b ]: %rax <-- %rdi (value a)   |
               |  [ a <  b ]: %rax unchanged (value b)  |
               +----------------------------------------+
                                   |
                                   v  [Monotonic Flow: %rip advances]
               +----------------------------------------+
               |             Function Exit              |
               |  ret (Returns value in %rax)           |
               +----------------------------------------+
Architectural Contrast: Conditional Jump (-O0) vs. Conditional Move (-O2)Metric-O0 (Conditional Jump: jle / jmp)-O2 (Conditional Move: cmovge)Instruction Count10 instructions (stack + jumps)4 instructions (all register-level)Branch PenaltySubject to branch misprediction penalty (15–30 cycles)0 branch penalty (pipeline never stalls)Execution FlowNon-monotonic (%rip branches to .L2 / .L3)Strictly monotonic (%rip advances linearly)Memory Access4 stack writes, 4 stack reads0 memory reads/writes (pure registers)
