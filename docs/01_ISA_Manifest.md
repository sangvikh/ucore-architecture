# $\mu$-Core 4-Bit Architecture Reference Manual (v9.3 Master Edition)

**Target Hardware:** Discrete NMOS / TTL (Preliminary Target: $< 1,000$ Transistors for **CPU Core Logic Only**; external socketed SRAM used for main and stack memory)

**Datapath Width:** True 4-Bit ($W = 4$)

**Memory Space:** 256 Nibbles ($16 \text{ Pages} \times 16 \text{ Nibbles/Page} = 128 \text{ Maximum Instructions}$)

**Bus Topology:** Open-Drain Shared Data Bus with Nominal $10\text{ k}\Omega$ Pull-Up Resistors ($2.2\text{ k}\Omega - 10\text{ k}\Omega$ allowable range) to VDD and NMOS Open-Drain Drivers

**ALU Datapath:** Dual-Operand Datapath with **Ephemeral 1-Cycle `ALU_A_LATCH**` (captured via `ALU_A_WE`), **Direct-Wired `ALU_B**` input from Register $B$, driving output via `ALU_OE`

**Stack Model:** 16-Nibble Empty Ascending (EA) Hardware Stack ($SP = 4\text{-bit}$ addressing dedicated external Stack SRAM)

**Sequencer Capacity:** **22 Micro-Steps ($T_0 \dots T_{21}$)** generated via an **11-Stage Johnson Counter** with **4 Sub-Tick Synchronous Bus Phase Control**

---

## 1. Architectural Overview, Principles & Invariants

### Architectural Principles

* **4 Sub-Tick Synchronous Bus Phase Control:** Every micro-step ($T_n$) is divided into 4 sub-ticks governed by a master oscillator and a 2-bit counter ($Q_1 Q_0$). This guarantees $OE$ and $WE$ overlap safely on the passive pull-up bus across all clock speeds:
* **Sub-tick `00` (Bus Setup):** $OE_{\text{source}} = 1, WE_{\text{target}} = 0$. Source drives bus; signal levels settle.
* **Sub-tick `01` (Latch Capture):** $OE_{\text{source}} = 1, WE_{\text{target}} = 1$. Target latch captures stable bus data.
* **Sub-tick `10` (Hold Window):** $OE_{\text{source}} = 1, WE_{\text{target}} = 0$. Target latch locks shut while source continues driving, guaranteeing hold time ($t_{\text{hold}}$).
* **Sub-tick `11` (Release & Advance):** $OE_{\text{source}} = 0, WE_{\text{target}} = 0$. Source releases bus back to passive pull-ups, and the sequencer advances to $T_{n+1}$.


* **Clocking Modes & Single-Stepping:**
* **`RUN` Mode:** Master clock continuously oscillates at $4\times$ micro-step frequency.
* **`STEP` Mode:** Master clock may be stopped indefinitely between synchronous sub-tick boundaries. Single-step debugging occurs by stepping one sub-tick or one full micro-step at a time without losing bus or latch state.


* **Open-Drain Bus Electrical Model:** The shared bus is pulled high via nominal $R_{PU} = 10\text{ k}\Omega$ resistors ($2.2\text{ k}\Omega - 10\text{ k}\Omega$ permitted) and driven low via NMOS open-drain pull-down transistors. The maximum allowable bus capacitance is $C_{\text{BUS}} = 100\text{ pF}$, establishing an RC time constant $\tau = R_{PU} \cdot C_{\text{BUS}} \le 1.0\text{ }\mu\text{s}$. Sub-tick duration must satisfy $t_{\text{sub}} \ge 5\tau$ to guarantee valid logic HIGH restoration upon bus release.
* **1-Bus Execution Limit:** The single shared bus permits exactly **one $OE$ driver at a time**. Multi-step operations are strictly decomposed into isolated sequential bus transactions.
* **11-Stage Johnson Sequencer Topology:** Uses an 11-stage twisted-ring counter (11 D flip-flops) to generate 22 distinct state vectors decoded into 22 1-hot step lines ($T_0 \dots T_{21}$). An explicit diode/transistor feedback gate forces any illegal state back into the valid ring sequence within 10 clock cycles.
* **Ephemeral ALU Shadow-Latch Architecture:** Resolves $A \to \text{ALU} \to A$ feedback race hazards using a single 4-bit input shadow latch (`ALU_A_LATCH`). Operations execute across a 2-microcycle rhythm:
* **Phase $T_0$ (Capture):** Source register drives the shared bus $\rightarrow$ captured into `ALU_A_LATCH` via `ALU_A_WE`.
* **Phase $T_1$ (Compute & Writeback):** `ALU_A_WE` is disabled (locking `ALU_A_LATCH`). The ALU computes combinational output, drives the bus via `ALU_OE`, and writes back to the destination register (`DEST_WE`).


* **Universal Datapath & Pointer Reuse:** Any architectural pointer or register ($A, SP, PC, D$) can be loaded into `ALU_A_LATCH` via the bus. All arithmetic, logic, pointer increments, and relative branch calculations reuse this single central ALU without dedicated address adders or incrementers.

---

### Formal Architectural Invariants

1. **Bus Invariant:** At most one $OE$ driver may be asserted during any sub-tick ($OE_{\text{total}} \le 1$).
2. **Write Invariant:** Any $WE$ strobe may be asserted exclusively during Sub-Tick `01`.
3. **ALU Invariant:** `ALU_A_LATCH` is written only via `ALU_A_WE` (Sub-tick `01`) and remains latched and stable throughout the subsequent compute/writeback step.
4. **Stack Invariant:** $SP$ (4-bit) always points to the next free stack nibble in the dedicated 16-nibble Stack SRAM.
5. **PC Invariant:** $PR:PC$ (8-bit) holds the address of the next instruction nibble to fetch.
6. **Instruction Invariant:** Every architectural instruction occupies exactly 2 nibbles (Opcode + Operand/SubOp).
7. **Sequencer Invariant:** Johnson state transitions and `RESET_SEQ` take effect synchronously at the boundary of Sub-Tick `11` (after bus release).

---

## 2. System Block Diagram & Memory Interface

```text
+-----------------------------------------------------------------------------------+
|                             CENTRAL CONTROL & SEQUENCER                           |
|  - Sub-Tick Generator (00=Setup, 01=WE, 10=Hold, 11=Adv/Reset)                      |
|  - 11-Stage Johnson Counter (22 State Vectors -> T0..T21 1-Hot Step Lines)        |
|  - Diode Matrix Control Decoder (ALU_A_WE / ALU_OE / REG_WE[x])                   |
+---------+------------------------+-------------------------+----------------------+
          |                        |                         |
          v (Control Lines)        v (Control Lines)         v (Control Lines)
+--------------------+  +--------------------+    +--------------------+
| MAIN PAGING RAM    |  | STACK RAM (EA)     |    | REGISTER FILE      |
| - Addr: MEM_ADDR   |  | - Addr: SP (4-Bit) |    | - A, B, C, D       |
| - Data: Open-Drain |  | - Data: Open-Drain |    | - PC, PR, SP       |
+---------+----------+  +---------+----------+    | - FLAGS (0x7)      |
          |                        |              +---------+----------+
          |                        |                        |
          +--------------------+   |                        |
                               |   |                        v
                               v   v========================+=======================
                                OPEN-DRAIN SHARED 4-BIT DATA BUS (10k Pull-Ups)
                               =============================+=======================
                                                            |
                                                            v
                                                 +----------------------+
                                                 | ALU_A_LATCH (4-Bit)  | <--- BUS (strobe ALU_A_WE)
                                                 +----------+-----------+
                                                            |
                                                            v (Input A)
Register B (Permanent Direct Wire) ----------------------> +----------------------+
                                                           | CENTRAL 4-BIT ALU    |
                                                           | - Inversion & Carry  |
                                                           | - C_fetch Shadow     |
                                                           +----------+-----------+
                                                                      |
                                                                      +---> Sideband (Z_out, C_out, N_out)
                                                                      |
                                                                      v (Drive via ALU_OE)
                                                             OPEN-DRAIN DATA BUS

```

### Memory Address Multiplexer (`MEM_ADDR[7:0]`)

The 8-bit memory address bus `MEM_ADDR[7:0]` to external Main RAM is driven by an internal 2:1 multiplexer:


$$MEM\_ADDR[7:0] = \begin{cases} PR:PC & \text{during Instruction Fetch } (T_0 \dots T_9) \\ C:D & \text{during Data Access } (LD/ST, T_{10} \dots T_{14}) \end{cases}$$

---

## 3. Register File, Stack Model & FLAGS Semantics

### A. General Register Target Mapping (3-Bit Field)

Used by `PUSH`, `POP`, and general operations:

| Binary Code | Register Mnemonic | Description |
| --- | --- | --- |
| `000` | **`A`** | Primary Accumulator & ALU working register |
| `001` | **`B`** | Secondary General-Purpose Register (Direct-wired to `ALU_B`) |
| `010` | **`C`** | Memory Page Pointer High (`C:D` pair) |
| `011` | **`D`** | Memory Offset Pointer Low (`C:D` pair) |
| `100` | **`PC`** | Program Counter (Low nibble offset) |
| `101` | **`PR`** | Page Register (High nibble page selector) |
| `110` | **`SP`** | Stack Pointer (Points to next free stack slot) |
| `111` | **`FLAGS`** | Status Flags Register (`[2]=N`, `[1]=C`, `[0]=Z`) |

### B. Memory-Restricted Target Mapping (`LD` / `ST`)

To eliminate complex sequencer side-effects, **`LD` and `ST` targets are strictly restricted to the core working set**: `A`, `B`, `C`, and `D` only. (Control/Pointer registers like `PC`, `PR`, `SP`, and `FLAGS` must be accessed via working registers and `MOV` or `POP`).

### C. Stack Model & Framing Specification

* **Hardware Sizing:** $SP$ is a 4-bit Empty Ascending (EA) register addressing a dedicated 16-nibble external Stack SRAM ($0\times 0 \dots 0\times F$).
* **Capacity:** Supports up to 16 pushed nibbles or **8 nested `CALL` frames** (each `CALL` pushes $PC$ then $PR$). Stack overflow/underflow behavior is undefined in hardware and must be prevented by software.
* **Stack Frame Diagram:**

```text
Stack Address        Contents                 Operation Sequence
  (SP Space)
             +-----------------------+
   SP - 2    |   PC Return Low       |  <-- Pushed first during CALL (T10)
             +-----------------------+
   SP - 1    |   PR Return High      |  <-- Pushed second during CALL (T13)
             +-----------------------+
    SP ----> |     (Empty Slot)      |  <-- SP points to next free location
             +-----------------------+

```

### D. FLAGS Register Semantics & Transfer Matrix

* **Bitfield Definitions:** `FLAGS[2] = N` (Negative), `FLAGS[1] = C` (Carry), `FLAGS[0] = Z` (Zero).
* **Borrow Convention:** For subtraction ($SUB, SBB$), $C = 1$ indicates **No Borrow** ($A \ge B$), and $C = 0$ indicates **Borrow Occurred** ($A < B$).
* **FLAGS Access Rules:**
* `PUSH FLAGS`: Reads `0 | FLAGS[2:0]` and writes to `RAM[SP]`.
* `POP FLAGS`: Pops nibble from `RAM[SP]` and directly overwrites `FLAGS[2:0]`.
* `MOV`: `FLAGS` cannot be the source or destination of a `MOV` instruction.


* **Flag Update Matrix:**

| Operation Class | `N` (Bit 2) | `Z` (Bit 0) | `C` (Bit 1) | Notes |
| --- | --- | --- | --- | --- |
| **Arithmetic (`ADD`, `ADC`, `SUB`, `SBB`, `INC`, `DEC`)** | Updated | Updated | Updated | Full arithmetic flag update |
| **Logical (`AND`, `OR`, `XOR`, `NOT`)** | Updated | Updated | **Written 0** | $C$ is explicitly cleared to `0` |
| **Pass-through (`PASS`)** | Unchanged | Unchanged | Unchanged | No flag modification |
| **Data Movement (`MOV`, `LDI`, `LD`, `ST`, `PUSH`, `POP` non-FLAGS)** | Unchanged | Unchanged | Unchanged | No flag modification |
| **Branching & Control (`JMP`, `CALL`, `RET`, `SKP`, `NOP`)** | Unchanged | Unchanged | Unchanged | No flag modification |
| **`POP FLAGS`** | **Loaded** | **Loaded** | **Loaded** | Overwritten directly from stack |

---

## 4. Central ALU Datapath & Hardware Decoding

The central ALU features streamlined hardware sharing via **operand inversion and carry-in muxing** ($A + \sim B + C_{\text{in}}$) to minimize transistor count.

### A. ALU Hardware Signal Control Decoding (`ALU_CTRL[3:0]`)

| `ALU_CTRL[3:0]` | Mnemonic | `INV_B` | `CIN_SEL` | `LOGIC_MODE` | `PASS_A` | Operation Formula | Flag Impact |
| --- | --- | --- | --- | --- | --- | --- | --- |
| **`0000`** | **`ADD`** | `0` | `0` ($0$) | `0` (Arith) | `0` | $A + B + 0$ | $Z, C, N$ Updated |
| **`0001`** | **`ADC`** | `0` | `1` ($C_{\text{in}}$) | `0` (Arith) | `0` | $A + B + C_{\text{in}}$ | $Z, C, N$ Updated |
| **`0010`** | **`SUB`** | `1` | `2` ($1$) | `0` (Arith) | `0` | $A + \sim B + 1$ | $Z, C, N$ Updated |
| **`0011`** | **`SBB`** | `1` | `1` ($C_{\text{in}}$) | `0` (Arith) | `0` | $A + \sim B + C_{\text{in}}$ | $Z, C, N$ Updated |
| **`0100`** | **`AND`** | `0` | `0` ($0$) | `1` (Logic) | `0` | $A \land B$ | $Z, N$ Valid, $C=0$ |
| **`0101`** | **`OR`** | `0` | `0` ($0$) | `1` (Logic) | `0` | $A \lor B$ | $Z, N$ Valid, $C=0$ |
| **`0110`** | **`XOR`** | `0` | `0` ($0$) | `1` (Logic) | `0` | $A \oplus B$ | $Z, N$ Valid, $C=0$ |
| **`0111`** | **`NOT`** | `0` | `0` ($0$) | `1` (Logic) | `0` | $\sim A$ (Selects $\sim A$ independently) | $Z, N$ Valid, $C=0$ |
| **`1000`** | **`PASS`** | `0` | `0` ($0$) | `0` (Arith) | `1` | $A$ (Pass-through) | Unaffected |
| **`1001`** | **`INC`** | `0` | `2` ($1$) | `0` (Arith) | `0` | $A + 0 + 1$ ($B$ forced 0) | $Z, C, N$ Updated |
| **`1010`** | **`DEC`** | `1` | `0` ($0$) | `0` (Arith) | `0` | $A + \text{0xF} + 0$ | $Z, C, N$ Updated |

*Note on Implicit Operands:* ALU operations (`0x5`) operate on implicit source operands $A$ and $B$, writing the result back to implicit destination $A$.

---

## 5. Specialized Operand Bitfields & Formal Relative Addressing

### A. `MOV dst, src` Operand Encoding (Opcode `0x1`)

* **Bits [3:2]:** Destination Register (`dst`) — restricted to core working set (`00=A`, `01=B`, `10=C`, `11=D`)
* **Bits [1:0]:** Source Register (`src`) — restricted to core working set (`00=A`, `01=B`, `10=C`, `11=D`)

### B. `LD` & `ST` Operand Bitfield (Opcodes `0x3` and `0x4`)

* **Bit 3 (MSB):** Auto-Increment Flag (`0` = Normal pointer, `1` = Auto-increment `[C:D++]` after access).
* **Bits [2:0]:** 3-Bit Register Target restricted to `A`, `B`, `C`, `D`.

### C. `SKP` Condition Encoding (Opcode `0xC`)

* **Bits [3:2]:** Condition Type (`00` = Zero `Z`, `01` = Carry `C`, `10` = Negative `N`, `11` = Always/Invert).
* **Bit [1]:** Inversion Flag (`0` = Branch if True, `1` = Branch if False/Not).
* **Bit [0]:** Unused / Reserved.
* **Execution Semantics:** A taken `SKP` advances the 8-bit program pointer ($PR:PC$) by **2 nibbles** (skipping the single subsequent 2-nibble instruction).

### D. Formal Signed Relative Address Arithmetic (`JMP` / `CALL`)

For a 4-bit two's-complement offset $r = IR_{\text{opr}}[3:0]$, relative jumping across page boundaries is defined formally by:


$$PC' = (PC + r) \pmod{16}$$

$$PR' = (PR + s + C_{PC}) \pmod{16}$$


where:


$$s = \begin{cases} 0 & \text{if } r_3 = 0 \text{ (Positive Offset: } 0 \dots +7\text{)} \\ 15 & \text{if } r_3 = 1 \text{ (Negative Offset: } -8 \dots -1\text{)} \end{cases}$$


$C_{PC} = \lfloor (PC + r) / 16 \rfloor$ is the carry-out from the low-nibble adder step.

During step $T_{13}$ ($JMP$) or $T_{19}$ ($CALL$), signal `OE_SIGN_EXT` drives $s$ (`0x0` or `0xF`) onto the bus into Register $B$, allowing the standard ALU $ADC$ operation to complete exact 8-bit relative page correction.

---

## 6. Master Micro-Step Execution Breakdown ($T_0 \dots T_{21}$)

### A. Standard 10-Step Fetch Cycle ($T_0 \dots T_9$) — Common to ALL Instructions

Every 2-nibble instruction fetch sequence uses the unified $T_0 / T_1$ ALU datapath sequence for **`INC_PC`** to handle page rollovers cleanly ($PC \leftarrow PC + 1$, $PR \leftarrow PR + C_{\text{fetch}}$):

```text
Opcode Fetch (Nibble 1):
  T0: MEM_OE -> IR_op                    ; Drive RAM[PR:PC] onto BUS into Instruction Register (Opcode)
  T1: OE_PC, ALU_A_WE                    ; Drive PC onto BUS, capture into ALU_A_LATCH
  T2: ALU_OE(INC), WE_PC                 ; ALU computes PC+1 -> BUS -> PC; capture C_out in C_fetch latch
  T3: OE_PR, ALU_A_WE                    ; Drive PR onto BUS, capture into ALU_A_LATCH
  T4: ALU_OE(ADC 0), WE_PR               ; ALU adds C_fetch -> BUS -> PR

Operand Fetch (Nibble 2):
  T5: MEM_OE -> IR_opr                   ; Drive RAM[PR:PC] onto BUS into Operand Register
  T6: OE_PC, ALU_A_WE                    ; Drive PC onto BUS, capture into ALU_A_LATCH
  T7: ALU_OE(INC), WE_PC                 ; ALU computes PC+1 -> BUS -> PC; capture C_out in C_fetch latch
  T8: OE_PR, ALU_A_WE                    ; Drive PR onto BUS, capture into ALU_A_LATCH
  T9: ALU_OE(ADC 0), WE_PR               ; ALU adds C_fetch -> BUS -> PR

```

---

### B. Micro-Step Detailed Trace per Opcode ($T_{10}\dots$)

#### `0x0 NOP` (No Operation)

* **$T_{10}$:** `RESET_SEQ` (Sampled at Sub-Tick 11 $\rightarrow$ Clears counter to $T_0$).

#### `0x1 MOV dst, src`

* **$T_{10}$:** `OE_src`, `WE_dst`, `RESET_SEQ`.

#### `0x2 LDI imm`

* **$T_{10}$:** `OE_IR_opr`, `WE_A`, `RESET_SEQ`.

#### `0x3 LD target, [auto_inc]`

* **$T_{10}$:** `MEM_OE_CD`, `WE_target`. *(If `auto_inc` = 0, trigger `RESET_SEQ`)*.
* **$T_{11}$:** `OE_D`, `ALU_A_WE`.
* **$T_{12}$:** `ALU_OE(INC)`, `WE_D`, Save $C_{\text{out}} \rightarrow C_{\text{fetch}}$.
* **$T_{13}$:** `OE_C`, `ALU_A_WE`.
* **$T_{14}$:** `ALU_OE(ADC 0)`, `WE_C`, `RESET_SEQ`.

#### `0x4 ST target, [auto_inc]`

* **$T_{10}$:** `OE_target`, `MEM_WE_CD`. *(If `auto_inc` = 0, trigger `RESET_SEQ`)*.
* **$T_{11}\dots T_{14}$:** Identical auto-increment step sequence as `LD` ($D \leftarrow D+1$, $C \leftarrow C + C_{\text{fetch}}$).

#### `0x5 ALU SubOp`

* **$T_{10}$:** `OE_A`, `ALU_A_WE`.
* **$T_{11}$:** `ALU_OE(SubOp)`, `WE_A`, `WE_FLAGS`, `RESET_SEQ`.

#### `0x6 JMP rel_off`

* **$T_{10}$:** `OE_IR_opr`, `WE_B`.
* **$T_{11}$:** `OE_PC`, `ALU_A_WE`.
* **$T_{12}$:** `ALU_OE(ADD)`, `WE_PC`, Save $C_{\text{out}} \rightarrow C_{\text{fetch}}$.
* **$T_{13}$:** `OE_SIGN_EXT(IR_opr[3])`, `WE_B` (Drives `0x0` if positive, `0xF` if negative).
* **$T_{14}$:** `OE_PR`, `ALU_A_WE`.
* **$T_{15}$:** `ALU_OE(ADC)`, `WE_PR`, `RESET_SEQ`.

#### `0x7 JMPF` (Far Jump)

* **$T_{10}$:** `OE_C`, `WE_PR`.
* **$T_{11}$:** `OE_D`, `WE_PC`, `RESET_SEQ`.

#### `0x8 CALL rel_off` (Subroutine Relative Call)

* **$T_{10}$:** `OE_PC`, `STACK_WE` ($\text{RAM}[SP] \leftarrow PC$).
* **$T_{11}$:** `OE_SP`, `ALU_A_WE`.
* **$T_{12}$:** `ALU_OE(INC)`, `WE_SP`.
* **$T_{13}$:** `OE_PR`, `STACK_WE` ($\text{RAM}[SP] \leftarrow PR$).
* **$T_{14}$:** `OE_SP`, `ALU_A_WE`.
* **$T_{15}$:** `ALU_OE(INC)`, `WE_SP`.
* **$T_{16}$:** `OE_IR_opr`, `WE_B`.
* **$T_{17}$:** `OE_PC`, `ALU_A_WE`.
* **$T_{18}$:** `ALU_OE(ADD)`, `WE_PC`, Save $C_{\text{out}} \rightarrow C_{\text{fetch}}$.
* **$T_{19}$:** `OE_SIGN_EXT(IR_opr[3])`, `WE_B`.
* **$T_{20}$:** `OE_PR`, `ALU_A_WE`.
* **$T_{21}$:** `ALU_OE(ADC)`, `WE_PR`, `RESET_SEQ`.

#### `0x9 CALLF` (Subroutine Far Call)

* **$T_{10}$:** `OE_PC`, `STACK_WE` ($\text{RAM}[SP] \leftarrow PC$).
* **$T_{11}$:** `OE_SP`, `ALU_A_WE`.
* **$T_{12}$:** `ALU_OE(INC)`, `WE_SP`.
* **$T_{13}$:** `OE_PR`, `STACK_WE` ($\text{RAM}[SP] \leftarrow PR$).
* **$T_{14}$:** `OE_SP`, `ALU_A_WE`.
* **$T_{15}$:** `ALU_OE(INC)`, `WE_SP`.
* **$T_{16}$:** `OE_C`, `WE_PR`.
* **$T_{17}$:** `OE_D`, `WE_PC`, `RESET_SEQ`.

#### `0xA RET` (Subroutine Return)

* **$T_{10}$:** `OE_SP`, `ALU_A_WE`.
* **$T_{11}$:** `ALU_OE(DEC)`, `WE_SP`.
* **$T_{12}$:** `STACK_OE` $\rightarrow$ `WE_PR` ($PR \leftarrow \text{RAM}[SP]$).
* **$T_{13}$:** `OE_SP`, `ALU_A_WE`.
* **$T_{14}$:** `ALU_OE(DEC)`, `WE_SP`.
* **$T_{15}$:** `STACK_OE` $\rightarrow$ `WE_PC`, `RESET_SEQ` ($PC \leftarrow \text{RAM}[SP]$).

#### `0xC SKP cond`

* *Condition False:* **$T_{10}$:** `RESET_SEQ`.
* *Condition True:*
* **$T_{10}$:** `OE_PC`, `ALU_A_WE`.
* **$T_{11}$:** `ALU_OE(INC)`, `WE_PC`, Save $C_{\text{out}} \rightarrow C_{\text{fetch}}$.
* **$T_{12}$:** `OE_PR`, `ALU_A_WE`.
* **$T_{13}$:** `ALU_OE(ADC 0)`, `WE_PR`.
* **$T_{14}$:** `OE_PC`, `ALU_A_WE`.
* **$T_{15}$:** `ALU_OE(INC)`, `WE_PC`, Save $C_{\text{out}} \rightarrow C_{\text{fetch}}$.
* **$T_{16}$:** `OE_PR`, `ALU_A_WE`.
* **$T_{17}$:** `ALU_OE(ADC 0)`, `WE_PR`, `RESET_SEQ`.



#### `0xD PUSH target`

* **$T_{10}$:** `OE_target`, `STACK_WE` ($\text{RAM}[SP] \leftarrow target$).
* **$T_{11}$:** `OE_SP`, `ALU_A_WE`.
* **$T_{12}$:** `ALU_OE(INC)`, `WE_SP`, `RESET_SEQ`.

#### `0xE POP target`

* **$T_{10}$:** `OE_SP`, `ALU_A_WE`.
* **$T_{11}$:** `ALU_OE(DEC)`, `WE_SP`.
* **$T_{12}$:** `STACK_OE` $\rightarrow$ `WE_target`, `RESET_SEQ` ($target \leftarrow \text{RAM}[SP]$).

#### `0xF HLT`

* **$T_{10}$:** Asserts `MICROSTEP_ENABLE = 0`. The sub-tick generator and Johnson counter freeze at $T_{10}$ Sub-Tick 11. Master oscillator continues running freely.

---

## 7. Master Micro-Step Count & Sequencer Summary Table

| Opcode | Mnemonic | Fetch Steps | Execution Steps | Total Micro-Steps | Execution Active Window | Sequencer Action |
| --- | --- | --- | --- | --- | --- | --- |
| **`0x0`** | **`NOP`** | 10 | 1 | **11** | $T_{10}$ | Resets at $T_{10}$ |
| **`0x1`** | **`MOV`** | 10 | 1 | **11** | $T_{10}$ | Resets at $T_{10}$ |
| **`0x2`** | **`LDI`** | 10 | 1 | **11** | $T_{10}$ | Resets at $T_{10}$ |
| **`0x3`** | **`LD`** | 10 | 1 or 5 | **11 or 15** | $T_{10}\dots T_{14}$ | Resets at $T_{10}$ or $T_{14}$ |
| **`0x4`** | **`ST`** | 10 | 1 or 5 | **11 or 15** | $T_{10}\dots T_{14}$ | Resets at $T_{10}$ or $T_{14}$ |
| **`0x5`** | **`ALU`** | 10 | 2 | **12** | $T_{10}\dots T_{11}$ | Resets at $T_{11}$ |
| **`0x6`** | **`JMP`** | 10 | 6 | **16** | $T_{10}\dots T_{15}$ | Resets at $T_{15}$ |
| **`0x7`** | **`JMPF`** | 10 | 2 | **12** | $T_{10}\dots T_{11}$ | Resets at $T_{11}$ |
| **`0x8`** | **`CALL`** | 10 | 12 | **22 (WORST CASE)** | $T_{10}\dots T_{21}$ | **Resets at $T_{21}$** |
| **`0x9`** | **`CALLF`** | 10 | 8 | **18** | $T_{10}\dots T_{17}$ | Resets at $T_{17}$ |
| **`0xA`** | **`RET`** | 10 | 6 | **16** | $T_{10}\dots T_{15}$ | Resets at $T_{15}$ |
| **`0xB`** | *Open* | — | — | — | — | Reserved for System/IO |
| **`0xC`** | **`SKP`** | 10 | 1 or 8 | **11 or 18** | $T_{10}\dots T_{17}$ | Resets at $T_{10}$ or $T_{17}$ |
| **`0xD`** | **`PUSH`** | 10 | 3 | **13** | $T_{10}\dots T_{12}$ | Resets at $T_{12}$ |
| **`0xE`** | **`POP`** | 10 | 3 | **13** | $T_{10}\dots T_{12}$ | Resets at $T_{12}$ |
| **`0xF`** | **`HLT`** | 10 | $\infty$ | $\infty$ | $T_{10}$ | Suspends Microstepping |

---

## 8. Sequencer & Preliminary Transistor Budget

### Sequencer Subsystems

1. **2-Bit Sub-Tick Generator & Decoder:**
* **Hardware:** 2 D flip-flops (~8 transistors) + 2 decoding gates (~4 transistors) = **~12 Transistors**.


2. **11-Stage Johnson Counter ($T_0 \dots T_{21}$):**
* **Hardware:** 11 D flip-flops (~44 transistors) with inverted feedback ($Q_{10} \rightarrow \overline{Q}_0$).
* **Fault Recovery:** 2-input NOR gate checking illegal state combinations ($Q_0 \dots Q_{10}$) to guarantee self-correction within 10 clock cycles = **~4 Transistors**.


3. **Diode Matrix Step Decoder:**
* Simple 2-input diode AND gates decode adjacent flip-flop states into 22 unique 1-hot step lines ($T_0 \dots T_{21}$).



### Preliminary Transistor-Level Estimate (Core CPU Logic)

| Subsystem | Components | Estimated Transistors |
| --- | --- | --- |
| **Register File** | $A, B, C, D, PC, PR, SP, IR_{\text{op}}, IR_{\text{opr}}$ ($9 \times 4\text{ bits} = 36\text{ FFs}$) | ~144 |
| **Status Registers** | $FLAGS$ (3 bits) + $C_{\text{fetch}}$ shadow latch (1 bit) | ~16 |
| **ALU Shadow Latch** | `ALU_A_LATCH` (4 bits) | ~16 |
| **Central 4-Bit ALU** | Adder/Logic core, carry chain, B-inverter, muxes | ~96 |
| **Sequencer & Clocking** | 11-Stage Johnson counter, fault recovery, sub-tick generator | ~60 |
| **Control ROM & Decoders** | Diode ROM matrix drivers, step decoders, opcode logic | ~180 |
| **Bus Drivers & MUXes** | Open-drain pull-down transistors, `MEM_ADDR` 2:1 MUX | ~80 |
| **Miscellaneous** | Reset gating, `HLT` latch, sign-extension driver | ~40 |
| **TOTAL PRELIMINARY ESTIMATE** | **Core CPU Logic Only** | **~632 Transistors** |

*Conclusion:* The core CPU logic comfortably satisfies the **$< 1,000$ transistor target** budget with a ~35% safety margin, validating the architectural efficiency of the ephemeral ALU latch and single shared bus topology.