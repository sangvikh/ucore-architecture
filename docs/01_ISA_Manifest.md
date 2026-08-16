# $\mu$-Core 4-Bit Architecture Reference Manual (v8.4 Master Edition)

**Target Hardware:** Discrete NMOS / TTL (< 1,000 Transistor Goal)

**Datapath Width:** True 4-Bit ($W = 4$)

**Memory Space:** 256 Nibbles ($16 \text{ Pages} \times 16 \text{ Nibbles/Page} = 128 \text{ Maximum Instructions}$)

**Bus Topology:** Active-Low Shared Data Bus with $10\text{k}\Omega$ Pull-Up Resistors to VDD and NMOS Pass-Transistor Drivers

**Stack Model:** Empty Ascending (EA) Hardware Stack with Isolated `STACK_OE` / `STACK_WE` Control

---

## 1. Architectural Overview & Principles

* **Single-Phase Multi-Step Execution:** Governed by a contention-free 10-step fetch cycle ($T_0 \dots T_9$) and instruction-specific execution phases ($T_{10}\dots$).
* **Active-Low Pull-Up Bus:** Bus lines float high via $10\text{k}\Omega$ resistors to VDD ($5\text{V}$). Registers pull lines down to GND using single NMOS pass transistors per bit, completely eliminating the classic NMOS pass-transistor $V_{tn}$ voltage drop.
* **Write-Enable Integration:** Bus reads are built directly into the register write-enable (`WE`) logic to minimize transistor count.
* **Time-Shared Central ALU:** Arithmetic, logic, address increments, and relative jump calculations all route through the primary central ALU, eliminating dedicated address adders.

---

## 2. System Block Diagram

```text
 +-----------------------------------------------------------------------------------+
 |                             CENTRAL CONTROL & SEQUENCER                           |
 |  - Single-Phase Multi-Step Ring Counter    - USE_TEMP_C / STACK_OE / STACK_WE     |
 |  - Diode/Transistor Decoding Matrix        - Break-Before-Make Timing Logic       |
 +---------+------------------------+-------------------------+----------------------+
           |                        |                         |
           v (Control Lines)        v (Control Lines)         v (Control Lines)
 +--------------------+  +--------------------+    +--------------------+
 | MAIN PAGING RAM    |  | STACK RAM (EA)     |    | REGISTER FILE      |
 | - Addr: MUX(PR:PC) |  | - Addr: Dedicated  |    | - A, B, C, D       |
 | - Data: Active-Low |  | - Data: Active-Low |    | - PC, PR, SP       |
 +---------+----------+  +---------+----------+    | - FLAGS (0x7)      |
           |                        |              +---------+----------+
           +--------------------+   |                        |
                                |   |                        |
                                v   v                        v
================================+===+========================+=======================
                         ACTIVE-LOW SHARED 4-BIT DATA BUS (10k Pull-Ups)
================================+====================================================
                                |
                                v
                     +----------------------+
                     | CENTRAL 4-BIT ALU    |
                     | - Input A / Input B  |
                     | - Bitmapped Logic    |
                     | - C_fetch Shadow Latch|
                     +----------+-----------+
                                |
                                +---> Sideband Wires (Z_out, C_out, N_out) ---> FLAGS Card

```

---

## 3. Register Target Mapping (`src` / `dst` 3-Bit Encoding)

Registers are selected via 3-bit fields (`IR_{opr}[2:0]`):

| Binary Code | Register Mnemonic | Description |
| --- | --- | --- |
| `000` | **`A`** | Primary Accumulator & ALU working register |
| `001` | **`B`** | Secondary General-Purpose Register |
| `010` | **`C`** | Memory Page Pointer High (`C:D` pair) |
| `011` | **`D`** | Memory Offset Pointer Low (`C:D` pair) |
| `100` | **`PC`** | Program Counter (Low nibble offset) |
| `101` | **`PR`** | Page Register (High nibble page selector) |
| `110` | **`SP`** | Stack Pointer (Points to next free stack slot) |
| `111` | **`FLAGS`** | Status Flags Register (`[2]=N`, `[1]=C`, `[0]=Z`) |

---

## 4. Central ALU Data Path & Function Table

The central ALU is a **time-shared arithmetic and logic unit** handling both unary and binary operations.

### Data Flow Path

1. **Source Selection:** Operands drive the active-low bus via NMOS pass transistors.
2. **Input Latching:** The ALU captures bus data into its internal input latches (Input A holds accumulator `A`, Input B captures incoming bus data).
3. **Execution:** Bitmapped `ALU_CTRL[3:0]` lines configure the logic gates.
4. **Writeback / Sideband:** Results drive the bus via `OE_ALU`, while status flags ($Z, C, N$) update via sideband wires.

### Unary vs. Binary Operations

* **Unary Operations (`INC`, `DEC`, `NOT`, `PASS`):** Require a single input from the bus; the second operand is hardwired internally (e.g., `+1` for increments, `0` for passthrough).
* **Binary Operations (`ADD`, `SUB`, `AND`, `OR`, `XOR`):** Require two inputs: Accumulator `A` and Register `B` (driven from the bus).

### ALU Function Table (`ALU_CTRL[3:0]`)

| `ALU_CTRL[3:0]` | Mnemonic | Operation Description | Flag Impact ($Z, C, N$) |
| --- | --- | --- | --- |
| **`0000`** | `ADD` | $A \leftarrow A + B$ | Updated from ALU result |
| **`0001`** | `ADC` | $A \leftarrow A + B + C_{\text{in}}$ | Updated from ALU result |
| **`0010`** | `SUB` | $A \leftarrow A - B$ | Updated from ALU result |
| **`0011`** | `SBB` | $A \leftarrow A - B - C_{\text{in}}$ | Updated from ALU result |
| **`0100`** | `AND` | $A \leftarrow A \land B$ | Updated ($N, Z$ valid, $C=0$) |
| **`0101`** | `OR` | $A \leftarrow A \lor B$ | Updated ($N, Z$ valid, $C=0$) |
| **`0110`** | `XOR` | $A \leftarrow A \oplus B$ | Updated ($N, Z$ valid, $C=0$) |
| **`0111`** | `NOT` | $A \leftarrow \neg A$ | Updated ($N, Z$ valid, $C=0$) |
| **`1000`** | `PASS` | $A \leftarrow \text{Input}$ (Identity buffer) | Unaffected |
| **`1001`** | `INC` | $A \leftarrow A + 1$ | Updated from ALU result |
| **`1010`** | `DEC` | $A \leftarrow A - 1$ | Updated from ALU result |
| **`1011`** | `DEC2` | Specialized decrement for stack pointer | Unaffected |

---

## 5. Specialized Operand Bitfields

Due to the strict 4-bit operand limit ($IR_{opr}$), specialized instructions use packed bitfields:

### A. `MOV dst, src` Operand Encoding (Opcode `0x1`)

* Bits [3:2]: Destination Register (`dst`) — restricted to core working set (`00=A`, `01=B`, `10=C`, `11=D`)
* Bits [1:0]: Source Register (`src`) — restricted to core working set (`00=A`, `01=B`, `10=C`, `11=D`)

### B. `LD` & `ST` Operand Bitfield (Opcodes `0x3` and `0x4`)

* **Bit 3 (MSB):** Auto-Increment Flag (`0` = Normal pointer, `1` = Auto-increment `[C:D++]` after access).
* **Bits [2:0]:** 3-Bit Register Target (`A`, `B`, `C`, `D`, `PC`, `PR`, `SP`, `FLAGS`).

### C. `SKP` Condition Encoding (Opcode `0xC`)

* Bits [3:2]: Condition Type (`00` = Zero `Z`, `01` = Carry `C`, `10` = Negative `N`, `11` = Always/Invert).
* Bit [1]: Inversion Flag (`0` = Branch if True, `1` = Branch if False/Not).
* Bit [0]: Unused / Reserved.

---

## 6. Contention-Free 10-Step Fetch Sequence ($T_0$ through $T_9$)

Every 2-nibble instruction fetch sequence is strictly separated in time to prevent bus contention:

```text
Nibble 1 (Opcode):
  T0: [MEM_OE] -----------> Read RAM to IR_op
  T1: [OE_PC]  -----------> Setup PC + 1 on ALU
  T2: [OE_ALU] ----------> Write back PC <- PC + 1
  T3: [OE_PR]  -----------> Setup Page Carry (PR + C_fetch)
  T4: [OE_ALU] ----------> Write back PR <- PR + C_out

Nibble 2 (Operand):
  T5: [MEM_OE] -----------> Read RAM to IR_opr
  T6: [OE_PC]  -----------> Setup PC + 1 on ALU
  T7: [OE_ALU] ----------> Write back PC <- PC + 1
  T8: [OE_PR]  -----------> Setup Page Carry (PR + C_fetch)
  T9: [OE_ALU] ----------> Write back PR <- PR + C_out

```

---

## 7. Master 16-Opcode Map (`0x0` to `0xF`)

| Opcode | Mnemonic | Operands / Bitfield | Action / Description |
| --- | --- | --- | --- |
| **`0x0`** | **`NOP`** | None | No operation (idle execution cycle) |
| **`0x1`** | **`MOV`** | `dst, src` (2-bit enc) | Register-to-register move: $dst \leftarrow src$ (restricted to `A, B, C, D`) |
| **`0x2`** | **`LDI`** | `imm` (4-bit constant) | Load immediate nibble into Accumulator: $A \leftarrow IR_{opr}$ |
| **`0x3`** | **`LD`** | `[auto_inc:1][reg:3]` | Load memory nibble from $\text{RAM}[C:D]$ into target register. Optional `[C:D++]`. |
| **`0x4`** | **`ST`** | `[auto_inc:1][reg:3]` | Store source register to memory at $\text{RAM}[C:D]$. Optional `[C:D++]`. |
| **`0x5`** | **`ALU`** | `SubOp[3:0]` | Perform ALU operation on $A$ and $B$, updating status flags ($Z, C, N$). |
| **`0x6`** | **`JMP`** | `rel_off` (Signed 4-bit) | Relative jump: $PC:PR \leftarrow PC:PR + \text{SignExt}(IR_{opr})$ |
| **`0x7`** | **`JMPF`** | None | Far absolute jump: $PR \leftarrow C$, $PC \leftarrow D$ |
| **`0x8`** | **`CALL`** | `rel_off` (Signed 4-bit) | Relative subroutine call (pushes 16-bit return address $\text{PR:PC}$ to stack). |
| **`0x9`** | **`CALLF`** | None | Far subroutine call using $C:D$ (pushes 16-bit return address $\text{PR:PC}$). |
| **`0xA`** | **`RET`** | None | **Universal Return**: Pops 16-bit $\text{PR:PC}$ from stack, resuming execution seamlessly. |
| **`0xB`** | *Open* | — | *Reserved / Available for I/O mapping or custom extensions* |
| **`0xC`** | **`SKP`** | `cond[3:2]...` | Conditional skip next instruction word ($PC \leftarrow PC + 2$ if condition met). |
| **`0xD`** | **`PUSH`** | `target` (3-bit enc) | Push target register onto stack RAM ($\text{RAM}[SP] \leftarrow target$, $SP \leftarrow SP + 1$). |
| **`0xE`** | **`POP`** | `target` (3-bit enc) | Pop stack into target register ($SP \leftarrow SP - 1$, $target \leftarrow \text{RAM}[SP]$). |
| **`0xF`** | **`HLT`** | None | Halt CPU execution and suspend clock sequencer. |