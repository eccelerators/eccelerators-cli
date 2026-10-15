# Changelog

All notable changes to this package are documented here.

## 1.2.0

- Depend on `Livt.IO 1.2.0`.
- Migrate parser and output buffers to `DistributedRam<byte, 64>` and compact
  terminal storage to `AsynchronousDistributedRam8x64` with explicit port wiring.
- Preserve write-before-read semantics without requiring zero-filled memory.
- Verify buffered UART byte handoff, transmit backpressure and loopback input
  through the current Try-operation API.
- Register the compact echo regression and update transport integration examples.

## 1.1.0 - 2026-08-30

- Add `CompactCli`, a free-text terminal with a distributed 64-byte line buffer,
  optional echo and backpressured streaming output for constrained FPGA applications.
- Add opt-in free-text lines and complete raw-line byte access while preserving
  strict excessive-argument rejection by default.

## 1.0.0 - 2026-08-26

- Add a transport-independent, bounded FPGA command-line interface.
- Support CR, LF, CRLF, backspace, delete, optional echo, and prompts.
- Parse one command and up to eight whitespace-separated arguments.
- Provide atomic output writes and explicit transport backpressure.
- Use a 64-byte output FIFO suitable for resource-constrained FPGAs.
- Store the command line only once and use byte-wide bounded counters and token
  spans to reduce synthesized state and arithmetic width.
- Store the parser line and output FIFO in separate `Livt.IO.Ram` components to
  reduce LUT and flip-flop usage.
- Cover disabled echo, input backpressure, failed atomic line writes, and full
  FIFO wraparound ordering in the test suite.
