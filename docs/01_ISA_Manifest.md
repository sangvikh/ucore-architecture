# μ-Core ISA Specification (v4.4.0 Canonical Standard)

**Status:** Canonical Reference Standard (Normative)

**Target Profile:** Discrete Transistors, Relays, CMOS/TTL, FPGA ($\mu 4, \mu 8, \mu 16$)

---

## 1. Architectural Philosophy & Parameterized Model ($W$)

The **μ-Core v4.4.0 ISA** is a parameterized, technology-independent architecture designed for maximum physical hardware minimalism, deterministic timing, and strict orthogonal execution. Datapath width $W$ ($W \ge 4$) defines storage units, registers, and memory boundaries. Every instruction occupies exactly **2 Storage Units ($2W$ bits)**: `[Opcode: W] [Operand: W]`.

| Architectural Property | Parameterized Definition | $\mu 4$ ($W=4$) | $\mu 8$ ($W=8$) | $\mu 16$ ($W=16$) |
| --- | --- | --- | --- | --- |
| **Register & Operand Width** | $W$ bits | 4 bits | 8 bits | 16 bits |
| **Instruction Size** | $2W$ bits ($2$ Storage Units) | 8 bits | 16 bits | 32 bits |
| **Instruction Page Capacity** | $2^{W-1}$ instructions | 8 instructions | 128 instructions | 32,768 instructions |
| **Data Memory Space ($C:D$)** | $2W$ bits ($2^{2W}$ locations) | 256 nibbles | 64 KiB | 4 GiB |
| **Indirect Sentinel (`MAX-1`)** | $2^W - 2$ | `0xE` | `$FE` | `$FFFE` |
| **Auto-Increment Sentinel (`MAX`)** | $2^W - 1$ | `0xF` | `$FF` | `$FFFF` |

---

## 2. Programmer State & Register Encodings

### Architectural State Tuple ($\mathcal{S}$)

$$\mathcal{S} = \langle A, B, C, D, PC, PR, SP, \text{Stack}[], \text{Memory}[] \rangle$$

### Condition Latches ($\mathcal{C}$)

Condition flags (**$ZF$**, **$CF$**) exist as **persistent hardware latches** on the ALU board:

* **Persistence:** $ZF$ and $CF$ hold their state across non-ALU operations (`MOV`, `LOAD`, `STORE`, `PUSH`, `POP`, `JMP`, `SKP`). They are updated *only* when an `ALU` instruction executes.
* **Isolation:** They are not general-purpose storage, are not addressable, are not connected to the main $W$-bit data bus, and are never saved or restored by `CALL` or `RET`.

### MOV Register Mapping (`Bits [3:0]`)

`MOV` uses bits `[3:2]` for Source Register (`src`) and bits `[1:0]` for Destination Register (`dst`):

| Binary ID | Hex / Dec | Register | Architectural Role |
| --- | --- | --- | --- |
| `00` | `0` | **A** | Accumulator (Primary ALU destination & memory gateway) |
| `01` | `1` | **B** | Secondary Register (Implicit ALU source) |
| `10` | `2` | **C** | High Data Page Selector / High Address Pointer ($Y$) |
| `11` | `3` | **D** | Low Data Offset Pointer / Index Register ($X$) |

### Stack Target Mapping (`Bits [2:0]` — Used in `PUSH` & `POP`)

Operands use bits `[2:0]` to select the physical source/destination for stack transfers:

| Binary ID | Target | Operation on `PUSH` / `POP` |
| --- | --- | --- |
| `000` (`0`) | **A** | Transfer Accumulator $A$ to/from Stack |
| `001` (`1`) | **B** | Transfer Secondary Register $B$ to/from Stack |
| `010` (`2`) | **C** | Transfer High Page Register $C$ to/from Stack |
| `011` (`3`) | **D** | Transfer Low Index Register $D$ to/from Stack |
| `100` (`4`) | **PC** | Transfer Program Counter $PC$ to/from Stack |
| `101` (`5`) | **PR** | Transfer Page Register $PR$ to/from Stack |
| `110` (`6`) | *RSVD* | Reserved |
| `111` (`7`) | *RSVD* | Reserved |

---

## 3. Instruction Addressing & Sequential Page Rollover

Instruction addressing operates on an **instruction index** model via hardwired bus alignment.

### Memory Address Formation ($T_0, T_1$)

For instruction fetches, the $W+1$ address bus lines ($\text{Addr}[W:0]$) are driven directly without an ALU cycle:

$$\text{Addr}[W:1] \gets PC[W-1:0]$$

$$\text{Addr}[0] \gets T_1 \quad (0 \text{ during } T_0 \text{ Opcode Fetch}, 1 \text{ during } T_1 \text{ Operand Fetch})$$

### Sequential Page Rollover

When $PC$ rolls over from $2^W - 1$ to $0$, the carry output automatically increments $PR$:

$$PC \gets (PC + 1) \bmod 2^W \quad \mid \quad \text{If } PC \text{ overflows: } PR \gets (PR + 1) \bmod 2^W$$

---

## 4. Universal Escape Sentinel Rules

The upper operand values enforce symmetric hardware escape triggers across memory and control-flow operations:

### Data Memory Accesses (`LOAD`, `STORE`)

| Operand Field State | Addressing Mode | Target RAM Address | Action / Side Effect |
| --- | --- | --- | --- |
| $\text{Operand} \le \text{MAX}-2$ | **Direct Offset** | $C \mathbin{:} \text{Operand}$ | Read/Write RAM at page-local offset $C \mathbin{:} \text{Operand}$ |
| $\text{Operand} == \text{MAX}-1$ | **Register Indirect** | $C \mathbin{:} D$ | Read/Write RAM at $C \mathbin{:} D$ |
| $\text{Operand} == \text{MAX}$ | **Auto-Increment** | $C \mathbin{:} D$ | Read/Write RAM at $C \mathbin{:} D$, then $D \gets (D + 1) \bmod 2^W$ at $T_4$ |

### Control-Transfer Operations (`JMP`, `JZ`, `JC`, `CALL`)

| Operand Field State | Branch Mode | Target Instruction Address | Action / Side Effect |
| --- | --- | --- | --- |
| $\text{Operand} \le \text{MAX}-1$ | **Page-Local Target** | $PR \mathbin{:} \text{Operand}$ | Transfer control within current active page ($PR$ unchanged) |
| $\text{Operand} == \text{MAX}$ | **Absolute Far Target** | $C \mathbin{:} D$ | Transfer control across pages: $PR \gets C, PC \gets D$ |

---

## 5. Primary Opcode Map (16 Opcodes)

Hardware decodes `Opcode[3:0]`; upper bits `Opcode[W-1:4]` are ignored.

| Opcode | Mnemonic | Operand Encoding | Primary Operation | Flag Effect |
| --- | --- | --- | --- | --- |
| `0x0` | **NOP** | Ignored (`0`) | Advance $PC \gets PC + 1$ | Preserved |
| `0x1` | **MOV** | `[3:2]=src, [1:0]=dst` | Register transfer: $R[dst] \gets R[src]$ | Preserved |
| `0x2` | **LOAD** | Offset / `MAX-1` / `MAX` | Read RAM into Accumulator $A$ via direct offset or $C \mathbin{:} D$ | Preserved |
| `0x3` | **STORE** | Offset / `MAX-1` / `MAX` | Write Accumulator $A$ to RAM via direct offset or $C \mathbin{:} D$ | Preserved |
| `0x4` | **ALU** | `[3:0]=sub-opcode` | Execute arithmetic/logic on $A$ and $B$ | **Updated** |
| `0x5` | **JMP** | Target / `MAX` | Unconditional jump (Local offset or Far via $C \mathbin{:} D$) | Preserved |
| `0x6` | **JZ** | Target / `MAX` | Conditional jump if $ZF = 1$ (Local or Far via $C \mathbin{:} D$) | Preserved |
| `0x7` | **JC** | Target / `MAX` | Conditional jump if $CF = 1$ (Local or Far via $C \mathbin{:} D$) | Preserved |
| `0x8` | **CALL** | Target / `MAX` | Subroutine call (Pushes 2-word frame $PR_{ret} \mathbin{:} PC_{ret}$) | Preserved |
| `0x9` | **RET** | Ignored (`0`) | Subroutine return (Pops 2-word frame into $PC$, then $PR$) | Preserved |
| `0xA` | **PUSH** | `[2:0]=Target ID` | Push selected register/system target to Hardware Stack | Preserved |
| `0xB` | **POP** | `[2:0]=Target ID` | Pop Hardware Stack into selected register target | Preserved |
| `0xC` | **IO** | `[2:0]=port, [3]=dir` | Peripheral transfer ($Dir=0 \implies \text{IN to } A, 1 \implies \text{OUT}$) | Preserved |
| `0xD` | **SKP** | `[1:0]=cond` | Skip next instruction if condition evaluates True | Preserved |
| `0xE` | **RSVD** | Reserved (`0`) | Reserved expansion opcode (Behaves strictly as NOP) | Preserved |
| `0xF` | **HLT** | Reserved (`0`) | Freeze execution phase counter until hardware reset | Preserved |

* **SKP Condition Encoding (`[1:0]`):** `00` = Always Skip, `01` = Skip if $ZF=1$, `10` = Skip if $CF=1$, `11` = Skip if $ZF=0$.

---

## 6. Structured Bit-Pattern Hardware ALU Specification

$$\text{Sub-Opcode Field Mapping: } \text{SubOp}[3:0] = \{ \text{NoWrite}, \text{BlockSel}, \text{Ctrl1}, \text{Ctrl0} \}$$

### Non-Writing ALU Operations Contract

For every non-writing ALU operation ($\text{SubOp}[3] = 1$), the condition latches ($ZF, CF$) are computed from the hypothetical $W$-bit bus result at $T_3$ **exactly as if the result had been written to Register $A$**, while Register $A$ remains unmodified.

### Signal Control Mapping

* **Bit 3 ($\text{NoWrite}$):** Inhibit Register $A$ write-enable pulse at $T_4$.
* **Bit 2 ($\text{BlockSel}$):** `0` = Select Full Adder path. `1` = Select Logic / Shift path.
* **Bits [1:0] ($\text{Ctrl}[1:0]$):**
* **On Adder Path ($\text{BlockSel} = 0$):**
* $\text{ForceB0} = \text{SubOp}[1]$ (Forces $B$ inputs to `0`).
* $\text{InvB} = \text{SubOp}[0]$ (Inverts $B$ inputs to $\bar{B}$).
* $C_{in} = \text{SubOp}[1] \oplus \text{SubOp}[0]$ (Drives carry-in for `SUB` and `INC`).


* **On Logic Path ($\text{BlockSel} = 1$):** Selects AND, OR, SHR, or XOR path.



```text
               ┌────────────────────────┐
   A [W-1:0] ──┤  4 Core Circuit Blocks ├────── Main Bus (T3)
   B [W-1:0] ──┤  (Adder, AND, OR, SHR) ├────── Flags (ZF, CF)
               └───────────┬────────────┘
                           │
SubOp[3] (No-Write) ───────┴─────────── Gate Register A Load Signal (T4)

```

### Structured ALU Sub-Opcode Engine

| Sub-Op | Mnemonic | Write $A$? | Operation Pseudocode | Zero Flag ($ZF$) | Carry Flag ($CF$) |
| --- | --- | --- | --- | --- | --- |
| `0x0` | **ADD** | **Yes** | $A \gets A + B$ | $ZF \gets (A'=0)$ | $CF \gets \text{CarryOut}$ |
| `0x1` | **SUB** | **Yes** | $A \gets A - B$ | $ZF \gets (A'=0)$ | $CF \gets \text{NoBorrow}$ |
| `0x2` | **INC** | **Yes** | $A \gets A + 1$ | $ZF \gets (A'=0)$ | $CF \gets \text{CarryOut}$ |
| `0x3` | **DEC** | **Yes** | $A \gets A - 1$ | $ZF \gets (A'=0)$ | $CF \gets \text{NoBorrow}$ |
| `0x4` | **AND** | **Yes** | $A \gets A \land B$ | $ZF \gets (A'=0)$ | $CF \gets 0$ |
| `0x5` | **OR** | **Yes** | $A \gets A \lor B$ | $ZF \gets (A'=0)$ | $CF \gets 0$ |
| `0x6` | **SHR** | **Yes** | $A \gets \lfloor A / 2 \rfloor$ | $ZF \gets (A'=0)$ | $CF \gets A[0]$ (Pre-shift LSB) |
| `0x7` | **XOR** | **Yes** | $A \gets A \oplus B$ | $ZF \gets (A'=0)$ | $CF \gets 0$ |
| `0x8` | **ADD-NW** | **No** | Test $A + B$ | $ZF \gets ((A+B)=0)$ | $CF \gets \text{CarryOut}$ |
| `0x9` | **CMP** | **No** | Test $A - B$ | $ZF \gets (A=B)$ | $CF \gets (A \ge B)$ |
| `0xA` | **INC-NW** | **No** | Test $A + 1$ | $ZF \gets ((A+1)=0)$ | $CF \gets \text{CarryOut}$ |
| `0xB` | **DEC-NW** | **No** | Test $A - 1$ | $ZF \gets ((A-1)=0)$ | $CF \gets \text{NoBorrow}$ |
| `0xC` | **TST** | **No** | Test $A \land B$ | $ZF \gets ((A \land B)=0)$ | $CF \gets 0$ |
| `0xD` | **OR-NW** | **No** | Test $A \lor B$ | $ZF \gets ((A \lor B)=0)$ | $CF \gets 0$ |
| `0xE` | **SHR-NW** | **No** | Test $A \gg 1$ | $ZF \gets (\lfloor A/2 \rfloor=0)$ | $CF \gets A[0]$ (Pre-shift LSB) |
| `0xF` | **TEQ** | **No** | Test $A \oplus B$ | $ZF \gets (A=B)$ | $CF \gets 0$ |

*Note on Implementation Profiles:* `0x7 (XOR)` and `0xF (TEQ)` are normative Base ISA operations. An ultralight implementation profile lacking physical XOR gates designates `0x7` and `0xF` as *Unimplemented / Reserved Extensions* in its profile documentation.

---

## 7. Unified Subroutine Frame Protocol

Every `CALL` instruction creates a standardized **2-word return frame** on the $W$-bit LIFO stack. Every `RET` instruction consumes exactly one 2-word return frame.

```text
 Stack Top (SP-1) -> [ Return PC ]  (PC + 1, or 0 if PC rolled over)
                     [ Return PR ]  (PR, or PR + 1 if PC rolled over)

```

### Page-Boundary Return Context ($PR_{ret} \mathbin{:} PC_{ret}$)

When executing `CALL` at index $PC$:

* $PC_{ret} = (PC + 1) \bmod 2^W$
* $PR_{ret} = (PC == 2^W - 1) \;?\; (PR + 1) \bmod 2^W \;:\; PR$

### Execution Flow

```text
CALL Target / MAX:
  T2: Stack <- PR_ret
  T3: Stack <- PC_ret
  T4: If Operand == MAX: PR <- C, PC <- D
      Else:             PC <- Operand (PR unchanged)

RET:
  T2: PC <- POP()
  T3: PR <- POP()
  T4: Complete (Next PC active)

```

---

## 8. Deterministic 5-Phase Execution Pipeline ($T_0..T_4$)

Every instruction executes across five fixed, deterministic clock phases:

```text
Phases:   |   T0   |   T1   |     T2     |     T3     |     T4     |
Action:   | Opcode | Operand| Execution  | Execution  | State      |
          | Fetch  | Fetch  | Phase 1    | Phase 2    | Commit     |

```

* **Phase $T_0$ (Opcode Fetch):** $\text{IR}_{\text{op}} \gets \text{Memory}[\text{Addr}_{T0}]$.
* **Phase $T_1$ (Operand Fetch):** $\text{IR}_{\text{opr}} \gets \text{Memory}[\text{Addr}_{T1}]$.
* **Phase $T_2$ (Execution Phase 1):** Sample registers; evaluate ALU operations, skip conditions, or execute Stack Phase 1 (`CALL` pushes $PR_{ret}$, `RET` pops $PC$).
* **Phase $T_3$ (Execution Phase 2):** RAM read/write cycles, I/O transfers, or Stack Phase 2 (`CALL` pushes $PC_{ret}$, `RET` pops $PR$).
* **Phase $T_4$ (Architectural Commit):**
* Default PC Advancement: $PC \gets PC + 1$ (Architectural effect of `SKP`: $PC \gets PC + 2$).
* Control Flow Branch Commit: If `JMP`/`CALL` taken, commit target to $PC$ (Local) or $PR \mathbin{:} PC$ (Far via $C \mathbin{:} D$).
* Auto-Increment Commit: If `Operand == MAX` on memory access, commit $D \gets (D + 1) \bmod 2^W$.



---

## Architectural Invariant Summary

$$\begin{aligned} \text{Datapath Width} &= W \text{ bits } (W \ge 4) \\ \text{Primary Registers} &= A, B, C, D \text{ (4 general/address registers)} \\ \text{Instruction Size} &= 2W \text{ bits } ([W\text{-bit Opcode}] \mathbin{:} [W\text{-bit Operand}]) \\ \text{Primary Opcodes} &= 16 \text{ primary opcodes} \\ \text{Clock Engine} &= 5 \text{ fixed, deterministic phases } (T_0..T_4) \\ \text{Stack Primitive} &= W\text{-bit abstract LIFO ring buffer} \\ \text{Subroutine Frame} &= 2 \text{ words } (PR_{ret} \mathbin{:} PC_{ret}) \text{ unconditionally} \\ \text{Instruction Fetch} &= \text{Hardwired wire shift } (PC \to \text{Addr}) \\ \text{Addressing Escapes} &= \text{Universal Sentinels } (\text{MAX}-1 \implies [C:D], \text{MAX} \implies \text{Far } [C:D] \text{ or } [C:D]+) \\ \text{ALU Control} &= \text{Direct hardware signal mapping } \{ \text{NoWrite}, \text{BlockSel}, \text{Ctrl1}, \text{Ctrl0} \} \end{aligned}$$
