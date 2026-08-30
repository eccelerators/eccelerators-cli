# Eccelerators CLI

`Eccelerators.Cli` is a transport-independent command-line interface for FPGA
applications written in Livt. It provides bounded terminal input, editing,
parsing, prompts, atomic output writes, and explicit backpressure. Applications
remain responsible for the physical byte transport and static command dispatch.

The first stable release is `1.0.0`.

## Features

- 64-byte bounded command lines
- up to eight whitespace-separated arguments
- CR, LF, and CRLF line endings
- backspace and delete editing
- optional local echo
- scheduled `> ` prompts
- 64-byte circular output FIFO
- atomic byte-array and CRLF line writes
- lossless input and output backpressure
- no dynamic allocation or command registration
- optional free-text lines with raw byte access

All capacities and commands are fixed at synthesis time.

For resource-constrained applications, `CompactCli` provides the same core
terminal contract with one 64-byte distributed-RAM line buffer and a
single-byte backpressured output slot. It deliberately omits tokenized
arguments, local echo, prompts, and the output FIFO; applications stream their
own text and perform static exact-command dispatch.

## Installation

Add the package to a Livt project with:

```toml
[dependencies]
"Eccelerators.Cli" = "1.0.0"
```

Import its public API with:

```livt
using Eccelerators.Cli
```

## Components

- `Cli` is the application-facing facade.
- `CliLineEditor` tracks bounded input length and line-ending state.
- `CliParser` stores a command snapshot and token spans.
- `CliOutput` queues echo, responses, errors, and prompts atomically.
- `CompactCli` provides bounded free-text input and streaming output with a
  smaller hardware footprint.

See [DESIGN.md](DESIGN.md) for ownership, data flow, and timing contracts.

## Application integration

An application owns one `Cli` and one byte transport. Its continuous process
services prompts, retries pending commands, drains output only after the
transport accepts a byte, and accepts input only when the CLI has capacity. The
transport operations below are pseudocode and must be replaced by the target
application's UART or other byte transport:

```livt
process Main()
{
	this.cli.Service()

	if (this.cli.HasCommand() == true) {
		var completed: bool = this.DispatchCommand()
		if (completed == true) { this.cli.CompleteCommand() }
	}

	if (this.cli.HasOutput() == true && transportCanWrite == true) {
		var value: byte = this.cli.PeekOutput()
		if (transportWrite(value) == true) { this.cli.ConsumeOutput() }
	}

	if (transportHasData == true && this.cli.CanAcceptByte() == true) {
		this.cli.AcceptByte(transportRead())
	}
}
```

The transport must not discard an input byte when `CanAcceptByte()` is false.
Likewise, call `ConsumeOutput()` only after the transport accepted the byte from
`PeekOutput()`. These two rules preserve data during backpressure.

The sibling `livt-uart-cli-app` repository provides a complete UART example.

### Compact integration

`CompactCli` is suited to applications such as the TinyStories showcase where
every non-command line is application data. A transport process accepts bytes
when `CanAcceptByte()` is true, waits for `HasLine()`, and calls `CompleteLine()`
after dispatch. CR, LF, CRLF, backspace/delete, printable input, and overflow
recovery are handled internally. `EnableEcho()` adds backpressured printable,
line-ending, and editing echo; `DisableEcho()` restores silent input.

Exact command matching uses a fixed-width argument so synthesis does not need
dynamic command storage. The expected command is left-aligned and padded to
16 bytes:

```livt
var helpCommand: byte[16] = ['/' as byte, 'h' as byte, 'e' as byte, 'l' as byte,
  'p' as byte, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0]
if (this.cli.CommandEquals(helpCommand, 5)) {
  // Stream the help response.
}
```

Output is explicitly backpressured. `WriteByte(value)` waits until its one-byte
slot is free; the transport reads `PeekOutput()` and calls `ConsumeOutput()`
only after the UART or other sink accepts that byte.

## Adding commands

Commands are statically dispatched by the application that owns `Cli`:

```livt
var status = "status".Encode()
var isStatus: bool = this.cli.CommandEquals(status)
if (isStatus == true) {
	var message = "OK".Encode()
	var written: bool = this.cli.TryWriteLine(message)
	return written
}
```

Returning `false` keeps the command pending so the application can retry after
the output transport drains.

## Public API

Input and lifecycle:

- `Service()`
- `CanAcceptByte()` and `AcceptByte(value)`
- `HasCommand()` and `CompleteCommand()`
- `EnableFreeText()`, `DisableFreeText()`, `GetLineLength()`, and `GetLineByte(index)`
- `Reset()`

Compact input and lifecycle:

- `CanAcceptByte()` and `AcceptByte(value)`
- `HasLine()`, `GetLineLength()`, `GetLineByte(index)`, and `CompleteLine()`
- `EnableEcho()` and `DisableEcho()`
- `HasOverflow()` and `Reset()`
- `CommandEquals(expected, expectedLength)`
- `WriteByte(value)`, `HasOutput()`, `PeekOutput()`, and `ConsumeOutput()`

Parsing:

- `CommandEquals(expected)` and `GetCommandLength()`
- `GetArgumentCount()`
- `GetArgumentLength(index)` and `GetArgumentByte(index, offset)`
- `ArgumentEquals(index, expected)`

Output:

- `CanWrite(length)`
- `TryWriteByte(value)`, `TryWrite(data)`, and `TryWriteLine(data)`
- `TryWriteArgumentsLine()`
- `HasOutput()`, `PeekOutput()`, and `ConsumeOutput()`
- `GetOutputCount()`

## Limits

- Command lines contain at most 64 printable bytes.
- At most eight arguments are retained; excess arguments reject the line.
- Free-text mode retains the complete 64-byte line and allows excess argument
  words to reach the application; only the first eight remain available through
  the argument API.
- Parsing is case-sensitive.
- Spaces and tabs separate arguments; quoting and escaping are not implemented.
- The output FIFO holds 64 bytes, so one atomic `TryWriteLine()` payload can be
  at most 62 bytes when the FIFO is empty.
- Applications must drain output while receiving echoed input.
- Dynamic command registration is intentionally outside the hardware model.

## Build and test

```bash
livt validate
livt test
```

The test suite covers complete FIFO ordering and wraparound, atomic writes, line
endings, editing, overflow recovery, parsing, argument limits, prompts, enabled
and disabled echo, input backpressure, command completion, compact exact-command
matching, and compact streaming-output backpressure.

## License

MIT. See [LICENSE](LICENSE).
