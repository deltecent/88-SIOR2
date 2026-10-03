# SIOA-REV-2 Technical Reference

## Overview

The SIOA-REV-2 (Serial I/O Adapter, Revision 2) is an Altair bus peripheral card designed for MITS Altair 8800 and compatible systems. It provides one full-duplex RS-232 serial port with:

- Programmable baud rates from 110 to 19,200 baud (via 8-position DIP switch)
- Programmable word format: 5–8 data bits, 1–2 stop bits, odd/even/no parity (via 5-position DIP switch)
- Configurable I/O port address (via 7-position DIP switch)
- Interrupt-driven operation with selectable interrupt vector (VI0–VI7)
- RS-232 DCE and DTE connectors (DB25 and DE9) plus an FTDI-compatible 6-pin TTL header
- On-board +5V regulation from the Altair +8V supply, plus −12V generation from the Altair −16V supply for the UART negative rail

The board interfaces to the Altair (S-100) bus through the 100-pin edge connector. The CPU (8080 or Z80) communicates with the card using two I/O port addresses: one for data and one for status/control.

---

## Project Resources

The SIOA-REV-2 project is hosted on GitHub at **https://github.com/deltecent/88-SIOR2**.

There you will find:

- **Test programs** — software for verifying board operation
- **Wiki** — additional documentation and usage notes
- **Issue tracker** — report bugs or request improvements

---

## IC Descriptions

### U1 — MAX232 (RS-232 Level Converter, DIP-16)

**Function:** Converts between TTL/CMOS logic levels (0/5V) and RS-232 voltage levels (±10V). Requires only a +5V supply; internal charge pumps generate ±10V for RS-232 drivers.

Contains two RS-232 line drivers and two RS-232 line receivers.

| Pin(s) | Name | Direction | Description |
|--------|------|-----------|-------------|
| 1, 3   | C1+, C1− | — | External charge pump capacitor (1µF) for +10V generator |
| 4, 5   | C2+, C2− | — | External charge pump capacitor (1µF) for −10V generator |
| 2      | V+   | Out | +10V internal rail |
| 6      | V−   | Out | −10V internal rail |
| 11, 10 | T1IN, T2IN | In | TTL inputs to RS-232 drivers |
| 14, 7  | T1OUT, T2OUT | Out | RS-232 outputs (+/−10V), connected to TXD line |
| 13, 8  | R1IN, R2IN | In | RS-232 inputs (from RXD line) |
| 12, 9  | R1OUT, R2OUT | Out | TTL outputs from RS-232 receivers |
| 15     | GND  | — | Ground |
| 16     | VCC  | — | +5V supply |

**In this design:**

- T1IN receives TSO (Transmitter Serial Output) from U2 UART
- T1OUT drives TXD line to RS-232 connectors (J2, J3, J4, J5)
- R1IN receives RXD from RS-232 connectors
- R1OUT drives RSI (Receiver Serial Input) to U2 UART

> **Important — TTL header use:** U1's R1OUT drives the same RSI net that the J1 TTL header exposes. **U1 must be removed from its socket when using the J1 TTL header** (e.g. with an FTDI USB-serial adapter), otherwise U1's receiver output contends with the external device driving RSI. With U1 removed, RS-232 operation on J2–J5 is disabled and only the TTL header is active.

---

### U2 — COM2502 (UART, DIP-40)

**Function:** Universal Asynchronous Receiver/Transmitter (AY-5-1013 family). Performs parallel-to-serial conversion for transmission and serial-to-parallel conversion for reception. Includes programmable word format (data bits, stop bits, parity), status flags, and tristate data outputs. The COM2502 requires a dual supply: +5V on VCC (pin 1) and **−12V on VGG (pin 2)**. The −12V rail is generated on-board from the Altair −16V supply (see Power Supply). The pin-compatible CMOS Intersil IM6402 runs from +5V only and leaves pin 2 unconnected.

The UART operates at 16× the desired baud rate on its clock inputs.

#### Compatible UARTs

This socket accepts any of the standard 40-pin AY-5-1013-pinout UARTs. The only wiring difference between them is the supply on pin 2 (VGG): the older PMOS parts require −12V there, while the later +5V-only (NMOS/CMOS) parts leave pin 2 unconnected. Install jumper **JP2** to enable the on-board −12V supply only for the parts that need it; leave JP2 open for the +5V-only parts.

| UART | −12V jumper (JP2) |
|------|:-----------------:|
| General Instrument AY-5-1012, AY-5-1013A | Required |
| Texas Instruments TMS6011 | Required |
| AMI S1883 | Required |
| SMC COM2502, COM2502H, COM2017 | Required |
| SMC COM8502, COM8017 | Not required (+5V only) |
| Western Digital TR1402, TR1602 | Required |
| Western Digital TR1863, TR1865 | Not required (+5V only) |
| Intersil IM6402 | Not required (+5V only) |

None of these parts require +12V — the dual-supply (PMOS) parts use +5V and −12V only, which is exactly what the board provides. On the +5V-only parts, pin 2 (VGG) is a no-connect, so leave JP2 open to avoid feeding −12V to an unused pin.

#### Pin Summary

| Pin | Name | Dir | Description |
|-----|------|-----|-------------|
| 1   | VCC  | In  | +5V supply |
| 2   | VGG  | In  | −12V supply (from on-board −16V→−12V regulator) |
| 3   | GND  | In  | Ground |
| 4   | /RDE | In | Receiver Data Enable (active low): enables RBR outputs onto bDI bus |
| 5–12 | RBR8–RBR1 | Out | Receiver Buffer Register, bits 8–1 (MSB first). Tristate; enabled by /RDE low. Also serve as status word outputs when /SWE low |
| 13  | PE   | Out | Parity Error (high = parity error detected). Tristate; enabled when /SWE (pin 16) is low; connected to bDI2 |
| 14  | FE   | Out | Framing Error (high = invalid stop bit). Tristate; enabled when /SWE (pin 16) is low; connected to bDI3 |
| 15  | OVR  | Out | Overrun Error (high = previous character not read). Tristate; enabled when /SWE (pin 16) is low; connected to bDI4 |
| 16  | /SWE | In | Status Word Enable (active low): enables all status outputs — RBR pins 5–12, PE, FE, OVR pins 13–15, DR, and TBMT. Connected to RDE; goes low on any non-data-read cycle |
| 17  | RC   | In  | Receiver Clock input (16× baud rate) |
| 18  | /RDAV | In | Receiver Data Available (tied to DR, active low, clears DR flag when read) |
| 19  | DR   | Out | Data Ready (high = received character is in RBR, ready to read) |
| 20  | RSI  | In  | Receiver Serial Input (TTL-level serial data in) |
| 21  | MR   | In  | Master Reset (active high; clears all status, disables outputs) |
| 22  | TBMT | Out | Transmitter Buffer Empty (high = TBR can accept new character) |
| 23  | /TDS | In | Transmitter Data Strobe (active low pulse: loads TBR from TBR1–TBR8) |
| 24  | TRE  | Out | Transmitter Register Empty (high = shift register is empty) |
| 25  | TSO  | Out | Transmitter Serial Output (serial data output) |
| 26–33 | TBR1–TBR8 | In | Transmitter Buffer Register inputs, bits 1–8 (LSB first) |
| 34  | CS   | In  | Control Strobe (active high pulse: loads NDB1, NDB2, NSB, NPB, POE into control register) |
| 35  | NPB  | In  | No Parity Bit (high = no parity bit transmitted/checked) |
| 36  | NSB  | In  | Number of Stop Bits (0 = 1 stop bit; 1 = 2 stop bits, or 1.5 for 5-bit words) |
| 37  | NDB2 | In  | Number of Data Bits bit 2 (MSB of word length select) |
| 38  | NDB1 | In  | Number of Data Bits bit 1 (LSB of word length select) |
| 39  | POE  | In  | Parity Odd/Even select (0 = even parity; 1 = odd parity) |
| 40  | TC   | In  | Transmitter Clock input (16× baud rate) |

#### Word Length Truth Table (NDB1, NDB2)

| NDB2 | NDB1 | Data Bits |
|------|------|-----------|
|  0   |  0   | 5 bits    |
|  0   |  1   | 6 bits    |
|  1   |  0   | 7 bits    |
|  1   |  1   | 8 bits    |

#### Stop Bit Truth Table (NSB)

| NSB | Stop Bits |
|-----|-----------|
| 0   | 1 stop bit |
| 1   | 2 stop bits (1.5 stop bits when word length = 5 data bits) |

#### Parity Truth Table (NPB, POE)

NPB selects whether a parity bit is generated/checked; POE selects odd vs. even and is a don't-care when parity is disabled.

| NPB | POE | Parity |
|-----|-----|--------|
| 1   |  X  | No parity bit (disabled) |
| 0   |  0  | Even parity |
| 0   |  1  | Odd parity |

#### Status Bit Summary

| Signal | Pin | Meaning when High |
|--------|-----|-------------------|
| DR  | 19 | Data Ready: received character is available in RBR |
| TBMT | 22 | TX Buffer Empty: transmitter is ready to accept a new character |
| TRE | 24 | TX Register Empty: transmitter shift register has finished sending |
| PE  | 13 | Parity Error: last received character had a parity error |
| FE  | 14 | Framing Error: last received character had an invalid stop bit |
| OVR | 15 | Overrun: received character lost (DR was not cleared before next arrived) |

---

### U7 — 74LS08 (Quad 2-Input AND Gate, DIP-14)

**Function:** Four independent 2-input AND gates. Used to combine the two port-decoded read/write strobes into unified bus-enable signals.

#### Truth Table (each gate)

| A | B | Y = A AND B |
|---|---|-------------|
| 0 | 0 | 0 |
| 0 | 1 | 0 |
| 1 | 0 | 0 |
| 1 | 1 | 1 |

**Gate A:** /DIEN = /SWE AND /RDE
  - Goes LOW (active) when either a status read or data read is occurring
  - Enables U13 (74LS244) to drive the Altair DI bus during any read cycle

**Gates B, C and D:** Unused; inputs tied off, outputs unconnected. (U14, the DO-bus buffer, is permanently enabled and no longer requires a /DOEN strobe.)

---

### U15 — 74LS368 (Hex Inverting Tristate Buffer, DIP-16)

**Function:** Six independent inverting buffers with tristate outputs (two banks of three, each with an independent active-low enable).

| /OE | Input | Output |
|-------|-------|--------|
|  0    |  0    |   1    |
|  0    |  1    |   0    |
|  1    |  X    |   Z (tristate) |

**Bank A (/OEa = GND, always enabled):**

- Buffer 1: DO0 → /D0 (inverted DO0; feeds U8 Gates A and D)
- Buffer 2: DO1 → /D1 (inverted DO1; feeds U8 Gates B and C)
- Buffer 3: /CTL → CTL (active-high CTL strobe; drives U2 CS pin 34 and U10, U16 NAND inputs)
- Buffer 4: unused

**Bank B (/OEb = /SWE from U6, enabled only during status read):**

- Buffer 5: TBMT (from U2 pin 22) → bDI7 = NOT(TBMT) (DI bus bit 7)
- Buffer 6: DR (from U2 pin 19) → bDI0 = NOT(DR) (DI bus bit 0)

---

### U3, U8, U10, U16 — 74LS00 (Quad 2-Input NAND Gate, DIP-14)

**Function:** Four 74LS00 packages, each containing four independent 2-input NAND gates, are used throughout the design:

- **U3** — 110-baud counter reload and the active-high RDE signal
- **U8** — set/clear inputs for the U9 interrupt-enable latches
- **U10** — interrupt qualification and combination
- **U16** — bus-strobe generation (/IO_WR, /IO_RD) and buffering the bus power-on-clear into the UART reset

#### Truth Table (each gate)

| A | B | Y = NAND(A,B) |
|---|---|---------------|
| 0 | 0 | 1 |
| 0 | 1 | 1 |
| 1 | 0 | 1 |
| 1 | 1 | 0 |

#### U3 — baud reload and RDE

**Gates A and B:** Unused; inputs tied off, outputs unconnected. (These gates drove the receive/transmit status LEDs in the prototype; the LEDs have been removed in this revision.)

**Gate C:** NAND(B110, U4 Q2) → U4 /PE (110-baud counter reload)

- When U4 Q3 (=B110) and Q2 are both high (count = 12–15): output = 0 → U4 loads preset = 2
- Implements the divide-by-11 feedback for 110-baud generation

**Gate D:** NAND(/RDE, +5V) = NOT(/RDE) → RDE (active-high data-read signal)

- Simply inverts the decoded active-low data-read signal to produce active-high RDE
- RDE drives U2 pin 16 (/SWE): when RDE=0 (not a data-port read), /SWE activates and U2 status outputs are enabled; when RDE=1 (data-port read), /SWE is inhibited to avoid conflict with RBR outputs

#### U8 — interrupt latch set/clear

U8 generates the /S and /R inputs for U9 based on DO0, DO1, and the CTL strobe from U15 (confirmed from netlist):

- Gate A: NAND(CTL, /D0) → /CLR_IN — drives U9 FF1 /R when CTL write and DO0=0
- Gate B: NAND(CTL, DO1) → /SET_OUT — drives U9 FF2 /S when CTL write and DO1=1
- Gate C: NAND(CTL, /D1) → /CLR_OUT — drives U9 FF2 /R when CTL write and DO1=0
- Gate D: NAND(CTL, DO0) → /SET_IN — drives U9 FF1 /S when CTL write and DO0=1

#### U10 — interrupt qualification

U10 qualifies the U9 interrupt-enable latches with the live UART status flags and combines them, producing the three interrupt-request signals routed to the jumper field (confirmed from netlist):

- Gate A: NAND(DR, Q1) → /IN_INT — RX interrupt request: goes low when a character is ready (DR=1) **and** the input interrupt is enabled (U9 FF1 Q1=1)
- Gate B: NAND(TBMT, Q2) → /OUT_INT — TX interrupt request: goes low when the transmitter can accept data (TBMT=1) **and** the output interrupt is enabled (U9 FF2 Q2=1)
- Gate C: NAND(/IN_INT, /OUT_INT) → intermediate
- Gate D: NAND(intermediate, +5V) = NOT(intermediate) → /BH_INT — combined request: /BH_INT = /IN_INT AND /OUT_INT, so it goes low when **either** /IN_INT or /OUT_INT is active

#### U16 — bus strobes and power-on clear

U16 generates /IO_WR and /IO_RD from the processor status/control signals, and buffers the bus Power-On Clear (POC, Altair bus pin 99) into the UART master reset.

**Gate A:** NAND(/pWR, /pWR) = NOT(/pWR) → pWR (intermediate)

- Inverts the processor active-low write strobe (/pWR, CPU control signal) to produce an active-high intermediate signal

**Gate B:** NAND(pWR, sOUT) → /IO_WR

- /IO_WR goes low (active) when a write cycle (pWR=1) and the processor output-cycle status (sOUT=1) are both asserted
- /IO_WR drives U6 (74LS138) to decode I/O write operations

**Gate C:** NAND(/POC, /POC) = NOT(/POC) → POC

- Inverts the active-low bus power-on-clear signal (/POC, from Altair bus pin 99) to produce active-high POC
- POC drives U2 pin 21 (MR, Master Reset) high at power-up to reset the UART

**Gate D:** NAND(pDBIN, sINP) → /IO_RD

- /IO_RD goes low (active) when the processor data-bus-in strobe (pDBIN=1, CPU control signal) and input-cycle status (sINP=1) are both asserted
- /IO_RD drives U6 (74LS138) to decode I/O read operations

---

### U4 — 74LS161 (Synchronous 4-Bit Binary Counter, DIP-16)

**Function:** Synchronous 4-bit binary counter with synchronous parallel load and asynchronous clear. Generates the **B110** baud clock (≈1,745 Hz ≈ 16 × 109 baud) by dividing the B1200 clock by approximately 11. The 110 baud rate cannot be obtained by binary division of the 1.2288 MHz oscillator, so this programmable divide-by-N counter is used.

| Pin | Name | Net | Description |
|-----|------|-----|-------------|
| 1   | /CLR | +5V | Asynchronous clear disabled (tied high) |
| 2   | CLK  | B1200 | Clock input: 19,200 Hz from U5 Counter B Q1 (pin 10) |
| 3   | D0   | GND | Parallel data bit 0 = 0 |
| 4   | D1   | +5V | Parallel data bit 1 = 1 |
| 5   | D2   | GND | Parallel data bit 2 = 0 |
| 6   | D3   | GND | Parallel data bit 3 = 0 → preset value = 0b0010 = 2 |
| 7   | CEP  | +5V | Count enable (always counting) |
| 9   | /PE | U3 Gate C | Synchronous parallel load (active low); driven by NAND(Q3, Q2) |
| 10  | CET  | +5V | Count enable terminal (always counting) |
| 11  | Q3   | B110 | MSB output → B110 ≈ 109 baud 16× clock |
| 12  | Q2   | Net-(U4-Q2) | Feedback to U3 Gate C for reload trigger |
| 13  | Q1   | NC  | Unconnected |
| 14  | Q0   | NC  | Unconnected |
| 15  | TC   | NC  | Terminal count, unconnected |

**Divide-by-11 operation:** The counter starts at preset value 2 (loaded on reset or reload). It counts 2→3→4→…→12. When the count reaches 12, Q3=1 AND Q2=1, so U3 Gate C output = NAND(Q3, Q2) = 0 → /PE goes low → the counter synchronously loads preset value 2 on the next rising clock edge. This gives a cycle length of 11 clock periods.

```
19,200 Hz ÷ 11 ≈ 1,745 Hz = B110 (≈ 109 baud × 16)
```

The 0.8% error from ideal 110 baud is within RS-232 tolerance.

---

### U5 — 74LS393 (Dual 4-Bit Binary Counter, DIP-14)

**Function:** Two independent 4-bit ripple binary counters in a single package. Counter A is clocked directly by the 1.2288 MHz crystal oscillator (X1). Counter A's Q3 output (B4800) clocks Counter B. Together they generate seven of the eight available baud rate clocks (B300–B19200); U4 generates the eighth (B110).

#### Baud Rate Outputs from U5 (1.2288 MHz input)

| Counter | Pin | Pinfunction | ÷ Factor | Frequency  | Net Name |
|---------|-----|-------------|----------|------------|----------|
| A       | 1   | CP_1  | (input) | 1,228,800 Hz | (oscillator) |
| A       | 2   | MR_2  | reset   | —          | GND (never resets) |
| A       | 3   | Q0_3  | ÷2      | 614,400 Hz | **Unconnected** (not used) |
| A       | 4   | Q1_4  | ÷4      | 307,200 Hz | **B19200** (16 × 19,200) |
| A       | 5   | Q2_5  | ÷8      | 153,600 Hz | **B9600** (16 × 9,600) |
| A       | 6   | Q3_6  | ÷16     | 76,800 Hz  | **B4800** (16 × 4,800) → clocks Counter B |
| B       | 13  | CP_13 | (input) | 76,800 Hz  | (from Counter A Q3) |
| B       | 12  | MR_12 | reset   | —          | GND (never resets) |
| B       | 11  | Q0_11 | ÷2      | 38,400 Hz  | **B2400** (16 × 2,400) |
| B       | 10  | Q1_10 | ÷4      | 19,200 Hz  | **B1200** (16 × 1,200) → clocks U4 |
| B       | 9   | Q2_9  | ÷8      | 9,600 Hz   | **B600** (16 × 600) |
| B       | 8   | Q3_8  | ÷16     | 4,800 Hz   | **B300** (16 × 300) |

---

### U11, U12 — 74LS85 (4-Bit Magnitude Comparator, DIP-16)

**Function:** Compares two 4-bit values (A and B) and asserts one of three outputs: A>B, A=B, or A<B. Two 74LS85 chips are cascaded to compare address bits **A1–A7** (7 bits) against the value programmed on SW3 (port address DIP switch). A0 is not compared here — it is decoded by U6 to distinguish the two port addresses. When A1–A7 match SW3, the board-select signal (BDSEL) is asserted, enabling the address decoder (U6).

**U11** compares the lower-order address bits **A1–A4** against **SA1–SA4**:

- A inputs: A3=A1, A2=A2, A1=A3, A0=A4
- B inputs: B3=SA1, B2=SA2, B1=SA3, B0=SA4
- Cascade inputs: Ia<b=GND, Ia=b=+5V, Ia>b=GND (initial "equal" seed)
- Oa=b output → U12 cascade input Ia=b

**U12** compares the upper-order address bits **A5–A7** against **SA5–SA7** (A3 bit position tied to GND = always-match):

- A inputs: A3=GND, A2=A5, A1=A6, A0=A7
- B inputs: B3=GND, B2=SA5, B1=SA6, B0=SA7
- Cascade input Ia=b: receives U11 Oa=b (previous bits all equal)
- Cascade inputs Ia<b=GND, Ia>b=GND
- **Oa=b output → BDSEL** (board select, active high)

#### Truth Table (cascaded pair, relevant outputs only)

| A[7:1] vs. SW3 | Oa>b | Oa=b (BDSEL) | Oa<b |
|----------------|------|--------------|------|
| Address > SW3  |  1   |  0           |  0   |
| Address = SW3  |  0   |  **1**       |  0   |
| Address < SW3  |  0   |  0           |  1   |

Only Oa=b is used: when A1–A7 match SA1–SA7, BDSEL goes high and U6 is enabled.

---

### U13 — 74LS244 (Octal Tristate Buffer, DIP-20)

**Function:** Eight non-inverting tristate buffers in two banks of four, each with an independent active-low enable. Drives the Altair **DI bus** (data into the CPU) from the board's internal **bDI** bus.

Both banks (OEa and OEb) are controlled by **/DIEN** from U7 Gate A. /DIEN goes active (low) when either a status read or a data read is occurring, enabling all eight buffers simultaneously.

| /OE | Input | Output |
|-------|-------|--------|
|  0    |  0    |   0    |
|  0    |  1    |   1    |
|  1    |  X    |   Z (tristate) |

**Signal mapping (non-inverting — output = input):**

| bDI Bus (input) | U13 Pin In | U13 Pin Out | Altair DI (output) |
|-----------------|------------|-------------|-------------------|
| bDI0 | I0b (pin 11) | O0b (pin 9)  | DI0 = bDI0 |
| bDI1 | I3a (pin 8)  | O3a (pin 12) | DI1 = bDI1 |
| bDI2 | I1b (pin 13) | O1b (pin 7)  | DI2 = bDI2 |
| bDI3 | I2a (pin 6)  | O2a (pin 14) | DI3 = bDI3 |
| bDI4 | I2b (pin 15) | O2b (pin 5)  | DI4 = bDI4 |
| bDI5 | I1a (pin 4)  | O1a (pin 16) | DI5 = bDI5 |
| bDI6 | I3b (pin 17) | O3b (pin 3)  | DI6 = bDI6 |
| bDI7 | I0a (pin 2)  | O0a (pin 18) | DI7 = bDI7 |

**During status read (/SWE active, RDE=0):** All U2 status outputs are enabled. bDI0 = NOT(DR) and bDI7 = NOT(TBMT) via U15 Bank B; bDI2, bDI3, bDI4 driven directly by PE, FE, OVR (pins 13–15); remaining bits from U2 status word outputs on pins 5–12.

**During data read (/RDE active, RDE=1):** /SWE=1 — all U2 status outputs (DR, TBMT, PE, FE, OVR) are high-Z. bDI0–7 carry RBR receive data only.

---

### U14 — 74LS244 (Octal Tristate Buffer, DIP-20)

**Function:** Eight **non-inverting** tristate buffers. Passes the Altair **DO bus** (data output from the CPU) through to the board's internal **bDO bus** unchanged — output logic level equals input logic level. The bDO bus connects to U2 TBR1–TBR8 for transmit data.

Both bank enables (OEa and OEb) are tied to GND, so U14 is **permanently enabled** — the bDO bus continuously mirrors the Altair DO bus. This is harmless because the UART only latches the bDO bus when the /TDS data-write strobe pulses; no separate /DOEN gating is required.

**Signal mapping (non-inverting — output = input):**

| Altair DO (input) | U14 Pin In | U14 Pin Out | bDO Bus (output) |
|------------------|------------|-------------|------------------|
| DO0 | I0b (pin 11) | O0b (pin 9)  | bDO0 = DO0 |
| DO1 | I3a (pin 8)  | O3a (pin 12) | bDO1 = DO1 |
| DO2 | I1b (pin 13) | O1b (pin 7)  | bDO2 = DO2 |
| DO3 | I2a (pin 6)  | O2a (pin 14) | bDO3 = DO3 |
| DO4 | I2b (pin 15) | O2b (pin 5)  | bDO4 = DO4 |
| DO5 | I1a (pin 4)  | O1a (pin 16) | bDO5 = DO5 |
| DO6 | I3b (pin 17) | O3b (pin 3)  | bDO6 = DO6 |
| DO7 | I0a (pin 2)  | O0a (pin 18) | bDO7 = DO7 |

> **Note:** DO0 and DO1 also connect **directly** (without going through U14) to U15 Bank A inputs. U15 Bank A **inverts** them to produce /D0 and /D1 for interrupt flip-flop control. U14 itself performs no inversion — the bDO bus is a buffered, logic-identical copy of the DO bus.

---

### U6 — 74LS138 (3-to-8 Line Decoder, DIP-16)

**Function:** Decodes three inputs into one of eight active-low outputs, subject to three enable inputs. Once the board is selected (BDSEL active), it decodes A0, /IO_WR, and /IO_RD to produce **four** qualified port-select signals — one for each of the four distinct operations (status read, data read, control write, data write).

**Pin connections:**

- Pin 1 (A0): A0 — distinguishes status/control port (A0=0) from data port (A0=1)
- Pin 2 (A1): /IO_WR — active-low write strobe (HIGH = not writing; LOW = writing)
- Pin 3 (A2): /IO_RD — active-low read strobe (HIGH = not reading; LOW = reading)
- Pin 4 (E1): GND — active-low enable, always enabled
- Pin 5 (E2): GND — active-low enable, always enabled
- Pin 6 (E3): BDSEL — active-high enable; only active when A1–A7 match SW3

#### Port Decode Truth Table

| BDSEL | /IO_RD | /IO_WR | A0 | Active Output | Net Name | Function |
|-------|----------|----------|----|---------------|----------|----------|
|  1    |    0     |    1     |  0 | O2 (pin 13)   | /SWE  | **Status read** |
|  1    |    0     |    1     |  1 | O3 (pin 12)   | /RDE  | **Data read** |
|  1    |    1     |    0     |  0 | O4 (pin 11)   | /CTL  | **Control write** |
|  1    |    1     |    0     |  1 | O5 (pin 10)   | /TDS  | **Data write** |
|  0    |    X     |    X     |  X | None (all high) | —       | Board not selected |

Outputs O0, O1, O6, O7 are never asserted in normal operation (invalid or unused address combinations). When BDSEL is inactive, all outputs remain high (inactive).

---

### U9 — 74LS74 (Dual D Flip-Flop with Set and Reset, DIP-14)

**Function:** Two independent positive-edge-triggered D flip-flops, each with asynchronous active-low set (/S) and reset (/R) inputs. Both D inputs and both CLK inputs are tied to GND, making the flip-flops operate in **asynchronous-only** mode: the Q output is controlled exclusively by /S (set) and /R (reset), never by the clock. The two Q outputs are the interrupt-enable latches; they feed U10, which qualifies them with the UART status flags and drives the Altair interrupt lines via JP3, JP4, and JP5.

#### Truth Table (asynchronous control only)

| /S | /R | Q | /Q  |
|------|------|---|------|
|  0   |  1   | 1 |  0   | (Set) |
|  1   |  0   | 0 |  1   | (Reset/Clear) |
|  0   |  0   | 1 |  1   | (Forbidden — avoid) |
|  1   |  1   | Q |  /Q  | (Hold) |

**FF1 — input (RX) interrupt enable:**

- **Set** (/S = /SET_IN low): CTL write with DO0=1
- **Cleared** (/R = /CLR_IN low): CTL write with DO0=0
- Q1 feeds U10 Gate A, where it is ANDed with DR to form the /IN_INT request

**FF2 — output (TX) interrupt enable:**

- **Set** (/S = /SET_OUT low): CTL write with DO1=1
- **Cleared** (/R = /CLR_OUT low): CTL write with DO1=0
- Q2 feeds U10 Gate B, where it is ANDed with TBMT to form the /OUT_INT request

---

## Functional Operation

### Power Supply

The Altair bus provides +8V unregulated on pins 1 and 51. VR1 (LM7805, TO-220) regulates this to +5V for all logic ICs. The MAX232 (U1) uses this +5V with internal charge pumps to generate the RS-232 ±10V levels — no external negative supply is needed for RS-232.

**−12V generation (UART VGG):** The COM2502 UART requires a −12V supply on its VGG pin (U2 pin 2). This is produced from the Altair −16V rail (bus pin 52) by a simple shunt (zener) regulator:

```
  Altair −16V ──► JP2 ──► R1 (130Ω) ──┬──► −12V ──► U2 pin 2 (VGG)
                                       │
                                  D1 (12V zener) ──► GND
                                       │
                                  C24 (47µF) ──► GND
```

R1 drops the difference between −16V and −12V; D1, a 12V zener from the −12V node to ground, clamps the rail at −12V; C24 filters it. **JP2** is a 2-pin jumper in series with the −16V feed — it must be installed to enable the −12V supply. If a CMOS IM6402 is fitted instead of the COM2502, the −12V rail is not required and JP2 may be left open.

C21 (47µF) and C23 (47µF) provide bulk decoupling on the +8V input and +5V output of VR1, respectively.

### Baud Rate Generation

```
X1 (1.2288 MHz)
     │
     └──► U5 Counter A (clocked directly at 1.2288 MHz)
               │
               ├─ Q0 (pin 3) ──► UNCONNECTED (614,400 Hz, not needed)
               ├─ Q1 (pin 4) ──► B19200 (307,200 Hz = 16 × 19,200)
               ├─ Q2 (pin 5) ──► B9600  (153,600 Hz = 16 × 9,600)
               └─ Q3 (pin 6) ──► B4800  (76,800 Hz = 16 × 4,800)
                                    │
                                    └──► U5 Counter B (clocked by B4800)
                                              │
                                              ├─ Q0 (pin 11) ──► B2400 (38,400 Hz)
                                              ├─ Q1 (pin 10) ──► B1200 (19,200 Hz) ──► U4 CLK
                                              ├─ Q2 (pin 9)  ──► B600  (9,600 Hz)
                                              └─ Q3 (pin 8)  ──► B300  (4,800 Hz)

B1200 (19,200 Hz)
     │
     └──► U4 (74LS161, divide by ≈11) ──► B110 (≈1,745 Hz ≈ 16 × 109 baud)
               Reload at count 12 via U3 Gate C feedback (preset = 2)
```

SW2 (8-position DIP switch) selects one of these eight baud rate clocks. The selected clock drives the **CLK** net, which connects to both RC (pin 17, Receiver Clock) and TC (pin 40, Transmitter Clock) of the UART (U2).

**SW2 Baud Rate Selection:**

| SW2 Position | Baud Rate |
|:------------:|-----------|
| 1            | 19,200    |
| 2            | 9,600     |
| 3            | 4,800     |
| 4            | 2,400     |
| 5            | 1,200     |
| 6            | 600       |
| 7            | 300       |
| 8            | 110       |

### Port Address Decode

The Altair bus carries an 8-bit I/O address (A0–A7) during IN and OUT instructions.

1. **U11, U12 (74LS85 cascade):** Compare address bits **A1–A7** against the value programmed on SW3 (7-position DIP switch). U11 compares A1–A4 vs SA1–SA4 (lower bits); U12 compares A5–A7 vs SA5–SA7 (upper bits). U11's equal output feeds U12's cascade input; U12's final A=B output asserts **BDSEL** (Board Select, active high). A0 is not part of this comparison.

2. **U6 (74LS138):** With BDSEL active as the G1 enable, U6 decodes A0, /IO_WR, and /IO_RD to produce exactly **four** port-select outputs:
   - **A0=0, reading:** /SWE (status read)
   - **A0=1, reading:** /RDE (data read)
   - **A0=0, writing:** /CTL (control write)
   - **A0=1, writing:** /TDS (data write)

**SW3 Port Address Configuration:**

SW3 sets address bits A1–A7 for the board's base address (A0 is always 0 at the base). Switch positions map as follows:

| SW3 Position | Address Bit |
|:------------:|-------------|
| 1 (pin 8)    | SA1 (A1)   |
| 2 (pin 9)    | SA2 (A2)   |
| 3 (pin 10)   | SA3 (A3)   |
| 4 (pin 11)   | SA4 (A4)   |
| 5 (pin 12)   | SA5 (A5)   |
| 6 (pin 13)   | SA6 (A6)   |
| 7 (pin 14)   | SA7 (A7)   |

SW3 selects any **even** base address from 0x00 to 0xFE (octal 000–376). The status/control port is always at the even address; the data port is at the next odd address (base + 1). A typical configuration with all switches closed places the card at port 0x00 (status/control at 0x00, data at 0x01), matching the original MITS Altair SIO.

### UART Format Configuration (SW1)

SW1 (5-position DIP switch) connects to U2 format control pins through resistor network RN1. A control port write (CS high) latches these settings into the UART:

| SW1 Position | UART Pin | Function |
|:------------:|----------|----------|
| 1 | NSB  (pin 36) | Stop bits (0=1, 1=2) |
| 2 | NDB1 (pin 38) | Word length bit 0 |
| 3 | NDB2 (pin 37) | Word length bit 1 |
| 4 | POE  (pin 39) | Parity (0=even, 1=odd) |
| 5 | NPB  (pin 35) | No parity (1=disabled) |

---

## Bus Operation Flows

### Status Read

A **status read** occurs when the CPU executes an IN instruction to the status/control port address (A0=0).

```
Signal path:

  A1–A7 ──► U11, U12 (74LS85) ──► BDSEL ──┐
  SW3   ──► (comparators)                   │
                                             ▼
  A0=0   ─────────────────────────────── U6 (74LS138) ──► /SWE
  /IO_RD ────────────────────────────────────────────          │
                                                               ├──► U15 Bank B (/OEb = 0):
                                                               │       DR   ──► NOT(DR)   = bDI0
                                                               │       TBMT ──► NOT(TBMT) = bDI7
                                                               │
                                                               └──► U3 Gate D: RDE = NOT(/RDE) = 0
                                                                      U2 pin 16 (/SWE = 0) ── all status enabled:
                                                                        PE  (pin 13) ──► bDI2
                                                                        FE  (pin 14) ──► bDI3
                                                                        OVR (pin 15) ──► bDI4
                                                                        status word  ──► bDI1, bDI5, bDI6

  /SWE=0 ────────────────────────────── U7 Gate A ──► /DIEN=0 ──► U13 ──► Altair DI0–DI7
```

1. CPU asserts the I/O address on A0–A7 and drives read strobes onto the Altair bus.
2. U11, U12 compare A1–A7 with SW3; if equal, **BDSEL** goes high.
3. U6 decodes A0=0 with read active → asserts **/SWE** (O2, active low).
4. /SWE enables **U15 Bank B**: NOT(DR) → bDI0; NOT(TBMT) → bDI7.
5. U3 Gate D derives **RDE=0** (not a data-port read) → U2 pin 16 (/SWE) = 0 → U2 enables all status outputs: status word on pins 5–12, PE (pin 13), FE (pin 14), OVR (pin 15), DR (pin 19), and TBMT (pin 22).
6. U7 Gate A: /DIEN = AND(/SWE, /RDE) → goes LOW → **U13** is enabled.
7. U13 drives bDI0–7 onto Altair **DI** bus (DI0–DI7).
8. CPU latches the status byte.

**Status byte bit assignments (on Altair DI bus):**

| DI Bit | Source | Logic |
|--------|--------|-------|
| 0 | U15 Bank B: NOT(DR) | **0 = data ready** (DR high); 1 = no data |
| 1 | U2 status word (bit unused) | — |
| 2 | U2 pin 13: PE (enabled by /SWE) | 0 = no parity error; **1 = parity error** |
| 3 | U2 pin 14: FE (enabled by /SWE) | 0 = no framing error; **1 = framing error** |
| 4 | U2 pin 15: OVR (enabled by /SWE) | 0 = no overrun; **1 = overrun** |
| 5 | U2 status word (TBMT) | — |
| 6 | U2 status word (DR) | — |
| 7 | U15 Bank B: NOT(TBMT) | **0 = TX ready** (TBMT high); 1 = TX busy |

Software polls bit 0 = 0 for receive ready, bit 7 = 0 for transmit ready, bits 2, 3, 4 for error conditions.

---

### Control Write

A **control write** occurs when the CPU executes an OUT instruction to the status/control port address (A0=0). This configures the UART word format and sets or clears the U9 flip-flop outputs that gate the interrupt drive circuitry.

```
Signal path:

  A1–A7  ──► U11, U12 (74LS85) ──► BDSEL ──┐
  SW3    ──► (comparators)                   │
                                              ▼
  A0=0   ─────────────────────────────── U6 (74LS138) ──► /CTL
  /IO_WR ────────────────────────────────────────────          │
                                                               └──► U15 Bank A: NOT(/CTL) = CTL
                                                                       CTL ──► U2 pin 34 (CS = 1)
                                                                              U2 latches SW1 format

  (U14 is permanently enabled; the bDO bus already mirrors Altair DO0–DO7)

  Altair DO0 ──► U15 Bank A ──► /D0 ──► U8 Gate A: NAND(CTL, /D0) ──► /CLR_IN  ──► U9 FF1 /R
  Altair DO0 ──────────────────────► U8 Gate D: NAND(CTL,  DO0) ──► /SET_IN  ──► U9 FF1 /S
  Altair DO1 ──► U15 Bank A ──► /D1 ──► U8 Gate C: NAND(CTL, /D1) ──► /CLR_OUT ──► U9 FF2 /R
  Altair DO1 ──────────────────────► U8 Gate B: NAND(CTL,  DO1) ──► /SET_OUT ──► U9 FF2 /S
```

1. CPU places the control word on the Altair **DO** bus and drives write strobes.
2. U6 decodes A0=0 with write → **/CTL** active (O4).
3. U15 Bank A, buffer 3: inverts /CTL → **CTL** (active high) → U2 pin 34 (CS).
4. U2 CS high → UART latches the format control bits from SW1 pins (NDB1, NDB2, NSB, NPB, POE).

**Simultaneously**, DO0 and DO1 gate U8 NAND outputs to drive /S and /R on U9:

| DO bus | Signal | NAND Gate | Output |
|--------|--------|-----------|--------|
| D0 = 1 | CTL AND DO0 | U8 Gate D | /SET_IN low → U9 FF1 /S |
| D0 = 0 | CTL AND /D0 | U8 Gate A | /CLR_IN low → U9 FF1 /R |
| D1 = 1 | CTL AND DO1 | U8 Gate B | /SET_OUT low → U9 FF2 /S |
| D1 = 0 | CTL AND /D1 | U8 Gate C | /CLR_OUT low → U9 FF2 /R |

**Interrupt enable/disable (DO0 and DO1):**

| DO0 | DO1 | Input (RX) Interrupt | Output (TX) Interrupt |
|:---:|:---:|----------------------|-----------------------|
|  0  |  0  | Disabled             | Disabled              |
|  1  |  0  | Enabled              | Disabled              |
|  0  |  1  | Disabled             | Enabled               |
|  1  |  1  | Enabled              | Enabled               |

To enable the input interrupt and disable the output interrupt, write 0x01 (D0=1, D1=0) to the control port. Bits D2–D7 are don't-care for interrupt control.

---

### Data Read

A **data read** occurs when the CPU executes an IN instruction to the data port (A0=1), after confirming bit 0 = 0 in the status byte.

```
Signal path:

  A1–A7 ──► U11, U12 (74LS85) ──► BDSEL ──┐
  SW3   ──► (comparators)                   │
                                             ▼
  A0=1   ─────────────────────────────── U6 (74LS138) ──► /RDE
  /IO_RD ────────────────────────────────────────────          │
                                                               ├──► U2 pin 4 (/RDE = 0):
                                                               │       RBR1–RBR8 ──► bDI0–bDI7
                                                               │
                                                               └──► U3 Gate D: RDE = NOT(/RDE) = 1
                                                                      U2 pin 16 (/SWE = 1) ── all status high-Z:
                                                                        DR, TBMT, PE, FE, OVR all high-Z
                                                                        U15 Bank B disabled (/OEb = 1)

  /RDE=0 ────────────────────────────── U7 Gate A ──► /DIEN=0 ──► U13 ──► Altair DI0–DI7
```

1. CPU asserts the data port address on A0–A7 (A0=1) and drives read strobes.
2. U11, U12 compare and assert BDSEL; U6 decodes A0=1 with read → **/RDE** active (O3).
3. /RDE asserts U2 pin 4 (/RDE) → U2 enables RBR1–RBR8 outputs onto bDI0–7.
4. U3 Gate D: RDE = NOT(/RDE) = 1 → U2 pin 16 (/SWE) = 1 → all U2 status outputs (DR, TBMT, PE, FE, OVR) go high-Z.
5. U15 Bank B is NOT enabled (/OEb = /SWE = 1).
6. U7 Gate A: /DIEN goes LOW → U13 enabled → drives bDI0–7 (= RBR1–8) onto Altair DI bus.
7. CPU latches the received character.
8. U2's DR flag is cleared automatically when /RDAV (tied to DR) is pulled low at the end of the cycle.

> **Note:** During a data read (RDE=1, /SWE=1), all U2 status outputs are high-Z. bDI0–7 carries RBR receive data only.

---

### Data Write (Transmit)

A **data write** occurs when the CPU executes an OUT instruction to the data port (A0=1) to transmit a character.

```
Signal path:

  A1–A7  ──► U11, U12 (74LS85) ──► BDSEL ──┐
  SW3    ──► (comparators)                   │
                                              ▼
  A0=1   ─────────────────────────────── U6 (74LS138) ──► /TDS
  /IO_WR ────────────────────────────────────────────          │
                                                               └──► U2 pin 23 (/TDS = 0): latch TBR

  Altair DO0–DO7 ──────────────────────► U14 (74LS244, always enabled) ──► bDO0–bDO7
                                                                                │
                                                                                └──► U2 TBR1–TBR8
                                                                                       │
                                                                 U2: TBR ──► TSO ──► U1 (MAX232) ──► TXD
```

1. CPU places the character on the Altair **DO** bus (DO0–DO7) and drives write strobes.
2. U11, U12 compare and assert BDSEL; U6 decodes A0=1 with write → **/TDS** active (O5).
3. U14 (permanently enabled) already mirrors DO0–DO7 onto bDO0–7, which connects to U2 TBR1–TBR8.
4. /TDS asserts U2 pin 23 (/TDS) directly → U2 latches TBR from the bDO bus.
5. U2 transfers the character to the transmit shift register and serially transmits via **TSO** at the programmed baud rate.
6. U1 (MAX232) converts TSO (TTL) to RS-232 levels on the **TXD** line.
7. When TBR is emptied, **TBMT** returns high; status bit 7 returns to 0 (TX ready).

---

### Interrupts

The SIOA board drives /pINT or a selected /VI line on the Altair bus active-low. The interrupt requests are produced by U10, which qualifies the U9 interrupt-enable latches with the live UART status flags. Each request is then routed to a target bus line through one of three jumper headers (JP3, JP4, JP5).

#### Signal Path

```
  U9 FF1 Q (RX enable) ──► U10 Gate A: NAND(DR, Q1)   ──► /IN_INT  ──► JP3 (IN)
  U9 FF2 Q (TX enable) ──► U10 Gate B: NAND(TBMT, Q2) ──► /OUT_INT ──► JP5 (OUT)

  /IN_INT, /OUT_INT ──► U10 Gates C, D ──► /BH_INT (= /IN_INT AND /OUT_INT) ──► JP4 (BOTH)

  JP3 / JP4 / JP5 ──► selected /VI0–/VI7 or /pINT line ──► Altair bus
```

- **/IN_INT** asserts (low) when a received character is ready (DR=1) and the RX interrupt is enabled.
- **/OUT_INT** asserts (low) when the transmitter is ready (TBMT=1) and the TX interrupt is enabled.
- **/BH_INT** is the combined request — it asserts when either /IN_INT or /OUT_INT is active.

#### Jumper Selection

Each jumper is a 2×9 header. One column carries the interrupt-request signal; the other column exposes the nine possible targets — /VI0 through /VI7 and /pINT (pins 17–18). A shunt connects the request to exactly one target.

| Jumper | Request | Altair Line Driven |
|--------|---------|--------------------|
| JP3 (IN)   | /IN_INT  | one of /VI0–/VI7 or /pINT (RX source) |
| JP4 (BOTH) | /BH_INT  | one of /VI0–/VI7 or /pINT (combined source) |
| JP5 (OUT)  | /OUT_INT | one of /VI0–/VI7 or /pINT (TX source) |

#### Priority Assignment

The /VI lines connect to the 88-VI Vectored Interrupt Board. Priority levels 0–7 correspond to /VI0–/VI7, with **0 being the lowest priority and 7 the highest**.

The three jumpers allow two interrupt priority configurations:

- **Separate priorities** — Install a shunt on JP3 (IN) and a separate shunt on JP5 (OUT). The receive and transmit interrupts are routed to different /VI lines, giving each a distinct priority level on the 88-VI interrupt controller.
- **Single shared priority** — Install a single shunt on JP4 (BOTH). The receive and transmit interrupts are combined onto one /VI line and share a single priority level.

#### Non-Vectored Interrupt (/pINT)

If the 88-VI vectored interrupt board is not present, place the shunt on the **/pINT** position (pins 17–18) of the desired jumper — JP3 (IN), JP4 (BOTH), or JP5 (OUT) — to drive the processor's interrupt line (/pINT, Altair pin 73) directly. This generates a single non-vectored interrupt to the CPU.

When using /PINT, the processor jumps immediately to octal location 70 (0x38) on interrupt. Place the interrupt service routine in locations **70–77 octal (0x38–0x3F)**.

#### Power-On Clear (POC)

U16 Gate C: NAND(/POC, /POC) = NOT(/POC) → **POC** (active high) drives U2 pin 21 (MR, Master Reset) high briefly at power-up to:

- Clear all UART status flags
- Reset the receiver and transmitter
- Ensure a known state before software initialization

The /POC signal is the Altair bus Power-On Clear line (bus pin 99), driven low briefly at power-up by the host system (front panel / power-on reset circuitry). The board receives it on pin 99 and uses it to reset the UART; it is not generated on the card.

---

## Connector Summary

| Ref | Description | Connector Type | Notes |
|-----|-------------|---------------|-------|
| J1  | TTL Header  | 1×6 pin header | TTL-level TX (TSO, pin 5) and RX (RSI, pin 4), no level shifting; pins 1–2 GND. Pinout matches the standard FTDI USB-to-serial 6-pin TTL header, so an FTDI USB-serial cable/adapter can connect directly. |
| J2  | DE9 DCE     | 2×5 IDC header | 9-pin DCE RS-232 (DTE device connects here) |
| J3  | DB25 DCE    | 2×7 IDC header | 25-pin DCE RS-232 (DTE device connects here) |
| J4  | DE9 DTE     | 2×5 IDC header | 9-pin DTE RS-232 (DCE device connects here) |
| J5  | DB25 DTE    | 2×7 IDC header | 25-pin DTE RS-232 (DCE device connects here) |

### RS-232 Connector Wiring

The four RS-232 ports are 2-row IDC pin headers, not D-sub connectors. The schematic annotation **CROSSOVER / DTK / INTEL / SUPERMICRO** does **not** describe a null-modem cable — it names the **ribbon-cable pin-mapping convention** used between a 2-row IDC header and a D-sub connector. The board's header pins are arranged so that a flat ribbon cable terminated with an **IDC-to-D-sub crimp connector** produces a correct, standard RS-232 D-sub pinout. This is what lets the board be cabled out with ordinary crimp-on connectors instead of soldered D-subs.

In this convention the ribbon conductors alternate between the two rows of D-sub pins: odd IDC pins map to the low-numbered D-sub pins, even IDC pins to the high-numbered ones.

**Crimp pin mapping (IDC header → D-sub):**

| IDC pin | DE9 pin (2×5) | DB25 pin (2×7) |
|:------:|:-------------:|:--------------:|
| 1  | 1 | 1  |
| 2  | 6 | 14 |
| 3  | 2 | 2  |
| 4  | 7 | 15 |
| 5  | 3 | 3  |
| 6  | 8 | 16 |
| 7  | 4 | 4  |
| 8  | 9 | 17 |
| 9  | 5 | 5  |
| 10 | — (key) | 18 |
| 11 | — | 6  |
| 12 | — | 19 |
| 13 | — | 7  |
| 14 | — | 20 |

**Connector gender:** the **DTE** headers (J4, J5) terminate to a **male** D-sub; the **DCE** headers (J2, J3) terminate to a **female** D-sub. Only TXD, RXD, and signal ground are wired — no hardware-handshake lines are connected. Signal direction is from the board's point of view: **TXD = board transmit (output)**, **RXD = board receive (input)**.

Note that DE9 and DB25 historically swap the TXD/RXD pin assignments (on a DTE, TXD is DE9 pin 3 but DB25 pin 2), and DTE vs. DCE swaps them again. The header wiring below already accounts for both, so every resulting D-sub is a standard, straight-through-cable-compatible pinout.

#### DCE headers — female D-sub (J2, J3)

A DTE device (terminal/PC) connects here with a straight-through cable.

**J2 — DE9 DCE (female):**

| IDC pin | Signal | DE9 pin |
|:------:|:------:|:-------:|
| 3 | TXD (board out) | 2 |
| 5 | RXD (board in)  | 3 |
| 9 | GND             | 5 |

**J3 — DB25 DCE (female):**

| IDC pin | Signal | DB25 pin |
|:------:|:------:|:--------:|
| 3  | RXD (board in)  | 2 |
| 5  | TXD (board out) | 3 |
| 13 | GND             | 7 |

#### DTE headers — male D-sub (J4, J5)

A DCE device (modem) connects here with a straight-through cable.

**J4 — DE9 DTE (male):**

| IDC pin | Signal | DE9 pin |
|:------:|:------:|:-------:|
| 3 | RXD (board in)  | 2 |
| 5 | TXD (board out) | 3 |
| 9 | GND             | 5 |

**J5 — DB25 DTE (male):**

| IDC pin | Signal | DB25 pin |
|:------:|:------:|:--------:|
| 3  | TXD (board out) | 2 |
| 5  | RXD (board in)  | 3 |
| 13 | GND             | 7 |

---

## Jumper Summary

| Ref | Label | Type | Function |
|-----|-------|------|----------|
| JP1 | 5V    | 1×2 header | Routes +5V onto the DE9 DCE connector (J2) for powering an external device. Leave open if not needed. |
| JP2 | -12V  | 1×2 header | Connects the Altair −16V rail to the on-board −12V regulator. **Install when using a COM2502** (which needs −12V on VGG); leave open for a +5V-only IM6402. |
| JP3 | IN    | 2×9 header | Routes the RX interrupt request (/IN_INT) to one of /VI0–/VI7 or /pINT |
| JP4 | BOTH  | 2×9 header | Routes the combined interrupt request (/BH_INT) to one of /VI0–/VI7 or /pINT |
| JP5 | OUT   | 2×9 header | Routes the TX interrupt request (/OUT_INT) to one of /VI0–/VI7 or /pINT |

---

## DIP Switch Summary

| Switch | Positions | Function |
|--------|:---------:|----------|
| SW1 | 5 | UART word format: NDB1, NDB2, NSB, NPB, POE (word length, stop bits, parity) |
| SW2 | 8 | Baud rate select: 110 / 300 / 600 / 1200 / 2400 / 4800 / 9600 / 19200 |
| SW3 | 7 | I/O port base address: bits A1–A7 (switch 1=A1, switch 7=A7); A0 selects status vs. data port |

---

## Key Internal Signal Reference

| Signal | Source | Destination | Description |
|--------|--------|-------------|-------------|
| CLK | SW2 output | U2 RC, U2 TC | Selected baud rate clock (16× baud) |
| RSI | U1 R1OUT | U2 pin 20 | TTL receive data from MAX232 |
| TSO | U2 pin 25 | U1 T1IN | TTL transmit data to MAX232 |
| TXD | U1 T1OUT | J2, J3, J4, J5 | RS-232 transmit line |
| RXD | J2, J3, J4, J5 | U1 R1IN | RS-232 receive line |
| TBMT | U2 pin 22 | U15 Bank B (pin 12), U10 Gate B (pin 4) | TX buffer empty flag |
| DR | U2 pin 19 | U15 Bank B (pin 14), U10 Gate A (pin 1) | Received data ready flag |
| /SWE | U6 O2 (pin 13) | U15 OEb, U7 Gate A | Decoded status read strobe (active low) |
| /RDE | U6 O3 (pin 12) | U2 pin 4, U7 Gate A, U3 Gate D | Decoded data read strobe (active low) |
| RDE | U3 Gate D | U2 pin 16 (/SWE) | Active-high data read signal = NOT(/RDE) |
| /CTL | U6 O4 (pin 11) | U15 Bank A I3 | Decoded control write strobe (active low) |
| /TDS | U6 O5 (pin 10) | U2 pin 23 | Decoded data write strobe (active low) |
| CTL | U15 Bank A O3 (pin 7) | U2 pin 34 (CS), U8 | Active-high control strobe |
| /pWR | CPU (bus pin 77) | U16 Gate A inputs (pins 1, 2) | Processor write strobe (active low); CPU control signal |
| pWR | U16 Gate A (pin 3) | U16 Gate B input (pin 4) | Intermediate active-high write signal = NOT(/pWR) |
| sOUT | CPU (bus pin 45) | U16 Gate B input (pin 5) | Processor output-cycle status (active high); CPU status signal |
| /IO_WR | U16 Gate B (pin 6) | U6 pin 2 | I/O write strobe: low when pWR and sOUT both active |
| pDBIN | CPU (bus pin 78) | U16 Gate D input (pin 12) | Processor data-bus-in strobe (active high); CPU control signal |
| sINP | CPU (bus pin 46) | U16 Gate D input (pin 13) | Processor input-cycle status (active high); CPU status signal |
| /IO_RD | U16 Gate D (pin 11) | U6 pin 3 | I/O read strobe: low when pDBIN and sINP both active |
| /POC | Altair bus (pin 99) | U16 Gate C inputs (pins 9, 10) | Bus power-on-clear (active low); system reset at power-up |
| /D0 | U15 Bank A O1 (pin 3) | U8 Gates A, D | NOT(DO0) |
| /D1 | U15 Bank A O2 (pin 5) | U8 Gates B, C | NOT(DO1) |
| /DIEN | U7 Gate A (pin 3) | U13 OEa, OEb | DI bus enable: low during any read |
| BDSEL | U12 Oa=b (pin 6) | U6 E3 (pin 6) | Board address match (active high) |
| bDI0–7 | U2 RBR, status, U15 Bank B | U13 inputs | Board-side DI bus |
| bDO0–7 | U14 outputs | U2 TBR1–TBR8 | Board-side DO bus |
| B110–B19200 | U4, U5 outputs | SW2 inputs | Available baud rate clocks |
| POC | U16 Gate C output (pin 8) | U2 pin 21 (MR) | Active-high master reset; NOT(/POC) |
| Q1 | U9 FF1 Q (pin 5) | U10 Gate A (pin 2) | RX interrupt enable latch |
| Q2 | U9 FF2 Q (pin 9) | U10 Gate B (pin 5) | TX interrupt enable latch |
| /IN_INT | U10 Gate A (pin 3) | JP3, U10 Gate C | RX interrupt request: low when DR and Q1 both high |
| /OUT_INT | U10 Gate B (pin 6) | JP5, U10 Gate C | TX interrupt request: low when TBMT and Q2 both high |
| /BH_INT | U10 Gate D (pin 11) | JP4 | Combined interrupt request: low when either /IN_INT or /OUT_INT active |
| /VI0–/VI7 | JP3, JP4, JP5 | Altair bus (pins 4–11) | Vectored interrupt outputs (active low) |
| /pINT | JP3, JP4, JP5 | CPU (Altair bus pin 73) | Processor interrupt request (active low) |
| −16V | Altair bus (pin 52) | JP2 → R1 | Unregulated negative rail for −12V generation |
| −12V | D1/R1/C24 node | U2 pin 2 (VGG) | On-board −12V supply for UART (12V zener regulated) |
| R1 | JP2-B node | −12V node | 130Ω series resistor in the −12V zener regulator |
| D1 | −12V node | GND | 12V zener; clamps the −12V rail |

---

## Bill of Materials

| Ref | Qty | Value | Footprint |
|-----|:---:|-------|-----------|
| C1, C2, C3, C5, C6 | 5 | 1uF | Disc capacitor, D4.3mm, P5.00mm |
| C4, C7, C8, C9, C10, C11, C12, C13, C14, C15, C16, C17, C18, C19, C20, C22 | 16 | 0.1uF | Disc capacitor, D4.3mm, P5.00mm |
| C21, C23, C24 | 3 | 47uF | Tantalum radial, D6.0mm, P2.50mm |
| D1 | 1 | 12V | Axial diode, DO-15, P12.70mm |
| HS1 | 1 | Heatsink | Aavid TV5G, TO-220 horizontal |
| J1 | 1 | TTL HEADER | Pin header 1×6, P2.54mm vertical |
| J2 | 1 | DE9 DCE HEADER | IDC header 2×5, P2.54mm vertical |
| J3 | 1 | DB25 DCE HEADER | IDC header 2×7, P2.54mm vertical |
| J4 | 1 | DE9 DTE HEADER | IDC header 2×5, P2.54mm vertical |
| J5 | 1 | DB25 DTE HEADER | IDC header 2×7, P2.54mm vertical |
| JP1 | 1 | 5V | Pin header 1×2, P2.54mm vertical |
| JP2 | 1 | -12V | Pin header 1×2, P2.54mm vertical |
| JP3 | 1 | IN | Pin header 2×9, P2.54mm vertical |
| JP4 | 1 | BOTH | Pin header 2×9, P2.54mm vertical |
| JP5 | 1 | OUT | Pin header 2×9, P2.54mm vertical |
| R1 | 1 | 130 | Axial resistor, L9.9mm D3.6mm, P12.70mm |
| RN1 | 1 | 4.7K | Resistor array SIP-6 |
| RN2 | 1 | 2.2K | Resistor array SIP-8 |
| SW1 | 1 | SW_DIP_x05 | DIP switch 5-position, 9.78×14.88mm, P2.54mm |
| SW2 | 1 | SW_DIP_x08 | DIP switch 8-position, 9.78×22.5mm, P2.54mm |
| SW3 | 1 | SW_DIP_x07 | DIP switch 7-position, 9.78×19.96mm, P2.54mm |
| U1 | 1 | MAX232 | DIP-16 socket, W7.62mm |
| U2 | 1 | COM2502 | DIP-40 socket, W15.24mm |
| U3, U8, U10, U16 | 4 | 74LS00 | DIP-14 socket, W7.62mm |
| U4 | 1 | 74LS161 | DIP-16 socket, W7.62mm |
| U5 | 1 | 74LS393 | DIP-14 socket, W7.62mm |
| U6 | 1 | 74LS138 | DIP-16 socket, W7.62mm |
| U7 | 1 | 74LS08 | DIP-14 socket, W7.62mm |
| U9 | 1 | 74LS74 | DIP-14 socket, W7.62mm |
| U11, U12 | 2 | 74LS85 | DIP-16 socket, W7.62mm |
| U13, U14 | 2 | 74LS244 | DIP-20 socket, W7.62mm |
| U15 | 1 | 74LS368 | DIP-16 socket, W7.62mm |
| VR1 | 1 | LM7805 | TO-220-3 horizontal, tab down |
| X1 | 1 | 1.2288 MHz | Oscillator DIP-8 |
