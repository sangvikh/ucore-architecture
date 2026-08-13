# $\mu$-Core Application Binary Interface (v1.0 Standard)

**Target ISA:** $\mu$-Core ISA v4.6.0

**Target Profile:** C Compilers, Hand-written Assembly, Runtime Linkers, OS Kernels

---

## 1. Register Classification & Calling Conventions

Because $\mu$-Core only has four general-purpose registers, **all registers are volatile (caller-saved) by default**. If a caller needs register values to survive across a `CALL`, the caller pushes them to the LIFO stack prior to the call.

### Register Assignment Matrix

| Register | ABI Name | Volatility | Primary Role |
| --- | --- | --- | --- |
| **`A`** | `arg0` / `rv0` | **Volatile** | **1st Argument** / **Primary Return Value** / Accumulator |
| **`B`** | `arg1` / `rv1` | **Volatile** | **2nd Argument** / **High Word Return Value** / Temp |
| **`C`** | `ptr_h` | **Volatile** | Pointer High Page (`PR` target / Memory Page Selector) |
| **`D`** | `ptr_l` | **Volatile** | Pointer Low Offset (Index Register / Memory Offset) |
| **`PC`** | `pc` | *Special* | Program Counter ($W$-bit page offset) |
| **`PR`** | `pr` | *Special* | Code Page Register ($W$-bit code page selector) |

> **Key Rule:** `C:D` forms the canonical **$2W$-bit Universal Pointer**. When passing a memory address argument to a function, `C:D` is always used.

---

## 2. Argument Passing & Return Values

### Argument Passing Protocol

Arguments are passed in registers from left to right:

1. **1st Scalar Argument ($W$ bits):** Passed in **`A`**.
2. **2nd Scalar Argument ($W$ bits):** Passed in **`B`**.
3. **3rd+ Scalar Arguments ($W$ bits):** Pushed onto the **Hardware Stack** in right-to-left order before `CALL`. The caller is responsible for cleaning up pushed arguments after `RET` (or the callee pops them if using a fixed signature).
4. **Pointer / Memory Address Argument ($2W$ bits):** Passed in **`C:D`** (`C` = Page, `D` = Offset).

### Return Value Protocol

1. **Scalar Return Value ($W$ bits):** Returned in **`A`**.
2. **Double-Word Return Value ($2W$ bits):** Low word returned in **`A`**, High word returned in **`B`** (`B:A`).
3. **Pointer / Address Return Value ($2W$ bits):** Returned in **`C:D`**.
4. **Void / No Return:** Contents of `A`, `B`, `C`, `D` are undefined upon return.

---

## 3. Standard Function Framing & Stack Protocol

Every subroutine call (`CALL`) automatically pushes `PR`, then `PC` (creating a 2-word hardware stack frame).

```text
Stack State immediately inside Callee:
[ SP - 1 ] -> Return PC
[ SP - 2 ] -> Return PR
[ SP - 3 ] -> Pushed Arg 3 (if applicable)
[ SP - 4 ] -> Pushed Arg 4 (if applicable)

```

### Leaf Functions (No nested `CALL`s)

Leaf functions do **not** need to save `PR` or `PC`. They can freely use `A`, `B`, `C`, `D` as scratch registers and return immediately using `RET`.

```assembly
; Leaf Function: add_and_double(A, B) -> A = (A + B) * 2
add_and_double:
    ALU   ADD       ; A <- A + B
    MOV   B, A      ; B <- A
    ALU   ADD       ; A <- A + B (A * 2)
    RET             ; Return to caller (Pops PC, then PR)

```

### Non-Leaf Functions (Nested `CALL`s)

If a function calls another function, its scratch data in `A`, `B`, `C`, `D` must be saved to the stack before invoking the nested `CALL`.

```assembly
; Non-Leaf Function Example: process_data(C:D)
process_data:
    PUSH  C         ; Preserve High Pointer
    PUSH  D         ; Preserve Low Pointer
    
    LOAD  MAX-1     ; A <- Memory[C:D]
    CALL  helper    ; Call helper function (clobbers A, B, C, D)
    
    POP   D         ; Restore Low Pointer
    POP   C         ; Restore High Pointer
    STORE MAX       ; Memory[C:D] <- A; D <- D + 1 (Auto-increment)
    RET

```

---

## 4. Multi-Precision ($2W$-bit) Arithmetic Conventions

For $2W$-bit values (e.g., 16-bit values on $\mu8$, or 32-bit values on $\mu16$):

* **Register Pairs:** Always represented as **`B:A`** (where `B` is High Word, `A` is Low Word).
* **Addition / Subtraction Pipeline:**
1. Process Low Word `A` first to establish `CF` (Carry Flag).
2. Process High Word `B` using persistent flag latches.



```assembly
; 2W-bit Addition: (B:A) = (B:A) + (D:C)
; Inputs:  A = Low1, B = High1, C = Low2, D = High2
; Output:  A = LowResult, B = HighResult
add2w:
    ; 1. Low Word Addition
    PUSH  B         ; Save High1
    MOV   B, C      ; B <- Low2
    ALU   ADD       ; A <- Low1 + Low2 (Updates CF)
    POP   C         ; C <- High1 (Restores High1 into C)
    
    ; 2. High Word Addition with Carry
    ; (Uses SKP C to handle carry propagation)
    PUSH  A         ; Save Low Result
    MOV   A, C      ; A <- High1
    MOV   B, D      ; B <- High2
    SKP   C         ; Skip next if CF == 0
    ALU   INC       ; A <- High1 + 1 (if Carry set)
    ALU   ADD       ; A <- High1 + High2
    MOV   B, A      ; B <- High Result
    POP   A         ; A <- Low Result
    RET

```

---

## 5. Memory & Linkage Layout Standards

1. **Page Zero (`PR = 0`):** Reserved for System Vector Table, I/O Driver Stubs, and Kernel Entry Points.
2. **Global Data Pointer (`C` Register):** High page index `C` acts as the active "Data Page".
3. **Function Pointers:** Represented as $2W$-bit structures `PR:PC` (`C` holds target `PR`, `D` holds target `PC`). Executed via `JMP MAX` or `CALL MAX`.

---

## Summary of Calling Rules

```text
Caller Responsibilities:
1. Put Arg 1 in A (or Pointer in C:D).
2. Put Arg 2 in B.
3. Push Arg 3..N onto Stack (if any).
4. Save any live registers (A, B, C, D) to Stack before CALL.
5. Issue CALL.
6. Pop Arg 3..N from Stack upon return.

Callee Responsibilities:
1. Perform computation.
2. Place Scalar Return in A (or Pointer in C:D, or 2W value in B:A).
3. Ensure Stack depth is balanced (all local pushes popped).
4. Issue RET.

```

---