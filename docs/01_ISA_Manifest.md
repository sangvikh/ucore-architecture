# μ-Core ISA Specification (v4.6.0 Final Canonical Standard)

**Status:** Fixed Canonical Reference Standard (Normative)

**Target Profile:** Discrete Transistors, Relays, CMOS/TTL, FPGA ($\mu4, \mu8, \mu16$)

---

## 1. Architectural Philosophy & Memory Topology

The **$\mu$-Core v4.6.0 ISA** is a parameterized, technology-independent architecture designed for physical hardware minimalism, deterministic timing, and strict orthogonal execution. Datapath width $W$ ($W \ge 4$) defines storage units, registers, memory boundaries, and pointer locations. Every instruction occupies exactly **2 Storage Units ($2W$ bits)**: `[Opcode: W] [Operand: W]`.

### Unified Memory Model

Instruction fetches and data accesses share a single unified **$2W$-bit address space** pointing to **$W$-bit storage units**. Addresses are formed as $\text{High}[W] : \text{Low}[W]$, providing $2^{2W}$ total addressable locations. Program storage is directly writable through standard `STORE` instructions.

### Parameterized Architectural Matrix ($W$)

| Architectural Property | Parameterized Definition | $\mu4$ ($W=4$) | $\mu8$ ($W=8$) | $\mu16$ ($W=16$) |
| --- | --- | --- | --- | --- |
| **Storage Unit & Register Width** | $W$ bits | 4 bits | 8 bits | 16 bits |
| **Instruction Size** | $2W$ bits (2 Storage Units) | 8 bits | 16 bits | 32 bits |
| **Memory Capacity per Page (`PR`)** | $2^W$ storage units ($2^{W-1}$ inst) | 16 nibbles (8 inst) | 256 bytes (128 inst) | 65,536 words (32,768 inst) |
| **Total Addressable Memory Space** | $2W$-bit address ($2^{2W}$ locations) | 256 nibbles (128 B) | 64 KiB (64 KiB) | 4 GiB words (**8 GiB**) |
| **Indirect RAM Sentinel (`MAX-1`)** | $2^W - 2$ | `0xE` | `$FE` | `$FFFE` |
| **Auto-Increment / Far Sentinel (`MAX`)** | $2^W - 1$ | `0xF` | `$FF` | `$FFFF` |

---

## 2. Programmer State & Reserved Encodings

### Architectural State Tuple ($S$)

```text
S = < A, B, C, D, PC, PR, Stack[], Memory[] >

```

* **Program Counter (`PC`):** Holds a $W$-bit program storage-unit address offset ($0, 1, 2, \dots, 2^W - 1$). Instruction alignment is purely a software convention; no hardware alignment checking or restriction is imposed.
* **Stack Model (Behavioral Contract):** Defined strictly by its abstract $W$-bit circular Last-In, First-Out (LIFO) behavior across `PUSH`, `POP`, `CALL`, and `RET`. Internal physical implementation details (stack pointers, physical depth, RAM structures) are non-architectural and unconstrained.

### Condition Latches ($C$)

Condition flags (**ZF**, **CF**) exist as **persistent hardware latches**:

* **Persistence:** `ZF` and `CF` hold state across non-ALU operations (`MOV`, `LOAD`, `STORE`, `PUSH`, `POP`, `JMP`, `SKP`). They are updated *only* when an `ALU` instruction executes.
* **Isolation:** They are not general-purpose storage, are not addressable, and are never saved or restored by `CALL` or `RET`.

### Reserved Encodings Policy

Any reserved opcode (`0xE`) or reserved operand target (`110`, `111` in `PUSH`/`POP`) executes as a non-architectural `NOP`, leaving all architectural state and condition flags preserved and unchanged.

### MOV Register Mapping (`Bits [3:2]=dst, [1:0]=src`) — Intel Convention

| Binary ID | Hex / Dec | Register | Architectural Role |
| --- | --- | --- | --- |
| `00` | `0` | **A** | Accumulator (Primary ALU destination & memory gateway) |
| `01` | `1` | **B** | Secondary Register (Implicit ALU source) |
| `10` | `2` | **C** | High Data Page Selector / High Address Pointer (`Y`) |
| `11` | `3` | **D** | Low Data Offset Pointer / Index Register (`X`) |

### Stack Target Mapping (`Bits [2:0]` — Used in `PUSH` & `POP`)

| Binary ID | Target | Operation on `PUSH` / `POP` |
| --- | --- | --- |
| `000` (`0`) | **A** | Transfer Accumulator `A` to/from LIFO Stack |
| `001` (`1`) | **B** | Transfer Secondary Register `B` to/from LIFO Stack |
| `010` (`2`) | **C** | Transfer High Page Register `C` to/from LIFO Stack |
| `011` (`3`) | **D** | Transfer Low Index Register `D` to/from LIFO Stack |
| `100` (`4`) | **PC** | Transfer Program Counter `PC` to/from LIFO Stack |
| `101` (`5`) | **PR** | Transfer Page Register `PR` to/from LIFO Stack |
| `110` (`6`) | *RSVD* | Reserved (Acts as NOP) |
| `111` (`7`) | *RSVD* | Reserved (Acts as NOP) |

---

## 3. Instruction Fetch Engine & Address Mechanics

### Universal Fetch Logic

At instruction boundary `T0`, `PC` holds starting address offset $N$. The fetch engine reads two contiguous storage units starting at $N$:

```text
Phase T0:  IR_op  <- Memory[PR : PC] ;  PC_INC pulse -> PC becomes N+1
Phase T1:  IR_opr <- Memory[PR : PC] ;  PC_INC pulse -> PC becomes N+2

```

### Universal PC Increments & Page Carry Propagation

Every `PC_INC` pulse—whether generated during fetch (`T0`, `T1`) or skip execution (`T2`, `T3`)—operates on the same counter line:

$$\text{PC}' = (\text{PC} + 1) \bmod 2^W$$

When `PC` overflows from $2^W - 1 \rightarrow 0$, a page carry pulse automatically increments `PR`:

$$\text{PR}' = (\text{PR} + 1) \bmod 2^W$$

---

## 4. Universal Escape Sentinel Rules

### Data Memory Accesses (`LOAD`, `STORE`)

| Opcode | Operand Field State | Addressing Mode | Target RAM Address / Action |
| --- | --- | --- | --- |
| **`LOAD`** | `Operand <= MAX - 2` | **Immediate Load** | `A <- Operand` *(No RAM Read)* |
|  | `Operand == MAX - 1` | **Register Indirect** | Read RAM at `C:D` into `A` |
|  | `Operand == MAX` | **Auto-Increment** | Read RAM at `C:D` into `A`; `D <- (D + 1) mod 2^W` at `T4` |
| **`STORE`** | `Operand <= MAX - 2` | **Direct Offset** | Write Accumulator `A` to RAM at `C:Operand` |
|  | `Operand == MAX - 1` | **Register Indirect** | Write Accumulator `A` to RAM at `C:D` |
|  | `Operand == MAX` | **Auto-Increment** | Write Accumulator `A` to RAM at `C:D`; `D <- (D + 1) mod 2^W` at `T4` |

* **Auto-Increment Page Wrapping:** The auto-increment operation modifies `D` via $D \leftarrow (D + 1) \bmod 2^W$. Increments wrap strictly within the current page (`C` remains unchanged).

### Control-Transfer Operations (`JMP`, `JZ`, `JC`, `CALL`)

Local control-transfer operands designate $W$-bit program offsets. No hardware alignment restriction is imposed.

| Operand Field State | Branch Mode | Target Memory Address | Action / Side Effect |
| --- | --- | --- | --- |
| `Operand <= MAX - 1` | **Page-Local Target** | `PR:Operand` | Branch to offset `Operand` in active page (`PR` unchanged) |
| `Operand == MAX` | **Absolute Far Target** | `C:D` | Far transfer: `PR <- C`, `PC <- D` |

---

## 5. Primary Opcode Map & Sub-Decoders (16 Opcodes)

Hardware decodes `Opcode[3:0]`; upper bits `Opcode[W-1:4]` are ignored.

| Opcode | Mnemonic | Operand Encoding | Primary Operation (Intel Convention) | Flag Effect |
| --- | --- | --- | --- | --- |
| `0x0` | **NOP** | Ignored (`0`) | Advance `PC <- PC + 2` | Preserved |
| `0x1` | **MOV** | `[3:2]=dst, [1:0]=src` | Register transfer: `R[dst] <- R[src]` | Preserved |
| `0x2` | **LOAD** | Immediate / `MAX-1` / `MAX` | Load Immediate constant or read RAM into `A` via `C:D` | Preserved |
| `0x3` | **STORE** | Offset / `MAX-1` / `MAX` | Write `A` to RAM via direct offset or `C:D` | Preserved |
| `0x4` | **ALU** | `[3:0]=sub-opcode` | Execute arithmetic/logic on `A` and `B` | **Updated** |
| `0x5` | **JMP** | Target / `MAX` | Unconditional jump (Local offset or Far via `C:D`) | Preserved |
| `0x6` | **JZ** | Target / `MAX` | Conditional jump if `ZF == 1` (Local offset or Far via `C:D`) | Preserved |
| `0x7` | **JC** | Target / `MAX` | Conditional jump if `CF == 1` (Local offset or Far via `C:D`) | Preserved |
| `0x8` | **CALL** | Target / `MAX` | Subroutine call (Pushes frame `PR`, then `PC`) | Preserved |
| `0x9` | **RET** | Ignored (`0`) | Subroutine return (Pops frame `PC`, then `PR`) | Preserved |
| `0xA` | **PUSH** | `[2:0]=Target ID` | Push selected register/system target to Hardware Stack | Preserved |
| `0xB` | **POP** | `[2:0]=Target ID` | Pop Hardware Stack into selected register target | Preserved |
| `0xC` | **IO** | `[2:0]=port, [3]=dir` | $W$-bit peripheral transfer (`Dir=0` -> `IN` to `A`, `1` -> `OUT` from `A`) | Preserved |
| `0xD` | **SKP** | `[1:0]=cond` | Skip next instruction if condition evaluates True | Preserved |
| `0xE` | **RSVD** | Reserved (`0`) | Reserved expansion opcode (Behaves strictly as NOP) | Preserved |
| `0xF` | **HLT** | Reserved (`0`) | Freeze execution phase counter until hardware reset | Preserved |

---

### Peripheral I/O Specification (`IO`)

`IO` performs $W$-bit parallel data transfers across 8 addressable peripheral ports (`port = Bits [2:0]`):

* **`IO IN` (`Dir = 0`):** Sample $W$-bit data from peripheral Port `[2:0]` during `T2`; commit to Accumulator `A` at `T3`.
* **`IO OUT` (`Dir = 1`):** Drive Accumulator `A` data onto peripheral Port `[2:0]` during `T3`.

---

### Skip Instruction Condition Mapping (`SKP`)

`SKP` evaluates persistent condition latches (`ZF`, `CF`) using `Bits [1:0]` of the operand. If `True`, `SKP_TAKEN` is latched, issuing two extra `PC_INC` pulses across phases `T2` and `T3` ($N+2 \rightarrow N+3 \rightarrow N+4$).

| Binary (`cond`) | Hex / Dec | Mnemonic | Evaluation / Trigger Condition | Hardware Action |
| --- | --- | --- | --- | --- |
| `00` | `0` | **SKP** | Unconditional (`True`) | Advance `PC <- PC + 4` |
| `01` | `1` | **SKP Z** | `ZF == 1` | Advance `PC <- PC + 4` if `ZF == 1` |
| `10` | `2` | **SKP C** | `CF == 1` | Advance `PC <- PC + 4` if `CF == 1` |
| `11` | `3` | **SKP NZ** | `ZF == 0` | Advance `PC <- PC + 4` if `ZF == 0` |

*Note: Upper operand bits `[W-1:2]` are ignored by the decoder.*

---

## 6. Structured Bit-Pattern Hardware ALU Specification

```text
Sub-Opcode Field Mapping: SubOp[3:0] = { NoWrite, BlockSel, Ctrl1, Ctrl0 }

```

### Non-Writing ALU Operations Contract

For every non-writing ALU operation (`SubOp[3] == 1`), condition latches (`ZF`, `CF`) are computed from the $W$-bit bus result at `T3` **exactly as if written to Register `A**`, while Register `A` remains unmodified.

### Signal Control Mapping

* **Bit 3 (`NoWrite`):** Inhibit Register `A` write-enable pulse at `T4`.
* **Bit 2 (`BlockSel`):** `0` = Select Full Adder path. `1` = Select Logic / Shift path.
* **Bits [1:0] (`Ctrl[1:0]`):**
* **On Adder Path (`BlockSel == 0`):** `ForceB0 = SubOp[1]`, `InvB = SubOp[0]`, `Cin = SubOp[1] ^ SubOp[0]`.
* **On Logic Path (`BlockSel == 1`):** Selects AND, OR, SHR, or XOR.



### Structured ALU Sub-Opcode Engine

| Sub-Op | Mnemonic | Write `A`? | Operation Pseudocode | Zero Flag (`ZF`) | Carry Flag (`CF`) |
| --- | --- | --- | --- | --- | --- |
| `0x0` | **ADD** | **Yes** | `A <- A + B` | `ZF <- (A' == 0)` | `CF <- CarryOut` |
| `0x1` | **SUB** | **Yes** | `A <- A - B` | `ZF <- (A' == 0)` | `CF <- NoBorrow` |
| `0x2` | **INC** | **Yes** | `A <- A + 1` | `ZF <- (A' == 0)` | `CF <- CarryOut` |
| `0x3` | **DEC** | **Yes** | `A <- A - 1` | `ZF <- (A' == 0)` | `CF <- NoBorrow` |
| `0x4` | **AND** | **Yes** | `A <- A & B` | `ZF <- (A' == 0)` | `CF <- 0` |
| `0x5` | **OR** | **Yes** | `A <- A | B` | `ZF <- (A' == 0)` | `CF <- 0` |
| `0x6` | **SHR** | **Yes** | `A <- floor(A / 2)` | `ZF <- (A' == 0)` | `CF <- A[0]` (Pre-shift LSB) |
| `0x7` | **XOR** | **Yes** | `A <- A ^ B` | `ZF <- (A' == 0)` | `CF <- 0` |
| `0x8` | **ADD-NW** | **No** | Test `A + B` | `ZF <- ((A + B) == 0)` | `CF <- CarryOut` |
| `0x9` | **CMP** | **No** | Test `A - B` | `ZF <- (A == B)` | `CF <- (A >= B)` |
| `0xA` | **INC-NW** | **No** | Test `A + 1` | `ZF <- ((A + 1) == 0)` | `CF <- CarryOut` |
| `0xB` | **DEC-NW** | **No** | Test `A - 1` | `ZF <- ((A - 1) == 0)` | `CF <- NoBorrow` |
| `0xC` | **TST** | **No** | Test `A & B` | `ZF <- ((A & B) == 0)` | `CF <- 0` |
| `0xD` | **OR-NW** | **No** | Test `A | B` | `ZF <- ((A | B) == 0)` | `CF <- 0` |
| `0xE` | **SHR-NW** | **No** | Test `A >> 1` | `ZF <- (floor(A / 2) == 0)` | `CF <- A[0]` (Pre-shift LSB) |
| `0xF` | **TEQ** | **No** | Test `A ^ B` | `ZF <- (A == B)` | `CF <- 0` |

---

## 7. Unified Subroutine Frame Protocol & Stack Mechanics

Subroutine control relies on a standardized 2-word frame pushed/popped across phases `T2` and `T3`.

```text
Stack Frame Layout:
Stack Top (SP-1) -> [ Return PC ]  (Sequential PC state produced by T0/T1 fetch)
Stack Top (SP-2) -> [ Return PR ]  (Page register state, auto-incremented if fetch rolled over)

```

### Execution Sequence:

* **`CALL` Phase `T2`:** Push `PR` onto stack.
* **`CALL` Phase `T3`:** Push `PC` onto stack.
* **`RET` Phase `T2`:** Pop stack into `PC`.
* **`RET` Phase `T3`:** Pop stack into `PR`.

*Zero arithmetic required: because fetch phases `T0` and `T1` already advanced `PC` to $N+2$ (and updated `PR` if rollover occurred), `CALL` simply transfers live register values `PR` and `PC` directly to the stack.*

---

## 8. Deterministic 5-Phase Execution Pipeline (`T0..T4`)

Every instruction executes across five fixed, deterministic clock phases driven by a 1-hot ring counter:

```text
Phases:   |   T0   |   T1   |     T2     |     T3     |     T4     |
Action:   | Opcode | Operand| Execution  | Execution  | State      |
          | Fetch  | Fetch  | Phase 1    | Phase 2    | Commit     |

```

### Phase Breakdown

* **Phase `T0` (Opcode Fetch):**
`IR_op <- Memory[PR : PC]`; issue `PC_INC` pulse (`PC` becomes $N+1$).
* **Phase `T1` (Operand Fetch):**
`IR_opr <- Memory[PR : PC]`; issue `PC_INC` pulse (`PC` becomes $N+2$).
* **Phase `T2` (Execution Phase 1):**
Sample registers; evaluate ALU operations, sample `IO IN`, or execute Stack Phase 1 (`CALL` pushes `PR`, `RET` pops `PC`).
* **`SKP` Logic:** Evaluate `ZF`/`CF` against `cond[1:0]`. If `True`, set `SKP_TAKEN = 1` and issue `PC_INC` pulse (`PC: N+2 -> N+3`).


* **Phase `T3` (Execution Phase 2):**
RAM read/write cycles, drive `IO OUT`, commit `IO IN` to `A`, or execute Stack Phase 2 (`CALL` pushes `PC`, `RET` pops `PR`).
* **`SKP` Logic:** If `SKP_TAKEN == 1`, issue a second `PC_INC` pulse (`PC: N+3 -> N+4`).


* **Phase `T4` (Architectural Commit):**
* **Default / SKP PC Status:** `PC` is fully settled at $N+2$ (normal execution) or $N+4$ (taken `SKP`).
* **Control Flow Branch Commit:** If `JMP`/`CALL` taken, write branch target to `PC` (Local) or `PR:PC` (Far via `C:D`).
* **Auto-Increment Commit:** If `Operand == MAX` on memory access, pulse `D` counter: $D \leftarrow (D + 1) \bmod 2^W$.


* **Halt Behavior (`HLT`):**
Freezes phase counter (`T0..T4`). Preserves all architectural state (`A, B, C, D, PC, PR`, latches, stack, RAM) until hardware reset.

---

## Hardware Control Matrix Logic Equation

For discrete logic implementation, the master `PC_INC` pulse line is defined as:

```text
PC_INC = T0 OR T1 OR (T2 AND SKP_TAKEN) OR (T3 AND SKP_TAKEN)

```

---

## Architectural Invariant Summary (v4.6.0 Final Reference)

* **Datapath Width:** $W$ bits ($W \ge 4$)
* **Primary Registers:** `A`, `B`, `C`, `D` (4 general/address registers)
* **Instruction Size:** $2W$ bits (`[W-bit Opcode] : [W-bit Operand]`)
* **Address Topology:** $2W$-bit address space ($2^{2W}$ locations of $W$ bits)
* **Program Counter (`PC`):** $W$-bit offset ($0, 1, \dots, 2^W - 1$), sequential increment via `PC_INC`
* **Page Capacity:** $2^W$ storage units per page `PR`
* **Clock Engine:** 5 fixed, deterministic phases (`T0..T4`)
* **Stack Primitive:** $W$-bit abstract circular LIFO buffer (implementation details unconstrained)
* **Subroutine Frame:** 2 words (Pushed `PR` then `PC`; Popped `PC` then `PR`)
* **Condition Latches:** Persistent `ZF`, `CF` latches (updated only by ALU operations)
* **ALU Control:** Direct hardware signal mapping `{ NoWrite, BlockSel, Ctrl1, Ctrl0 }`
* **`MOV` Bitfield:** `[3:2]=dst, [1:0]=src` (Intel Convention `MOV dst, src`)
* **`SKP` Increment Engine:** 2-step pulse across phases `T2` and `T3` on `SKP_TAKEN`
* **LOAD Escapes:** `<= MAX - 2` -> Immediate | `MAX - 1` -> `[C:D]` | `MAX` -> `[C:D]+`
* **STORE Escapes:** `<= MAX - 2` -> Offset `[C:Opr]` | `MAX - 1` -> `[C:D]` | `MAX` -> `[C:D]+`
* **Branch Escapes:** `<= MAX - 1` -> Page Local (`PR:Opr`) | `MAX` -> Far Target (`C:D`)