# 88-SIOR2

SIOA-REV-2 (Serial I/O Adapter, Revision 2) — an Altair 8800 / S-100 bus serial I/O card.

The card provides one full-duplex RS-232 serial port for MITS Altair 8800 and compatible systems. It is 100% compatible with the original MITS 88-SIOA and 88-SIOB Rev 1 boards, so existing software and port configurations work unchanged.

## Order

Order the board on [Tindie](https://www.tindie.com/products/44097/).

## Features

- Baud rates from 110 to 19,200 (8-position DIP switch)
- Word format: 5–8 data bits, 1–2 stop bits, odd/even/no parity (5-position DIP switch)
- Configurable I/O port address (7-position DIP switch); status/control at the base address, data at base + 1
- Interrupt-driven operation with selectable vector (VI0–VI7) or non-vectored /pINT
- RS-232 DCE and DTE headers (DE9 and DB25) plus an FTDI-compatible 6-pin TTL header
- On-board +5V regulation from the Altair +8V supply, and −12V generation from the −16V supply for the UART
- Socket accepts COM2502 and other AY-5-1013-pinout UARTs, including the +5V-only IM6402

## Documentation

- [Wiki](https://github.com/deltecent/88-SIOR2/wiki) — IC descriptions, configuration, bus operation, signal reference and BOM
- [Technical Reference (Markdown)](SIOA-REV-2-Technical-Reference.md)
- [Technical Reference (PDF)](SIOA-REV-2-Technical-Reference.pdf)

## Issues

Report bugs or request improvements on the [issue tracker](https://github.com/deltecent/88-SIOR2/issues).
