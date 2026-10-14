# rea-triage

Turn a Linux core dump into a bug report you can paste into an issue — without
running the program.

```bash
rea-triage ./core ./parser > crash.md
```

```markdown
# Crash: NULL pointer dereference in `sum+0x31`

`parser` crashed with **SIGSEGV**: null pointer dereference at `sum+0x31`. …

| Field         | Value                      |
| ------------- | -------------------------- |
| Signal        | SIGSEGV (code 1)           |
| Fault address | `0x0`                      |
| Location      | `sum+0x31 (parser+0x11da)` |
| Signature     | `7a6c9d31052c`             |

## Backtrace

#0 0x556fdda1b1da sum+0x31 [parser]
#1 0x556fdda1b26a main+0x75 [parser]

## Instructions at the fault

=> 0x556fdda1b1da <sum+49>: mov (%rax),%eax

## Registers used by the faulting instruction

- `rax` = `0x0` ← NULL

## Suggested next steps

- [ ] Find where the pointer used by the faulting instruction is assigned …
```

It is a thin layer over [REA](https://github.com/morluto/rea) for developers
debugging **their own** programs. It uses only REA's public CLI, so it can
live and ship separately from REA.

## What it does

| Step | Source                       | Fact                                                                      |
| ---- | ---------------------------- | ------------------------------------------------------------------------- |
| 1    | `rea inspect-recorded-crash` | signal, fault address, registers, mapped files (NT_FILE)                  |
| 2    | `rea inspect-binary-layout`  | function symbols, load segments, build hardening                          |
| 3    | rea-triage                   | load bias from mappings + segments → `function+offset` for any address    |
| 4    | `gdb` (optional)             | backtrace, inlined frames, source lines, disassembly                      |
| 5    | rea-triage                   | classification, relevant registers, ASLR-stable signature, Markdown issue |

Crash classes: NULL / near-NULL dereference, stack overflow, invalid access,
write to read-only memory, jump to non-executable memory, general protection
fault, assertion failure, uncaught C++ exception, heap corruption abort,
divide by zero, illegal instruction, bus error, trap.

When the crash is inside a library (e.g. `strlen(NULL)` in libc), the title
points at the first frame in **your** binary and the summary names the library
function it called.

The **signature** hashes the crash class with the top three frame names, never
addresses, so the same bug groups together across runs and machines.

## Requirements

- Linux x86-64 cores (REA's core reader is Linux amd64 only today).
- REA ≥ 6.4 (`npm install -g rea-agents`, or point `REA_TRIAGE_REA` at a
  `scripts/rea.mjs` checkout).
- `REA_PWNTOOLS_PYTHON`: a Python with pwntools 4.15.0, pyelftools 0.33 and
  unicorn 2.1.2 (REA never installs these for you):
  ```bash
  python3 -m venv ~/.rea-pwn && ~/.rea-pwn/bin/pip install pwntools==4.15.0 pyelftools==0.33 unicorn==2.1.2
  export REA_PWNTOOLS_PYTHON=~/.rea-pwn/bin/python
  ```
- Optional: `gdb` for backtraces and disassembly (`--no-gdb` to skip).

## Usage

```
rea-triage <core> <binary> [--json] [-o FILE] [--no-gdb] [--gdb PATH]
```

Exit codes: `0` report produced, `1` triage failed, `2` invalid arguments.
`--json` emits the same facts for tooling (bigints as hex strings).

## Safety and limits

- **Nothing is executed.** REA reads the core file; gdb loads it with
  `core-file` semantics, script auto-loading off and debuginfod off.
- The binary is matched to the core **by file name only**. If you pass a
  different build, function names will be wrong; gdb's "core file may not
  match" warning is surfaced in the report's Warnings section.
- Without gdb there is no unwinding, so only the faulting frame is shown.
- Classification is heuristic and states its confidence.

## Design: opt-in reproduction (not implemented)

Running the target would allow "reproduce, then confirm the fix". REA treats
target execution as a scope decision, so v1 is read-only. The planned design:

- An explicit `--reproduce -- <cmd> [args]` flag; never inferred from the core.
- Only the caller's own binary, under REA's process supervision
  (`src/process/` owned process groups, deadlines, private roots), with core
  dumps enabled in a private directory.
- The new core goes through the same read-only pipeline; reports are compared
  by signature to say "reproduced" / "different crash" / "no crash".
- `--verify-fix <new-binary>` reruns the same command against a rebuilt binary
  and reports whether the signature disappears.

This needs a maintainer scope decision before implementation.

## Development

```bash
npm install
npm test        # unit tests; the e2e test runs when gcc, gdb,
                # REA_TRIAGE_REA and REA_PWNTOOLS_PYTHON are available
```
