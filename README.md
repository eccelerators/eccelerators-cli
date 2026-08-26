# Eccelerators CLI

`Eccelerators.Cli` is a transport-independent command-line interface for FPGA
applications written in Livt. It provides bounded terminal input, editing,
parsing, prompts, atomic output writes, and explicit backpressure. Applications
remain responsible for the physical byte transport and static command dispatch.

The package is preparing for its first `0.1.0` release and is not published yet.

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

All capacities and commands are fixed at synthesis time.

## Installation

After the first release, add the package to a Livt project with:

```toml
[dependencies]
"Eccelerators.Cli" = "0.1.0"
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
- `Reset()`

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
and disabled echo, input backpressure, and command completion.

## License

MIT. See [LICENSE](LICENSE).
