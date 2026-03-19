# Exodus Claude Plugin

Claude Code plugin for the [Exodus Emulation Platform](http://www.exodusemulator.com) MCP integration.

Provides CPU debugging, memory inspection, VDP access, disassembly, breakpoints, and system control for Mega Drive/Genesis reverse engineering.

## Installation

### From local directory

```sh
claude --plugin-dir /path/to/exodus-claude-plugin
```

## Prerequisites

- Exodus Emulation Platform with [ExodusMCP](https://github.com/sadnescity/exodus-mcp-extension) extension loaded
- ExodusMCP.dll placed in the Assemblies folder (or configured AssembliesPath)
- A Mega Drive system module loaded

## Skills

| Skill | Description |
|-------|-------------|
| `exodus-setup` | MCP server setup, memory map, tool inventory |
| `exodus-cpu-debug` | CPU debugging, memory, VDP, disassembly, breakpoints, watchpoints, screenshot, pixel query (26 tools) |
| `exodus-workflows` | Reverse engineering workflows for common debugging tasks |

## Tools (26)

| Category | Tools |
|----------|-------|
| System control | `get_system_status`, `run_system`, `stop_system` |
| Devices | `list_devices` |
| CPU registers | `read_cpu_registers` |
| Memory | `read_memory`, `write_memory` |
| Memory search | `search_memory` |
| Disassembly | `disassemble` |
| Breakpoints | `set_breakpoint`, `remove_breakpoint`, `list_breakpoints` |
| Watchpoints | `set_watchpoint`, `remove_watchpoint`, `list_watchpoints` |
| Execution | `step_device` |
| VDP raw | `read_vram`, `read_cram`, `read_vsram`, `read_vdp_registers` |
| VDP decoded | `read_sprite_table`, `read_palette`, `read_nametable`, `read_vdp_state` |
| Screenshot & pixel | `screenshot`, `query_pixel` |

## Address Format

All tools accept addresses as hex strings or integers:
- Motorola: `"$FF0000"` (preferred for M68000)
- C-style: `"0xFF0000"`
- Zilog: `"FF0000h"` (for Z80)
- Integer: `16711680`

All returned addresses use the Motorola `$XXXXXX` convention.

## Mega Drive Quick Reference

- **Main CPU:** Motorola 68000, 7.67 MHz, 16/32-bit, big-endian
- **Sound CPU:** Zilog Z80, 3.58 MHz, 8-bit
- **RAM:** 64 KB (68K) + 8 KB (Z80)
- **Video:** Sega 315-5313 VDP, 64 KB VRAM, 128 B CRAM, 80 B VSRAM
- **Sound:** Yamaha YM2612 (FM) + TI SN76489 (PSG)
