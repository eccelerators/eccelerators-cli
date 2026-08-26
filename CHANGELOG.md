# Changelog

All notable changes to this package are documented here.

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
