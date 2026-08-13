# μ-Core ISA Specification (v4.6.0 Canonical Standard)

**Status:** Fixed Canonical Reference Standard (Normative)

**Target Profile:** Discrete Transistors, Relays, CMOS/TTL, FPGA ($\mu 4, \mu 8, \mu 16$)

---

## 1. Architectural Philosophy & Memory Topology

The **μ-Core v4.6.0 ISA** is a parameterized, technology-independent architecture designed for maximum physical hardware minimalism, deterministic timing, and strict orthogonal execution. Datapath width $W$ ($W \ge 4$) defines storage units, registers, memory boundaries, and pointer locations. Every instruction occupies exactly **2 Storage Units ($2W$ bits)**: `[Opcode: W] [Operand: W]`.

### Unified Memory Model

Instruction fetches and data accesses address the **same unified $2W$-bit memory space**. Program storage is directly writable through standard `STORE` instructions, and the architecture imposes no separate instruction/data physical memory distinction.

### Parameterized Architectural Matrix ($W$)

| Architectural Property | Parameterized Definition | $\mu 4$ ($W=4$) | $\mu 8$ ($W=8$) | $\mu 16$ ($W=16$) |
| --- | --- | --- | --- | --- |
| **Register & Operand Width** | $W$ bits | 4 bits | 8 bits | 16 bits |
| **Instruction Size** | $2W$ bits ($2$ Storage Units) | 8 bits | 16 bits | 32 bits |
| **Memory Capacity per Page ($PR$)** | $2^W$ storage units ($2^{W-1}$ inst) | 16 nibbles (8 inst) | 256 bytes (128 inst) | 65,536 words (32,768 inst) |
| **Total Unified Memory ($C:D$)** | $2W$ bits ($2^{2W}$ locations) | 256 nibbles ($128 \text{ B}$) | 64 KiB ($64 \text{ KiB}$) | 4 GiB words (**8 GiB**) |
| **Indirect RAM Sentinel (`MAX-1`)** | $2^W - 2$ | `0xE` | `$FE` | `$FFFE` |
| **Auto-Increment / Far Sentinel (`MAX`)** | $2^W - 1$ | `0xF` | `$FF` | `$FFFF` |

---

## 2. Programmer State & Reserved Encodings

### Architectural State Tuple ($\mathcal{S}$)

$$\mathcal{S} = \langle A, B, C, D, PC, PR, \text{Stack}[], \text{Memory}[] \rangle$$

* **Architectural PC:** Holds an even storage-unit address ($0, 2, 4, \dots, 2^W - 2$).
* **Stack Model:** The stack is defined purely by its abstract $W$-bit circular LIFO behavior across `PUSH`, `POP`, `CALL`, and `RET`. Internal stack pointers or physical storage topology are non-architectural implementation details.

### Condition Latches ($\mathcal{C}$)

Condition flags (**$ZF$**, **$CF$**) exist as **persistent hardware latches** on the ALU board:

* **Persistence:** $ZF$ and $CF$ hold state across non-ALU operations (`MOV`, `LOAD`, `STORE`, `PUSH`, `POP`, `JMP`, `SKP`). They are updated *only* when an `ALU` instruction executes.
* **Isolation:** They are not general-purpose storage, are not addressable, and are never saved or restored by `CALL` or `RET`.

### Reserved Encodings Policy

Any reserved opcode (`0xE`) or reserved operand target (`110`, `111` in `PUSH`/`POP`) shall execute as a non-architectural `NOP`, leaving all architectural state and condition flags preserved and unchanged.

### MOV Register Mapping (`Bits [3:2]=src, [1:0]=dst`)

| Binary ID | Hex / Dec | Register | Architectural Role |
| --- | --- | --- | --- |
| `00` | `0` | **A** | Accumulator (Primary ALU destination & memory gateway) |
| `01` | `1` | **B** | Secondary Register (Implicit ALU source) |
| `10` | `2` | **C** | High Data Page Selector / High Address Pointer ($Y$) |
| `11` | `3` | **D** | Low Data Offset Pointer / Index Register ($X$) |

### Stack Target Mapping (`Bits [2:0]` — Used in `PUSH` & `POP`)

| Binary ID | Target | Operation on `PUSH` / `POP` |
| --- | --- | --- |
| `000` (`0`) | **A** | Transfer Accumulator $A$ to/from LIFO Stack |
| `001` (`1`) | **B** | Transfer Secondary Register $B$ to/from LIFO Stack |
| `010` (`2`) | **C** | Transfer High Page Register $C$ to/from LIFO Stack |
| `011` (`3`) | **D** | Transfer Low Index Register $D$ to/from LIFO Stack |
| `100` (`4`) | **PC** | Transfer Program Counter $PC$ (even memory address) to/from Stack |
| `101` (`5`) | **PR** | Transfer Page Register $PR$ to/from LIFO Stack |
| `110` (`6`) | *RSVD* | Reserved (Acts as NOP) |
| `111` (`7`) | *RSVD* | Reserved (Acts as NOP) |

---

## 3. Instruction Addressing & Fetch Mechanics

### Architectural PC vs. Transient Fetch Counter State

* **Architectural $PC$:** At instruction boundaries ($T_0$ and $T_4$), $PC$ strictly represents the even starting location of an instruction ($2k$).
* **Transient Fetch State:** During fetch phases, physical counter transitions update $PC$ dynamically:

$$\text{At } T_0: \text{Addr} = PC \ (2k) \quad \implies PC \text{ clocks to } 2k+1 \text{ at end of } T_0$$


$$\text{At } T_1: \text{Addr} = PC \ (2k+1) \implies PC \text{ clocks to } 2k+2 \text{ at end of } T_1$$


* **Zero-Arithmetic Return Context:** By phase $T_2$, $PC$ already sits at $2k+2$, naturally forming $PC_{ret}$ for `CALL` operations without requiring adder hardware.

### Generalized PC Advancement & Rollover

$$PC' = (PC + \Delta) \bmod 2^W$$

* **Ordinary Advancement ($\Delta = 2$):** Standard two-step fetch advancement ($2k \to 2k+2$). If $PC + 2 \ge 2^W$, page carry automatically increments $PR \gets (PR + 1) \bmod 2^W$.
* **Taken Skip ($\Delta = 4$):** Executed by `SKP` when a condition evaluates true, sending an additional $+2$ pulse to $PC$ at $T_4$ (advancing $2k+2 \to 2k+4$). Page carry increments $PR$ if rollover occurs.

---

## 4. Universal Escape Sentinel Rules

### Data Memory Accesses (`LOAD`, `STORE`)

| Opcode | Operand Field State | Addressing Mode | Target RAM Address / Action |
| --- | --- | --- | --- |
| **`LOAD`** | $\text{Operand} \le \text{MAX}-2$ | **Immediate Load** | $A \gets \text{Operand}$ *(No RAM Read)* |
|  | $\text{Operand} == \text{MAX}-1$ | **Register Indirect** | Read RAM at $C \mathbin{:} D$ into $A$ |
|  | $\text{Operand} == \text{MAX}$ | **Auto-Increment** | Read RAM at $C \mathbin{:} D$ into $A$; $D \gets (D + 1) \bmod 2^W$ at $T_4$ |
| **`STORE`** | $\text{Operand} \le \text{MAX}-2$ | **Direct Offset** | Write Accumulator $A$ to RAM at $C \mathbin{:} \text{Operand}$ |
|  | $\text{Operand} == \text{MAX}-1$ | **Register Indirect** | Write Accumulator $A$ to RAM at $C \mathbin{:} D$ |
|  | $\text{Operand} == \text{MAX}$ | **Auto-Increment** | Write Accumulator $A$ to RAM at $C \mathbin{:} D$; $D \gets (D + 1) \bmod 2^W$ at $T_4$ |

* **Auto-Increment Page Wrapping:** The auto-increment operation modifies $D$ via $D \gets (D + 1) \bmod 2^W$. Increments wrap strictly within the current page ($C$ remains unchanged).

### Control-Transfer Operations (`JMP`, `JZ`, `JC`, `CALL`)

Local branch target operands MUST be even memory addresses ($0, 2, 4, \dots, \text{MAX}-1$). Odd local target values (excluding `MAX`) are reserved.

| Operand Field State | Branch Mode | Target Memory Address | Action / Side Effect |
| --- | --- | --- | --- |
| $\text{Operand} \le \text{MAX}-1$ (Even) | **Page-Local Target** | $PR \mathbin{:} \text{Operand}$ | Branch to even address in active page ($PR$ unchanged) |
| $\text{Operand} == \text{MAX}$ (Odd) | **Absolute Far Target** | $C \mathbin{:} D$ | Far transfer: $PR \gets C, PC \gets D$ |

---

## 5. Primary Opcode Map (16 Opcodes)

Hardware decodes `Opcode[3:0]`; upper bits `Opcode[W-1:4]` are ignored.

| Opcode | Mnemonic | Operand Encoding | Primary Operation | Flag Effect |
| --- | --- | --- | --- | --- |
| `0x0` | **NOP** | Ignored (`0`) | Advance $PC \gets PC + 2$ | Preserved |
| `0x1` | **MOV** | `[3:2]=src, [1:0]=dst` | Register transfer: $R[dst] \gets R[src]$ | Preserved |
| `0x2` | **LOAD** | Immediate / `MAX-1` / `MAX` | Load Immediate constant or read RAM into $A$ via $C \mathbin{:} D$ | Preserved |
| `0x3` | **STORE** | Offset / `MAX-1` / `MAX` | Write $A$ to RAM via direct offset or $C \mathbin{:} D$ | Preserved |
| `0x4` | **ALU** | `[3:0]=sub-opcode` | Execute arithmetic/logic on $A$ and $B$ | **Updated** |
| `0x5` | **JMP** | Target / `MAX` | Unconditional jump (Local even offset or Far via $C \mathbin{:} D$) | Preserved |
| `0x6` | **JZ** | Target / `MAX` | Conditional jump if $ZF = 1$ (Local even offset or Far via $C \mathbin{:} D$) | Preserved |
| `0x7` | **JC** | Target / `MAX` | Conditional jump if $CF = 1$ (Local even offset or Far via $C \mathbin{:} D$) | Preserved |
| `0x8` | **CALL** | Target / `MAX` | Subroutine call (Pushes 2-word frame $PR_{ret} \mathbin{:} PC_{ret}$) | Preserved |
| `0x9` | **RET** | Ignored (`0`) | Subroutine return (Pops 2-word frame into $PC$, then $PR$) | Preserved |
| `0xA` | **PUSH** | `[2:0]=Target ID` | Push selected register/system target to Hardware Stack | Preserved |
| `0xB` | **POP** | `[2:0]=Target ID` | Pop Hardware Stack into selected register target | Preserved |
| `0xC` | **IO** | `[2:0]=port, [3]=dir` | $W$-bit peripheral transfer ($Dir=0 \implies \text{IN to } A, 1 \implies \text{OUT}$) | Preserved |
| `0xD` | **SKP** | `[1:0]=cond` | Skip next instruction if condition evaluates True | Preserved |
| `0xE` | **RSVD** | Reserved (`0`) | Reserved expansion opcode (Behaves strictly as NOP) | Preserved |
| `0xF` | **HLT** | Reserved (`0`) | Freeze execution phase counter until hardware reset | Preserved |

### Peripheral I/O Specification (`IO`)

`IO` performs $W$-bit parallel data transfers across 8 addressable peripheral ports (`port = Bits [2:0]`):

* **`IO IN` ($Dir = 0$):** Sample $W$-bit data from peripheral Port `[2:0]` during $T_2$; commit to Accumulator $A$ at $T_3$.
* **`IO OUT` ($Dir = 1$):** Drive Accumulator $A$ data onto peripheral Port `[2:0]` during $T_3$.

---

## 6. Structured Bit-Pattern Hardware ALU Specification

$$\text{Sub-Opcode Field Mapping: } \text{SubOp}[3:0] = \{ \text{NoWrite}, \text{BlockSel}, \text{Ctrl1}, \text{Ctrl0} \}$$

### Non-Writing ALU Operations Contract

For every non-writing ALU operation ($\text{SubOp}[3] = 1$), condition latches ($ZF, CF$) are computed from the $W$-bit bus result at $T_3$ **exactly as if written to Register $A$**, while Register $A$ remains unmodified.

### Signal Control Mapping

* **Bit 3 ($\text{NoWrite}$):** Inhibit Register $A$ write-enable pulse at $T_4$.
* **Bit 2 ($\text{BlockSel}$):** `0` = Select Full Adder path. `1` = Select Logic / Shift path.
* **Bits [1:0] ($\text{Ctrl}[1:0]$):**
* **On Adder Path ($\text{BlockSel} = 0$):** $\text{ForceB0} = \text{SubOp}[1]$, $\text{InvB} = \text{SubOp}[0]$, $C_{in} = \text{SubOp}[1] \oplus \text{SubOp}[0]$.
* **On Logic Path ($\text{BlockSel} = 1$):** Selects AND, OR, SHR, or XOR.



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

---

## 7. Unified Subroutine Frame Protocol

Every `CALL` instruction creates a standardized **2-word return frame** on the $W$-bit LIFO stack. Every `RET` instruction consumes exactly one 2-word return frame.

```text
 Stack Top (SP-1) -> [ Return PC ]  (Direct RAM address: PC + 2, or 0 if rolled over)
                     [ Return PR ]  (PR, or PR + 1 if PC rolled over)

```

### Page-Boundary Return Context ($PR_{ret} \mathbin{:} PC_{ret}$)

When executing `CALL` at memory location $PC$:

* $PC_{ret} = (PC + 2) \bmod 2^W$
* $PR_{ret} = (PC == 2^W - 2) \;?\; (PR + 1) \bmod 2^W \;:\; PR$

---

## 8. Deterministic 5-Phase Execution Pipeline ($T_0..T_4$)

Every instruction executes across five fixed, deterministic clock phases driven by a 1-hot ring counter:

```text
Phases:   |   T0   |   T1   |     T2     |     T3     |     T4     |
Action:   | Opcode | Operand| Execution  | Execution  | State      |
          | Fetch  | Fetch  | Phase 1    | Phase 2    | Commit     |

```

* **Phase $T_0$ (Opcode Fetch):** $\text{IR}_{\text{op}} \gets \text{Memory}[\text{Addr}_{T0}]$ where $\text{Addr}_{T0} = PC$. ($PC$ clocks to $2k+1$ at end of $T_0$).
* **Phase $T_1$ (Operand Fetch):** $\text{IR}_{\text{opr}} \gets \text{Memory}[\text{Addr}_{T1}]$ where $\text{Addr}_{T1} = PC$. ($PC$ clocks to $2k+2$ at end of $T_1$).
* **Phase $T_2$ (Execution Phase 1):** Sample registers; evaluate ALU operations, skip conditions, sample `IO IN`, or execute Stack Phase 1 (`CALL` pushes $PR_{ret}$, `RET` pops $PC$).
* **Phase $T_3$ (Execution Phase 2):** RAM read/write cycles, drive `IO OUT`, commit `IO IN` to $A$, or Stack Phase 2 (`CALL` pushes $PC_{ret}$, `RET` pops $PR$).
* **Phase $T_4$ (Architectural Commit):**
* Default PC Advancement: $PC$ already sits at $2k+2$. (Taken `SKP` sends extra pulse: $PC \gets PC + 2$).
* Control Flow Branch Commit: If `JMP`/`CALL` taken, commit target to $PC$ (Local) or $PR \mathbin{:} PC$ (Far via $C \mathbin{:} D$).
* Auto-Increment Commit: If `Operand == MAX` on memory access, pulse $D$ count pin: $D \gets (D + 1) \bmod 2^W$.


* **Halt Behavior (`HLT`):** Freezes execution phase counter ($T_0..T_4$). Preserves all architectural state ($A, B, C, D, PC, PR$, condition latches, stack, RAM) unchanged until hardware reset.

---

## Architectural Invariant Summary (v4.6.0 Final Reference)

$$\begin{aligned} \text{Datapath Width} &= W \text{ bits } (W \ge 4) \\ \text{Primary Registers} &= A, B, C, D \text{ (4 general/address registers)} \\ \text{Instruction Size} &= 2W \text{ bits } ([W\text{-bit Opcode}] \mathbin{:} [W\text{-bit Operand}]) \\ \text{Unified Memory} &= 2W\text{-bit unified space } (C:D \implies \text{Full Space}, PR:PC \implies \text{Code Page}) \\ \text{Program Counter } (PC) &= \text{Direct even memory address } (0, 2, 4, \dots, 2^W - 2) \\ \text{Page Capacity} &= 2^W \text{ storage units } (2^{W-1} \text{ instructions per page } PR) \\ \text{Clock Engine} &= 5 \text{ fixed, deterministic phases } (T_0..T_4) \\ \text{Stack Primitive} &= W\text{-bit abstract LIFO ring buffer} \\ \text{Subroutine Frame} &= 2 \text{ words } (PR_{ret} \mathbin{:} PC_{ret}) \text{ unconditionally} \\ \text{Condition Latches} &= \text{Persistent } ZF, CF \text{ latches (updated only by ALU operations)} \\ \text{ALU Control} &= \text{Direct hardware signal mapping } \{ \text{NoWrite}, \text{BlockSel}, \text{Ctrl1}, \text{Ctrl0} \} \\ \text{LOAD Escapes} &= \le \text{MAX}-2 \implies \text{Immediate}, \quad \text{MAX}-1 \implies [C:D], \quad \text{MAX} \implies [C:D]+ \\ \text{STORE Escapes} &= \le \text{MAX}-2 \implies \text{Offset } [C:\text{Opr}], \quad \text{MAX}-1 \implies [C:D], \quad \text{MAX} \implies [C:D]+ \\ \text{Branch Escapes} &= \le \text{MAX}-1 \text{ (Even)} \implies \text{Page Local } (PR \mathbin{:} \text{Opr}), \quad \text{MAX} \text{ (Odd)} \implies \text{Far Target } (C \mathbin{:} D) \end{aligned}$$