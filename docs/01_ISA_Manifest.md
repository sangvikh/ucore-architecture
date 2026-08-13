# μ-Core ISA Specification (v4.2.0)

**Status:** Canonical Reference Standard (Normative)

**Target Profile:** Discrete Transistors, Relays, CMOS/TTL, FPGA ($\mu 4, \mu 8, \mu 16$)

---

### 1. Architectural Philosophy & Parameterized Model ($W$)

The **μ-Core v4.2.0 ISA** is a parameterized, technology-independent architecture designed for maximum physical hardware minimalism, deterministic timing, and strict orthogonal execution. Datapath width $W$ ($W \ge 4$) defines storage units, registers, and memory boundaries. Every instruction occupies exactly **2 Storage Units ($2W$ bits)**: `[Opcode: W] [Operand: W]`.

| Architectural Property | Parameterized Definition | $\mu 4$ ($W=4$) | $\mu 8$ ($W=8$) | $\mu 16$ ($W=16$) |
| --- | --- | --- | --- | --- |
| **Register & Operand Width** | $W$ bits | 4 bits | 8 bits | 16 bits |
| **Instruction Size** | $2W$ bits ($2$ Storage Units) | 8 bits | 16 bits | 32 bits |
| **Instruction Page Capacity** | $2^{W-1}$ instructions | 8 instructions | 128 instructions | 32,768 instructions |
| **Data Memory Space ($C:D$)** | $2W$ bits ($2^{2W}$ locations) | 256 nibbles | 64 KiB | 4 GiB |
| **Indirect Sentinel (`MAX-1`)** | $2^W - 2$ | `0xE` | `$FE` | `$FFFE` |
| **Auto-Increment Sentinel (`MAX`)** | $2^W - 1$ | `0xF` | `$FF` | `$FFFF` |

---

### 2. Programmer State & Register Encodings

#### Architectural State Tuple ($\mathcal{S}$)

$$\mathcal{S} = \langle A, B, C, D, PC, PR, SP, \text{FLAGS}, \text{Stack}[], \text{Memory}[] \rangle$$

Condition flags ($ZF, CF$) exist purely as discrete hardware latches on the ALU board and are not connected to the main $W$-bit data bus or stack.

#### MOV Register Mapping (`Bits [3:0]`)

`MOV` uses bits `[3:2]` for Source Register (`src`) and bits `[1:0]` for Destination Register (`dst`):

| Binary ID | Hex / Dec | Register | Architectural Role |
| --- | --- | --- | --- |
| `00` | `0` | **A** | Accumulator (Primary ALU destination & memory gateway) |
| `01` | `1` | **B** | Secondary Register (Implicit ALU source) |
| `10` | `2` | **C** | High Data Page Selector / High Address Pointer ($Y$) |
| `11` | `3` | **D** | Low Data Offset Pointer / Index Register ($X$) |

#### Stack Target Mapping (`Bits [2:0]` — Used in `PUSH` & `POP`)

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

### 3. Instruction Addressing & Sequential Page Rollover

Instruction addressing operates on an **instruction index** model via hardwired bus alignment.

#### Memory Address Formation ($T_0, T_1$)

For instruction fetches, the $W+1$ address bus lines ($\text{Addr}[W:0]$) are driven directly without an ALU cycle:

$$\text{Addr}[W:1] \gets PC[W-1:0]$$

$$\text{Addr}[0] \gets T_1 \quad (0 \text{ during } T_0 \text{ Opcode Fetch}, 1 \text{ during } T_1 \text{ Operand Fetch})$$

#### Sequential Page Rollover

When $PC$ rolls over from $2^W - 1$ to $0$, the carry output automatically increments $PR$:

$$PC \gets (PC + 1) \bmod 2^W \quad \mid \quad \text{If } PC \text{ overflows: } PR \gets (PR + 1) \bmod 2^W$$

---

### 4. Dual-Sentinel Memory Access Modes

For memory accesses (`LOAD`, `STORE`), the upper two operand values act as hardware escape triggers:

| Operand Field State | Addressing Mode | Target RAM Address | Action / Side Effect |
| --- | --- | --- | --- |
| $\text{Operand} \le \text{MAX}-2$ | **Direct Offset** | $C \mathbin{:} \text{Operand}$ | Immediate load or page-local RAM access |
| $\text{Operand} == \text{MAX}-1$ | **Register Indirect** | $C \mathbin{:} D$ | Read/Write RAM at $C \mathbin{:} D$ |
| $\text{Operand} == \text{MAX}$ | **Auto-Increment** | $C \mathbin{:} D$ | Read/Write RAM at $C \mathbin{:} D$, then $D \gets (D + 1) \bmod 2^W$ at $T_4$ |

---

### 5. Primary Opcode Map (16 Opcodes)

Hardware decodes `Opcode[3:0]`; upper bits `Opcode[W-1:4]` are ignored.

| Opcode | Mnemonic | Operand Encoding | Primary Operation | Flag Effect |
| --- | --- | --- | --- | --- |
| `0x0` | **NOP** | Ignored (`0`) | Advance $PC \gets PC + 1$ | Preserved |
| `0x1` | **MOV** | `[3:2]=src, [1:0]=dst` | Register transfer: $R[dst] \gets R[src]$ | Preserved |
| `0x2` | **LOAD** | Offset / `MAX-1` / `MAX` | Immediate load or RAM read into Accumulator $A$ | Preserved |
| `0x3` | **STORE** | Offset / `MAX-1` / `MAX` | Write Accumulator $A$ to RAM offset or $C \mathbin{:} D$ | Preserved |
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

### 6. Minimalist Hardware ALU Specification

The mandatory ALU hardware requires only a single $W$-bit full adder, bitwise AND/OR logic gates, and a 1-bit right-wire shift ($A[i+1] \to A[i]$).

$$\text{Sub-Opcode Format: } \text{SubOp}[3:0] = \{ \text{WriteDisable}, \text{OpID}[2:0] \}$$

Setting Bit 3 (`SubOp[3] = 1`) inhibits the $T_4$ write-enable pulse to Register $A$, enabling non-destructive comparison operations without additional arithmetic hardware.

```text
               ┌────────────────────────┐
   A [W-1:0] ──┤                        ├────── Main Bus (T3)
               │   4-Op Core Logic      │
   B [W-1:0] ──┤   (ADD, SUB, AND, OR)  ├────── Flags (ZF, CF)
               └───────────┬────────────┘
                           │
SubOp[3] (No-Write) ───────┴─────────── Gate Register A Load Signal (T4)

```

#### Mandatory Sub-Opcode Engine

| Sub-Op | Mnemonic | Write A? | Operational Pseudocode | Flags ($ZF, CF$) |
| --- | --- | --- | --- | --- |
| `0x0` | **ADD** | **Yes** | $A \gets (A + B) \bmod 2^W$ | $ZF \gets (A'=0), CF \gets \text{CarryOut}$ |
| `0x1` | **SUB** | **Yes** | $A \gets (A - B + 2^W) \bmod 2^W$ | $ZF \gets (A'=0), CF \gets \text{NoBorrow}$ |
| `0x2` | **AND** | **Yes** | $A \gets A \land B$ | $ZF \gets (A'=0), CF \gets 0$ |
| `0x3` | **OR** | **Yes** | $A \gets A \lor B$ | $ZF \gets (A'=0), CF \gets 0$ |
| `0x4` | **SHR** | **Yes** | $A \gets \lfloor A / 2 \rfloor$ | $ZF \gets (A'=0), CF \gets A[0]$ |
| `0x9` | **CMP** | **No** | Evaluate $A - B$ | $ZF \gets (A = B), CF \gets (A \ge B)$ |
| `0xA` | **TST** | **No** | Evaluate $A \land B$ | $ZF \gets ((A \land B) == 0), CF \gets 0$ |

#### Software Synthesis Rules

* **Left Shift (`SHL A`):** Executed via `ADD A` with $B = A$.
* **Increment (`INC A`):** Software sets $B = 1$, then executes `ADD`.
* **Decrement (`DEC A`):** Software sets $B = 1$, then executes `SUB`.
* **Logical NOT (`NOT A`):** Executed via `SUB` from `MAX` ($2^W - 1$).

*Note: Sub-opcodes `0x5..0x8` and `0xB..0xF` are reserved for hardware expansion. In minimal builds, reserved sub-opcodes behave as non-destructive `TST` or `NOP`.*

---

### 7. Unified Subroutine Frame Protocol

Every `CALL` instruction creates a standardized **2-word return frame** on the $W$-bit LIFO stack. Every `RET` instruction consumes exactly one 2-word return frame.

```text
 Stack Top (SP-1) -> [ Return PC ]  (PC + 1, or 0 if PC rolled over)
                     [ Return PR ]  (PR, or PR + 1 if PC rolled over)

```

#### Page-Boundary Return Context ($PR_{ret} \mathbin{:} PC_{ret}$)

When executing `CALL` at index $PC$:

* $PC_{ret} = (PC + 1) \bmod 2^W$
* $PR_{ret} = (PC == 2^W - 1) \;?\; (PR + 1) \bmod 2^W \;:\; PR$

#### Execution Flow

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

### 8. Deterministic 5-Phase Execution Pipeline ($T_0..T_4$)

Every instruction executes across five fixed clock phases:

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
* Default PC Advancement: $PC \gets PC + 1$ (If `SKP` condition met: $PC \gets PC + 2$).
* Control Flow Branch Commit: If `JMP`/`CALL` taken, commit target to $PC$ (Local) or $PR \mathbin{:} PC$ (Far via $C \mathbin{:} D$).
* Auto-Increment Commit: If `Operand == MAX` on memory access, commit $D \gets (D + 1) \bmod 2^W$.
