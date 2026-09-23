---
description: "Exodus MCP extension setup: installing ExodusMCP.dll, port 8600, Streamable HTTP transport, address formats, Mega Drive memory map and CPUs, tool categories, connection troubleshooting. Use when setting up or troubleshooting the Exodus MCP connection."
---

# Exodus MCP Server Setup Guide

## Key Configuration Points

Exodus uses an MCP extension plugin (ExodusMCP.dll) that starts an HTTP server automatically when loaded. Place ExodusMCP.dll in the Exodus Assemblies folder (or wherever AssembliesPath points in settings.xml).

The default port is 8600, configurable via the extension's settings in the Exodus module XML.

## Communication Protocol

The server operates via Streamable HTTP at the endpoint `POST http://localhost:8600/mcp` using JSON-RPC 2.0 format. The server is stateless (no sessions, JSON responses only) and implements MCP `2026-07-28`, while still accepting `initialize`-handshake clients on `2025-11-25`, `2025-06-18`, `2025-03-26` and `2024-11-05`. `GET`/`DELETE` on `/mcp` return 405.

Every tool carries annotations (`readOnlyHint`, `destructiveHint`, `idempotentHint`): only `run_system`, `stop_system`, `write_memory`, the breakpoint/watchpoint set/remove tools, `step_device` and `query_pixel` are marked non-read-only.

## Server Information

- **Server name:** exodus-mcp
- **Capabilities:** tools (26 tools)

## Address Format

All tools accept addresses as hex strings or integers:
- Motorola: `"$FF0000"` (preferred for M68000)
- C-style: `"0xFF0000"`
- Zilog: `"FF0000h"` (for Z80)
- Integer: `16711680` (a bare number without `$`, `0x` or `h` is always decimal)

All addresses returned by tools use the Motorola `$XXXXXX` convention.

## Mega Drive/Genesis Memory Map

| Region | Address Range | Size |
|--------|--------------|------|
| ROM cartridge | $000000 - $3FFFFF | Up to 4 MB |
| Z80 RAM | $A00000 - $A01FFF | 8 KB |
| I/O area | $A10000 - $A1001F | Controller/misc |
| VDP ports | $C00000 - $C00008 | VDP data/control |
| 68K RAM | $FF0000 - $FFFFFF | 64 KB |

## Mega Drive CPUs

### Motorola 68000 (Main CPU)
- 16/32-bit, big-endian
- Clock: 7.67 MHz (NTSC) / 7.60 MHz (PAL)
- 8 data registers (D0-D7), 8 address registers (A0-A7/SP)
- 24-bit address bus (16 MB address space)

### Zilog Z80 (Sound CPU)
- 8-bit, little-endian
- Clock: 3.58 MHz (NTSC) / 3.55 MHz (PAL)
- 8 KB dedicated RAM at $A00000 (68K view)
- Controls the YM2612 and SN76489 sound chips

## Tool Categories

| Category | Tools | Count |
|----------|-------|-------|
| System control | get_system_status, run_system, stop_system | 3 |
| Device listing | list_devices | 1 |
| CPU registers | read_cpu_registers | 1 |
| Memory access | read_memory, write_memory | 2 |
| Memory search | search_memory | 1 |
| Disassembly | disassemble | 1 |
| Breakpoints | set_breakpoint, remove_breakpoint, list_breakpoints | 3 |
| Watchpoints | set_watchpoint, remove_watchpoint, list_watchpoints | 3 |
| Execution | step_device | 1 |
| VDP raw | read_vram, read_cram, read_vsram, read_vdp_registers | 4 |
| VDP decoded | read_sprite_table, read_palette, read_nametable, read_vdp_state | 4 |
| Screenshot & pixel | screenshot, query_pixel | 2 |
| **Total** | | **26** |

## Common Issues

- Ensure ExodusMCP.dll is in the Assemblies folder and a system module is loaded
- The server only binds to localhost (127.0.0.1) -- remote connections are not supported, and requests with a non-local `Origin` header are rejected with 403
- Most tools need a system module to be loaded -- load a Mega Drive module first
- Memory read max size is 4096 bytes per call (larger `length` values are clamped). Use `search_memory` for finding patterns across larger ranges
- Device names must match the instance names shown by `list_devices`
- CPU tools (registers, memory, disassembly, breakpoints, step) require a processor device name
