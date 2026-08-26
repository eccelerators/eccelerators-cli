# Eccelerators.Cli design

## Ownership

One application component owns exactly one `Cli`. `Cli` owns its editor, parser,
and output queue. Application command handlers call the owning `Cli` directly;
the instance is not stored by multiple command components.

This ownership model maps cleanly to generated hardware and avoids multiple
access paths to the same stateful component.

## Input path

`CanAcceptByte()` reserves enough output capacity for the worst immediate
terminal action and refuses new bytes while a command is pending. `AcceptByte()`
updates the editor state and the parser's single line snapshot, then queues echo
bytes.

Line behavior:

- `CR` completes a line and suppresses one following `LF`.
- `LF` completes a line.
- backspace and delete remove one stored character.
- unsupported control bytes are consumed and ignored.
- bytes beyond 64 characters set overflow and are not stored or echoed.

An overflowed line produces `Error: line too long`, resets the line, and
schedules another prompt.

Input with more than eight arguments produces `Error: too many arguments` and
is rejected before application dispatch, so an application never executes a
silently truncated command.

## Parsing

`CliParser` owns a byte snapshot so parsing does not repeatedly traverse a
cross-component reference. It records the start and length of the command plus
eight arguments. The parser does not copy individual tokens or allocate memory.
The line editor tracks only length and editing state, avoiding a second 64-byte
line buffer. Lengths, token spans, and FIFO indices use byte-wide storage because
all capacities are at most 64 bytes.

## Output and backpressure

`CliOutput` is a 64-byte circular FIFO. Array and line writes first verify the
complete required capacity, so failure leaves the queue unchanged.

The parser line and output FIFO are stored in separate `Livt.IO.Ram` components.
Each logical buffer uses only its first 64 byte addresses, trading two block RAM
instances for substantially less LUT and flip-flop state than compiler-generated
fixed-array mutation shadows. They remain separate because a pending command must
stay readable while its response is queued.

The application must only call `ConsumeOutput()` after its transport accepts the
byte returned by `PeekOutput()`. This preserves every echo, response, error, and
prompt during downstream backpressure.

## Reference resource usage

An out-of-context synthesis of the complete public `Cli` component with Vivado
2026.1 for `xc7a35tcpg236-1` uses 5,335 logic LUTs, 5,774 flip-flops, and two
RAMB18 blocks. The equivalent fixed-array implementation used 9,321 logic LUTs
and 10,044 flip-flops without block RAM. The RAM-backed buffers therefore reduce
standalone CLI logic by 3,986 LUTs and 4,270 flip-flops. Application-level
utilization varies with the methods used and the surrounding synthesis context.

`TryWriteArgumentsLine()` provides the common command-response operation of
joining parsed arguments with spaces and CRLF. It reserves space for the entire
line before writing, preserving the same atomic response contract.

## Command completion

The application keeps a parsed command pending until its handler returns true.
After successful output enqueueing, `CompleteCommand()` clears input/parser
state and schedules the next prompt. This permits retrying a response when the
output queue lacks space.
