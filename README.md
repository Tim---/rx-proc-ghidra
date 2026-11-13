# rx-proc-ghidra

This Ghidra module adds support for disassembling and decompiling Renesas RX architecture binaries.

This module is a work in progress, but currently implements disassembly and decompilation rules for
most of the instructions supported by the RX62T processor.

## Languages

| Language id | Data byte order | Use for |
|---|---|---|
| `rx62t:LE:32:default` | little-endian | RX in little-endian mode (the usual case) |
| `rx62t:BE:32:default` | big-endian | RX in big-endian mode, with code presented in fetch order (see below) |

Both share one instruction specification (`rx62t.sinc`) and the same compiler spec (Renesas CC-RX ABI:
arguments in R1–R4 then the stack, R15 = address for a returned structure, return value in R1,
R6–R13 preserved, caller removes stack arguments).

### Big-endian mode

When an RX runs in big-endian mode, data in memory is big-endian, but the CPU still fetches
instructions as little-endian 32-bit words. In a raw big-endian flash image the code is therefore
byte-reversed within each aligned 32-bit word. To analyse such an image:

1. Un-swap the 32-bit words of the **code** regions only (leave constant data, strings and vector
   tables untouched), so the instruction stream is in fetch order.
2. Import with `rx62t:BE:32:default`. Instructions decode exactly as in the LE language, and
   multi-byte data (constant tables, vector tables, mixed-width accesses to RAM) reads big-endian.

Which regions are code is image-specific, so it is not done by this module.

## Installation

`unzip ghidra_9.0.2_PUBLIC_20190626_RX62T.zip -d ${ghidra_install_dir}/Ghidra/Extensions`

## Building

`gradle -PGHIDRA_INSTALL_DIR=<ghidra_install_dir>`

## Testing

`test/run_tests.sh` compiles the SLEIGH specs into a private staging directory and runs every vector
in `test/vectors/*.vec` headlessly under each language id in `rx62t.ldefs`: the bytes are
disassembled, the listing text checked, the instruction single-stepped in Ghidra's emulator and
registers, flags and memory compared. It also checks the register layout and the calling
conventions. The installed extension is not touched.

```
test/run_tests.sh                       # all vectors, all languages
test/run_tests.sh test/vectors/minmax.vec
```

With `GHIDRA_INSTALL_DIR=<ghidra install>` it uses that installation's `support/sleigh` and
`support/analyzeHeadless`; otherwise it uses the Flatpak (`GHIDRA_FLATPAK=org.ghidra_sre.Ghidra`,
`GHIDRA_VERSION=12.1.3`), in which case `STAGE` (default `$HOME/rx-proc-test`) must be under `$HOME`.
Only the files the script creates inside `STAGE` are removed. The vector format is documented at the
top of `test/ghidra_scripts/RxVectorTest.java`. Semantics are checked against the RXv1 software
manual (R01US0032EJ) and encodings against binutils `opcodes/rx-decode.opc`.
