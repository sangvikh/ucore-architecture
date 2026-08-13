# μ-Core Application Binary Interface (ABI Specification v1.0)

**Target ISA:** μ-Core v4.6.0 Standard

**Datapath Width:** $W$-bit ($W \ge 4$)

**Status:** Canonical Reference Specification (Locked)

---

## 1. Data Representation & Endianness

| Data Type | Width (Bits) | Alignment | Memory Representation |
| --- | --- | --- | --- |
| **Storage Unit (`unit_t`)** | $W$ bits | 1 Storage Unit | Single memory location |
| **Instruction (`inst_t`)** | $2W$ bits | Even Storage Unit | Opcode at $2k$, Operand at $2k+1$ |
| **Pointer / Full Address (`addr_t`)** | $2W$ bits | 1 Storage Unit | $2W$-bit address split into $C$ (High $W$) and $D$ (Low $W$) |
| **Multi-Unit Integers** | $N \times W$ bits | 1 Storage Unit | **Little-Endian:** LSB unit at address $k$, MSB unit at $k + N - 1$ |

* **Extended Pointer Format ($2W$ bits):** Extended addresses are represented as $C \mathbin{:} D$, where $C$ holds the High $W$-bit page index ($Y$) and $D$ holds the Low $W$-bit offset pointer ($X$).

---

## 2. Register Conventions & Preservation Rules

Because μ-Core features four general/address registers ($A, B, C, D$), register usage is strictly partitioned between caller and callee to minimize stack overhead.

| Register | Binary ID | ABI Name | Preservation Rule | Primary Usage Role |
| --- | --- | --- | --- | --- |
| **A** | `00` | `a0` | **Caller-Saved** | Primary Argument (`arg0`), Primary Return Value (`rv0`), ALU Accumulator |
| **B** | `01` | `a1` | **Caller-Saved** | Secondary Argument (`arg1`), Secondary Return Value (`rv1`), ALU Source |
| **C** | `10` | `p_hi` | **Volatile / Scratch** | High Address Pointer ($Y$) / Far Branch Target Page / Callee-Saved for Local Calls |
| **D** | `11` | `p_lo` | **Volatile / Scratch** | Low Address Pointer ($X$) / Index Register / Callee-Saved for Local Calls |

### Flag Latches ($ZF, CF$)

* **Zero Flag ($ZF$) & Carry Flag ($CF$):** **Volatile**. Functions may overwrite condition flags without preserving them. Flags are optionally used as boolean return statuses (e.g., $CF = 1 \implies \text{Error/Borrow}$).

---

## 3. Subroutine Calling Convention

### Argument Passing

1. **Argument 1 (`arg0`):** Passed in Register **$A$**.
2. **Argument 2 (`arg1`):** Passed in Register **$B$**.
3. **Arguments 3+ (`arg2`..`argN`):** Because μ-Core's stack lacks an offset pointer (no $SP + \text{offset}$ addressing), arguments beyond 2 **MUST NOT** be passed on the stack. Instead, additional arguments are passed via a **Parameter Block in RAM**, with Register Pair **$C \mathbin{:} D$** holding the $2W$-bit memory address of the parameter block.

```text
  C:D ────> [ Arg 2 (Unit 0) ]
            [ Arg 3 (Unit 1) ]
            [ Arg 4 (Unit 2) ]

```

### Return Values

* **Scalar Return ($W$ bits):** Returned in Register **$A$**.
* **Extended / Pointer Return ($2W$ bits):** Returned across Register Pair **$C \mathbin{:} D$** ($C = \text{High Page}$, $D = \text{Low Offset}$).
* **Status Flags:** Functions may report success/failure via $ZF$ ($1 = \text{Success}$, $0 = \text{Failure}$) or $CF$ ($1 = \text{Error}$).

### Local vs. Far Subroutine Execution

#### Local Call (Within Active Page $PR$)

Caller loads $A$ and $B$, then issues `CALL Target`.

```assembly
; Local Call Example (Target in active page)
LOAD 0x05      ; arg0 = 5
MOV  A, B      ; arg1 = 5
LOAD 0x10      ; arg0 = 0x10
CALL 0x40      ; Local call to offset 0x40

```

#### Far Call (Cross-Page Execution)

Caller loads target page into $C$, target offset into $D$, arguments into $A$ and $B$, then issues `CALL MAX`.

```assembly
; Far Call Example (Target Page 2, Offset 0x10)
LOAD 2
MOV  A, C      ; C = Page 2
LOAD 0x10
MOV  A, D      ; D = Offset 0x10
LOAD 0x42      ; arg0 = 0x42
CALL MAX       ; Far Call via C:D (Hardware pushes PR_ret:PC_ret frame)

```

---

## 4. Stack Frame Protocol

The μ-Core stack is a hardware-enforced, abstract LIFO ring buffer.

### Subroutine Entry & Exit Lifecycle

1. **Call Push:** Executing `CALL` automatically pushes a 2-word return frame onto the stack:

$$\text{Stack Top} \longrightarrow [\, PC_{ret} \,] \longrightarrow [\, PR_{ret} \,]$$


2. **Prologue:** If the callee needs to preserve $C$ or $D$ across internal operations, it pushes them immediately upon entry:
```assembly
PUSH C      ; Save caller's High Pointer
PUSH D      ; Save caller's Low Pointer

```


3. **Epilogue:** Before returning, the callee pops any saved registers in reverse order:
```assembly
POP  D      ; Restore caller's Low Pointer
POP  C      ; Restore caller's High Pointer
RET         ; Pops PC_ret, then PR_ret into hardware registers

```



### Stack Balance Invariant

> **Normative Rule:** Every function MUST leave the stack in the exact state it was found upon entry. The total number of `PUSH` operations inside a function MUST equal the total number of `POP` operations prior to executing `RET`.

---

## 5. Standard System Call & I/O Port Assignments

Standard peripheral device addresses and kernel services are mapped to fixed $W$-bit I/O ports and Page 0 syscall vectors.

### Standard $W$-Bit Peripheral I/O Port Map (`IO`)

| Port ID | Direction | Functional Mapping | Operational Behavior |
| --- | --- | --- | --- |
| `0` | **IN / OUT** | **Console TTY Stream** | Reads/Writes ASCII or raw $W$-bit character stream |
| `1` | **IN / OUT** | **Block Storage Stream** | Sequentially streams $W$-bit blocks from ROM/Flash/Tape |
| `2` | **IN / OUT** | **System Timer / Clock** | Reads tick counter or writes timer match value |
| `3` | **IN / OUT** | **Interrupt / Status** | Reads pending interrupt mask or writes ACK bits |
| `4`..`6` | **IN / OUT** | **User Expansion** | General-purpose user peripheral channels |
| `7` | **OUT** | **System Control / Power** | Bits `[0]`: Soft Reset, `[1]`: Power Off / HLT |

### System Call Interface (`Syscall`)

System calls to Page 0 (Kernel / Boot Services) use $A$ as the **Syscall ID Number**:

* **Syscall Vector Entry:** Executed via `CALL` to Page 0, Address `0x00` ($C=0, D=0$).
* **Syscall ID (`arg0`):** Register $A$ contains the requested service ID.
* **Syscall Parameters:** Register $B$ or RAM Parameter Block at $C \mathbin{:} D$.

| Syscall ID ($A$) | Service Mnemonic | Input Parameters | Output / Status |
| --- | --- | --- | --- |
| `0x0` | **SYS_YIELD** | None | Yields CPU time slice |
| `0x1` | **SYS_READ** | $B = \text{Port ID}$ | $A = \text{Data Read}$, $ZF = 1 \implies \text{Success}$ |
| `0x2` | **SYS_WRITE** | $B = \text{Port ID}$, RAM at $C \mathbin{:} D$ | $A = \text{Units Written}$ |
| `0x3` | **SYS_EXEC** | $C \mathbin{:} D = \text{Program Page Buffer}$ | Transfer control to target page |
| `0xF` | **SYS_EXIT** | $B = \text{Exit Code}$ | Halts task / Returns to shell |