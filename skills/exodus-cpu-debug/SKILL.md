---
description: "Exodus (Sega Mega Drive/Genesis) MCP tool reference: system control, device listing, 68000/Z80 registers, memory read/write/search, disassembly, breakpoints, watchpoints, stepping, raw VDP VRAM/CRAM/VSRAM and registers, decoded sprites/palette/nametables/VDP state, screenshot and per-pixel rendering info. Use when debugging Mega Drive code, setting breakpoints or watchpoints, inspecting memory, or inspecting VDP graphics state."
---

# Exodus CPU Debugging & Memory Tools

## Address Format

All address parameters accept:
- Motorola hex string: `"$FF0000"` (preferred for M68000)
- C-style hex string: `"0xFF0000"`
- Zilog hex string: `"FF0000h"` (for Z80)
- Plain integer: `16711680`

All addresses returned by tools use the Motorola `$XXXXXX` convention.

## Device Names

Many tools require a `device` parameter. Use `list_devices()` first to get the exact instance names. Typical Mega Drive names:
- `"Main 68000"` -- main CPU
- `"Z80"` -- sound CPU
- `"VDP"` -- video display processor

## System Control (3 tools)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `get_system_status()` | none | Get system status: running/stopped and loaded modules |
| `run_system()` | none | Start or resume emulation |
| `stop_system()` | none | Pause emulation |

## Device Listing (1 tool)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `list_devices()` | none | List all loaded devices with class and instance names |

## CPU Registers (1 tool)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `read_cpu_registers(device)` | `device` (string, required) | Read all CPU registers |

- `device` -- processor instance name, e.g. `"Main 68000"` or `"Z80"`

For M68000, returns full register dump:
```
D0=$00000000  A0=$00FF8000
D1=$0000FFFF  A1=$00C00004
D2=$00000003  A2=$00000200
...
PC=$00204E  SR=$2700  SSP=$00FF8000  USP=$00000000
```
For other processors, returns PC only.

## Memory Access (2 tools)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `read_memory(device, address, length)` | all required | Read bytes from a processor's memory space |
| `write_memory(device, address, data)` | all required | Write bytes to a processor's memory space |

**read_memory parameters:**
- `device` (string, required) -- processor instance name, e.g. `"Main 68000"`
- `address` (string or int, required) -- start address, e.g. `"$FF0000"` or `16711680`
- `length` (int, required) -- number of bytes, 1-4096. For larger ranges use `search_memory`

**write_memory parameters:**
- `device` (string, required) -- processor instance name, e.g. `"Main 68000"`
- `address` (string or int, required) -- start address, e.g. `"$FF0000"`
- `data` (int array, required) -- byte values 0-255, e.g. `[0, 16, 255]`

**M68000 address space:** ROM $000000, RAM $FF0000, VDP $C00000, Z80 $A00000.
**Z80 address space:** RAM $0000, YM2612 $4000, bank-switched 68K bus $8000.

## Memory Search (1 tool)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `search_memory(device, hex, start?, end?)` | device + hex required | Search memory for a byte pattern |

- `device` (string, required) -- processor instance name, e.g. `"Main 68000"`
- `hex` (string, required) -- hex byte pattern without spaces or prefix, e.g. `"00FF8000"` for bytes 00 FF 80 00
- `start` (string or int, optional) -- start address, default `"$000000"`
- `end` (string or int, optional) -- end address, default `"$FFFFFF"`

Returns list of matching addresses (max 64 results):
```
3 match(es):
$00A104
$FF0200
$FF1C80
```

**Tips:**
- Mega Drive is big-endian. To find a 16-bit value like 1000 ($03E8): search for `"03E8"`.
- To find a 32-bit value like 100000 ($000186A0): search for `"000186A0"`.
- Narrow the range for faster searches, e.g. `start="$FF0000"` to search only 68K RAM.

## Disassembly (1 tool)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `disassemble(device, address, count?)` | device + address required | Disassemble CPU instructions |

- `device` (string, required) -- processor instance name, e.g. `"Main 68000"`
- `address` (string or int, required) -- start address, e.g. `"$000200"`
- `count` (int, optional) -- number of real instructions to return, default 10

Returns plain text with one instruction per line. Non-code regions (data tables, tile data) are skipped with a comment:
```
$000200  TST.l    $00A10008
$000206  BNE.b    *+$6
         ; $000208-$00020D: 6 bytes skipped (not code)
$00020E  MOVE.l   #$00FF0000, A0
```
- `count` counts actual instructions, not bytes scanned -- data gaps don't reduce the count.
- If disassembling a data region, most output will be skip comments. Use `read_memory` instead for raw data.

## Breakpoints (3 tools)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `set_breakpoint(device, address)` | both required | Set an execution breakpoint |
| `remove_breakpoint(device, address)` | both required | Remove a breakpoint |
| `list_breakpoints(device)` | required | List all breakpoints on a device |

- `device` (string, required) -- processor instance name, e.g. `"Main 68000"`
- `address` (string or int, required) -- breakpoint address, e.g. `"$000200"`

When a breakpoint hits, emulation pauses automatically. Use `read_cpu_registers()` to inspect, then `run_system()` or `step_device()` to continue.

## Watchpoints (3 tools)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `set_watchpoint(device, address, size?, read?, write?)` | device + address required | Set a memory watchpoint |
| `remove_watchpoint(device, address)` | both required | Remove a watchpoint |
| `list_watchpoints(device)` | required | List all watchpoints on a device |

- `device` (string, required) -- processor instance name, e.g. `"Main 68000"`
- `address` (string or int, required) -- start address, e.g. `"$FF8E00"`
- `size` (int, optional) -- range in bytes, default 1
- `read` (bool, optional) -- break on reads, default false
- `write` (bool, optional) -- break on writes, default true

When a watchpoint triggers, emulation pauses. Use `read_cpu_registers()` to see which instruction caused the access and what values are in the registers (e.g. source pointer in A0).

**Tips:**
- Write watchpoints are ideal for finding code that modifies a RAM variable.
- Set `size` to match the variable width (e.g. 2 for a word, 4 for a long).
- Use `read=true, write=false` to find code that reads from a location.

## Execution Control (1 tool)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `step_device(device)` | required | Execute one instruction on a processor |

- `device` (string, required) -- processor instance name, e.g. `"Main 68000"`
- System must be stopped first via `stop_system()`
- Returns: `{"pc": "$00204E", "status": "stepped"}`

## VDP Raw Memory (4 tools)

**IMPORTANT:** VDP tools access VDP hardware directly. They do NOT take a `device` parameter — do not pass `device`.

| Tool | Parameters | Description |
|------|-----------|-------------|
| `read_vram(address, length)` | both required | Read raw VRAM bytes (64 KB) |
| `read_cram(address, length)` | both required | Read raw CRAM bytes (128 B) |
| `read_vsram(address, length)` | both required | Read raw VSRAM bytes (80 B) |
| `read_vdp_registers()` | none | Read all 24 VDP registers |

**read_vram parameters:**
- `address` (string or int, required) -- VRAM address, e.g. `"$0000"`. Range: $0000-$FFFF
- `length` (int, required) -- bytes to read, 1-4096

**read_cram parameters:**
- `address` (string or int, required) -- CRAM address, e.g. `"$00"`. Range: $00-$7F
- `length` (int, required) -- bytes to read, 1-128

**read_vsram parameters:**
- `address` (string or int, required) -- VSRAM address, e.g. `"$00"`. Range: $00-$4F
- `length` (int, required) -- bytes to read, 1-80

**Notes:**
- These access VDP memory directly, not through the 68K address space.
- Prefer the decoded VDP tools below for sprite, palette, and nametable inspection.
- `read_vdp_registers` returns plain text: `R00 = $04`, `R01 = $74`, etc.

## VDP Decoded Data, Screenshot & Pixel Info (6 tools)

**IMPORTANT:** VDP tools access VDP hardware directly. They do NOT take a `device` parameter — do not pass `device`.

| Tool | Parameters | Description |
|------|-----------|-------------|
| `read_sprite_table()` | none | Decode the full sprite attribute table |
| `read_palette()` | none | Read all 64 colors as 8-bit RGB |
| `read_nametable(plane)` | plane required | Decode a plane's tile map |
| `read_vdp_state()` | none | Read full VDP configuration |
| `screenshot()` | none | Save current frame to a temp PNG file, return its path |
| `query_pixel(x, y)` | both required | Get rendering info for a screen pixel |

### read_sprite_table

No parameters. Returns plain text table (up to 80 sprites, stops at link chain end):
```
#    X      Y     W  H  Pat     Pal  Pri HF VF Link
0    128    200   2x2 $1A0   0    1   0  0  1
1    144    200   2x2 $1A4   0    1   0  0  0
```
Width/height in cells (8px each). Pattern is the VRAM tile index.

### read_palette

No parameters. Returns all 4 rows x 16 colors decoded to 8-bit RGB:
```
Palette 0:
   0: R=  0 G=  0 B=  0 (#000000)
   1: R= 32 G= 64 B=224 (#2040E0)
```

### read_nametable

- `plane` (string, required) -- must be `"a"`, `"b"`, or `"window"`
- `row_start` (int, optional) -- first row to read, default 0
- `row_count` (int, optional) -- number of rows to read, default 8

Returns per-cell columnar format:
```
Plane A: 64x32 cells, base=$C000, rows 0-7
Row Col Pat     Pal Pri HF VF
  0   0 $001   0   0   0  0
  0   1 $002   1   0   0  0
  0   2 $002   1   0   1  1
  1   0 $010   0   1   0  0
```
- Pat = VRAM tile index (each tile is 32 bytes at `pattern * $20`)
- Pal = palette row 0-3. Pri = priority. HF/VF = horizontal/vertical flip
- Default returns 8 rows. Use `row_start`/`row_count` to page through the full plane

### read_vdp_state

No parameters. Returns full decoded VDP configuration:
```
Display: ON, Mode 5, H40 (320px)
Plane size: 64x32 cells (512x256 px)
Nametable A:   $C000
Nametable B:   $E000
Window:        $D000
Sprite table:  $B800
HScroll data:  $BC00
HScroll mode:  Full screen
VScroll mode:  Full screen
Background:    Palette 0, Color 0
Auto-increment: 2 bytes
DMA:           Enabled
```
Use this first to understand the VDP layout before reading nametables or raw VRAM.

### screenshot

No parameters. Captures the current VDP rendered frame and saves it as a PNG to a temp file. Returns the file path.

Output format:
```
Screenshot saved: 320x224
C:\Users\...\AppData\Local\Temp\exo1A2B.png
```

Use the Read tool on the returned path to view the image when visual context is needed. Prefer data tools (`read_vdp_state`, `read_nametable`, `query_pixel`) over screenshot for structured analysis — only read the screenshot file when you need to visually identify what's on screen.

**Tips:**
- The image is the full rendered frame (including border if visible).
- Works while the system is running or stopped.
- The file is written to the system temp directory and persists until cleaned up.

### query_pixel

- `x` (int, required) -- horizontal pixel position (0 = left edge). H32: 0-255, H40: 0-319
- `y` (int, required) -- vertical pixel position (0 = top edge). V28: 0-223, V30: 0-239

Returns detailed per-pixel rendering info:
```
Pixel (128,100)
Source: Layer A
Color: R=32 G=64 B=224 (#2040E0)
Palette: row 1, entry 5
HCounter=384 VCounter=100
Mapping VRAM addr: $C108
Tile: $01A  Pal: 1  Pri: 0  HF: 0  VF: 0
Pattern pos: row 4, col 0
Tile data VRAM addr: $0340
```

For sprite pixels, also includes:
```
Sprite entry: #3 (table addr $B818)
Sprite cell: 2x2 at (1,0)
```

**Field reference:**
- **Source** -- which layer produced this pixel: Sprite, Layer A, Layer B, Window, Background, Border, Blanking, CRAM Write
- **Mapping VRAM addr** -- the nametable entry address in VRAM that defines this tile
- **Tile** -- VRAM tile pattern index. Tile data is at `tile_index * $20` in VRAM
- **Pattern pos** -- which row/column within the 8x8 tile this pixel falls on
- **Sprite entry** -- the sprite attribute table entry number and its VRAM address

**Tips:**
- This is the fastest way to trace a visible element back to its source data. Point at a pixel and get the full rendering chain.
- For layer pixels, use the mapping VRAM address to find the nametable entry, and the tile index to read the pattern data with `read_vram`.
- For sprite pixels, cross-reference the sprite entry number with `read_sprite_table()` for the full sprite attributes.
- Combine with `read_palette()` to see the exact color values for the palette row/entry.
