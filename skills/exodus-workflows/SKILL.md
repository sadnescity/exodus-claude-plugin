---
description: "Exodus Mega Drive/Genesis reverse engineering workflows: find code that writes to RAM, cheat search, trace execution, verify ROM patches, find text strings and pointer tables, vector table/VBlank analysis, VDP DMA tracing, controller input tracing, game state machines, trace a screen pixel back to its tile in ROM. Use when planning or executing Mega Drive runtime debugging tasks."
---

# Exodus Reverse Engineering Workflows

## Workflow A: Find What Code Writes to a RAM Address

Use this when you know the address of a variable and want to find the code responsible for modifying it.

1. `stop_system()` -- pause emulation
2. `set_watchpoint(device="Main 68000", address="$FF8E00", size=4, write=true)` -- watch for writes to the address
3. `run_system()` -- resume and trigger the write in-game
4. Wait for the watchpoint to hit (emulator pauses automatically)
5. `read_cpu_registers(device="Main 68000")` -- D0-D7/A0-A7 show the full context (source pointer, values being written)
6. `disassemble(device="Main 68000", address=PC, count=16)` -- view the code that triggered the write
7. `remove_watchpoint(device="Main 68000", address="$FF8E00")` -- clean up

**Tips:**
- Set `size` to match the variable width (2 for word, 4 for long).
- The register dump after a watchpoint hit often reveals the source address in an A register.
- Use `read=true, write=false` to catch code that reads from a location instead.

## Workflow B: Find a Game Variable by Value (Cheat Search)

Use this when you can see a value on screen (lives, score, HP) but do not know its RAM address.

1. `stop_system()` at a known state (e.g. when lives = 3)
2. `search_memory(device="Main 68000", hex="0003", start="$FF0000", end="$FFFFFF")` -- search 68K RAM for the value (big-endian 16-bit: 3 = `0003`)
3. Note the candidate addresses
4. `run_system()` -- change the value in-game (e.g. lose a life so lives = 2)
5. `stop_system()`
6. `search_memory(device="Main 68000", hex="0002", start="$FF0000", end="$FFFFFF")` -- search for the new value
7. Compare the two result sets -- addresses present in both are strong candidates
8. `read_memory(device="Main 68000", address="$FFXXXX", length=16)` -- verify the value at the candidate
9. `write_memory(device="Main 68000", address="$FFXXXX", data=[0,99])` -- write 99 to confirm

**Tips:**
- Mega Drive is big-endian. 16-bit 1000 = `"03E8"`, 32-bit 1000 = `"000003E8"`.
- For 8-bit values (e.g. lives), search with `"03"` but expect many false positives -- narrow with `start`/`end`.
- Some games store scores and timers as BCD (binary-coded decimal): 1000 = `"1000"` not `"03E8"`. If hex search fails, try BCD.
- Use `read_vdp_state()` to check if the value might be in VDP registers instead of RAM.

## Workflow C: Trace Code Execution

Use this to understand control flow through a function.

1. `stop_system()` -- pause emulation
2. `read_cpu_registers(device="Main 68000")` -- get current PC
3. `disassemble(device="Main 68000", address="$XXXXXX", count=20)` -- see what's ahead
4. `step_device(device="Main 68000")` -- step one instruction
5. `read_cpu_registers(device="Main 68000")` -- check new PC
6. Repeat steps 3-5 to trace through the logic

## Workflow D: Verify ROM Patches at Runtime

Use this after applying patches to confirm they are correct in memory.

1. Load the ROM in Exodus
2. `stop_system()` -- pause after ROM is loaded
3. `read_memory(device="Main 68000", address="$XXXXXX", length=N)` -- read bytes at the patched location
4. Compare hex bytes with expected patch values
5. `disassemble(device="Main 68000", address="$XXXXXX")` -- verify instruction mnemonics

## Workflow E: Inspect Full System State

Use this to understand the current state of all system components.

1. `stop_system()` -- pause emulation
2. `screenshot()` -- capture what's on screen for visual context
3. `list_devices()` -- see all devices
4. `read_cpu_registers(device="Main 68000")` -- check main CPU PC
5. `read_cpu_registers(device="Z80")` -- check sound CPU PC
6. `read_vdp_state()` -- get VDP configuration (plane sizes, base addresses, display mode)
7. `read_palette()` -- see current color palette
8. `read_sprite_table()` -- see active sprites
9. `read_memory(device="Main 68000", address="$FF0000", length=256)` -- inspect start of 68K RAM

## Workflow F: Set Up Multi-Breakpoint Investigation

Use this to monitor multiple code paths simultaneously.

1. `stop_system()`
2. `set_breakpoint(device="Main 68000", address="$XXXX")` -- VBlank handler
3. `set_breakpoint(device="Main 68000", address="$YYYY")` -- game logic entry
4. `set_breakpoint(device="Main 68000", address="$ZZZZ")` -- DMA routine
5. `list_breakpoints(device="Main 68000")` -- verify all are set
6. `run_system()` -- first hit reveals execution order
7. `read_cpu_registers(device="Main 68000")` -- inspect state at each hit
8. `remove_breakpoint(device="Main 68000", address="$XXXX")` -- remove as you finish each

## Workflow G: Investigate VDP Graphics

Use this to understand how the game renders graphics.

1. `stop_system()` -- pause during gameplay
2. `read_vdp_state()` -- get plane sizes, nametable addresses, scroll modes
3. `read_nametable(plane="a")` -- see Plane A tile layout
4. `read_nametable(plane="b")` -- see Plane B tile layout
5. `read_sprite_table()` -- see all active sprites with positions, sizes, patterns
6. `read_palette()` -- see all 64 colors across 4 palette rows
7. `read_vram(address="$0000", length=32)` -- read a specific tile pattern (32 bytes = one 8x8 tile)
8. `read_vsram(address="$00", length=80)` -- see per-column vertical scroll values
9. `query_pixel(x=160, y=112)` -- pick any pixel to see which layer, tile, and palette produced it

## Workflow H: Find a Specific Byte Sequence in ROM

Use this to locate data structures, text strings, or known code patterns in the ROM.

1. `stop_system()`
2. `search_memory(device="Main 68000", hex="XXXX", start="$000000", end="$3FFFFF")` -- search entire ROM area
3. For each result, `disassemble(device="Main 68000", address="$XXXXXX")` or `read_memory(...)` to examine context

## Workflow I: Find Text Strings in ROM

Use this when looking for in-game text for translation, analysis, or modification.

1. `stop_system()`
2. Convert the target text to hex bytes using the game's encoding:
   - ASCII: "HELLO" = `"48454C4C4F"`
   - Shift-JIS: convert each character to its 2-byte code
   - Custom table: use the game-specific character mapping (if known)
3. `search_memory(device="Main 68000", hex="48454C4C4F", start="$000000", end="$3FFFFF")` -- search ROM for the encoded string
4. `read_memory(device="Main 68000", address="$RESULT", length=64)` -- read context around the match to see the full string and any preceding/following control codes
5. If the encoding is unknown, try reading visible text areas with `read_memory` and look for patterns in byte values to deduce the character table

**Tips:**
- Many games use custom character tables where A=0, B=1, etc. (not ASCII). If ASCII search fails, look for sequential byte runs that match sequential letters.
- Text entries are often terminated by $00 or $FF.
- Control codes for newlines, pauses, and name variables are typically bytes in the $F0-$FF range.
- Japanese games often use Shift-JIS, but some use custom 1-byte encodings for katakana/hiragana.

## Workflow J: Pointer Table Discovery

Use this after finding data (text, graphics, level layouts) to locate the pointer table that references it.

1. Find the data's ROM address using Workflow H or I (e.g. string at `$01A3C0`)
2. Convert the address to big-endian hex bytes:
   - 32-bit pointer: `$01A3C0` → `"0001A3C0"`
   - 24-bit pointer (common): `$01A3C0` → `"01A3C0"`
   - 16-bit offset (if data is in a known bank): just the low 16 bits `"A3C0"`
3. `search_memory(device="Main 68000", hex="0001A3C0", start="$000000", end="$3FFFFF")` -- search ROM for the pointer value
4. `read_memory(device="Main 68000", address="$RESULT-8", length=32)` -- read around the match to see the full pointer table (adjacent entries should point to nearby data)
5. Verify adjacent pointers by reading what they point to:
   `read_memory(device="Main 68000", address="$NEXT_PTR_VALUE", length=32)` -- should be similar data (next string, next graphic, etc.)

**Tips:**
- If 32-bit search finds nothing, try 16-bit offset. Some games use base + offset tables.
- Pointer tables are usually contiguous and sorted. If you see 4-byte aligned values incrementing, that's a table.
- Some games use relative offsets from the table start rather than absolute ROM addresses.
- Once you find the table, count entries to determine the total number of items (strings, levels, etc.).

## Workflow K: Interrupt Vector Analysis

Use this as a first step when starting RE on an unknown ROM. The M68000 vector table at $000000 reveals key entry points.

1. `stop_system()`
2. `read_memory(device="Main 68000", address="$000000", length=16)` -- read the first 4 vectors:
   - $000000: Initial SSP (stack pointer)
   - $000004: Reset vector (program entry point)
   - $000008: Bus error handler
   - $00000C: Address error handler
3. `read_memory(device="Main 68000", address="$000060", length=32)` -- read the 68000 autovectors (levels 1-7). On the Mega Drive only levels 2, 4 and 6 are wired:
   - $000060: Spurious interrupt
   - $000064: Level 1 (unused)
   - $000068: Level 2 -- external interrupt (EXT): TH pin of a controller port, used by light guns; enabled by VDP register 11 bit 3
   - $00006C: Level 3 (unused)
   - $000070: Level 4 -- HBlank handler (horizontal interrupt, enabled by VDP register 0 bit 4, interval in register 10)
   - $000074: Level 5 (unused)
   - $000078: Level 6 -- VBlank handler (vertical interrupt, enabled by VDP register 1 bit 5 -- runs every frame)
   - $00007C: Level 7 (unused, NMI)
4. `disassemble(device="Main 68000", address=RESET_VECTOR, count=30)` -- disassemble the reset/entry point to find initialization code
5. `disassemble(device="Main 68000", address=VBLANK_VECTOR, count=30)` -- disassemble the VBlank handler (this is the main per-frame update, often calls game logic, DMA, controller reads)

**Tips:**
- The reset vector is where the game starts. Follow it to find hardware init, SEGA header check, and jump to main loop.
- The VBlank handler is the most important interrupt -- it runs once per frame and typically handles DMA transfers, palette updates, scroll updates, and triggers the main game logic.
- Many games use a thin VBlank handler that just sets a flag, with the main loop polling that flag. Others do everything in VBlank.
- HBlank is used for raster effects (water wobble, sky gradients). Not all games use it.

## Workflow L: DMA Transfer Tracing

Use this to find what code loads graphics into VRAM via DMA. Watchpoints work on all memory-mapped I/O, including VDP ports.

1. `stop_system()`
2. `set_watchpoint(device="Main 68000", address="$C00004", size=4, write=true)` -- catch writes to VDP control port (used for DMA setup and all VDP register writes)
3. `run_system()` -- resume emulation
4. Wait for the watchpoint to hit (emulator pauses automatically)
5. `read_cpu_registers(device="Main 68000")` -- check registers. The data register (usually D0) holds the value being written to the control port
6. `disassemble(device="Main 68000", address=PC-16, count=20)` -- view surrounding code to see the full DMA setup sequence
7. `read_vdp_registers()` -- read VDP registers to see DMA configuration:
   - R19 ($13): DMA length low byte
   - R20 ($14): DMA length high byte
   - R21 ($15): DMA source low byte
   - R22 ($16): DMA source mid byte
   - R23 ($17): DMA source high byte + DMA type (bits 6-7: 00=68K→VRAM, 10=VRAM fill, 11=VRAM copy)
8. `remove_watchpoint(device="Main 68000", address="$C00004")` -- clean up

**Tips:**
- The VDP control port receives ALL register writes, not just DMA. Expect frequent hits. Look for DMA-specific patterns: writes to R19-R23 followed by the final control word with bit 7 set.
- DMA source address in registers R21-R23 is a word address (shift left by 1 to get the byte address in 68K space).
- To narrow hits, set a breakpoint at the VBlank handler instead (find it via Workflow K) and step through -- most games do DMA exclusively during VBlank.
- Common DMA sequence: set R19-R20 (length), R21-R23 (source+type), then write destination + DMA trigger to the control port.

## Workflow M: Controller Input Tracing

Use this to find the input handling code. Watchpoints work on I/O port addresses since the M68000 uses memory-mapped I/O.

1. `stop_system()`
2. `set_watchpoint(device="Main 68000", address="$A10003", size=1, read=true)` -- catch reads from controller 1 data port
3. `run_system()` -- resume emulation
4. Wait for the watchpoint to hit (emulator pauses automatically)
5. `read_cpu_registers(device="Main 68000")` -- registers show the context of the read
6. `disassemble(device="Main 68000", address=PC-8, count=20)` -- view the controller polling routine
7. Look for where the read value is stored in RAM (typically a MOVE.b to a RAM address like $FF00XX)
8. `read_memory(device="Main 68000", address="$FF00XX", length=4)` -- read the RAM copy of controller state
9. `remove_watchpoint(device="Main 68000", address="$A10003")` -- clean up
10. Now use Workflow A on the RAM copy address to trace game logic that reacts to input

**Tips:**
- Controller 1 data: `$A10003`. Controller 2 data: `$A10005`. Control registers: `$A10009`/`$A1000B`.
- Most games read the port twice per frame (for the 3-button protocol) or more (for 6-button). The polling routine typically XORs current and previous state to detect new presses vs. held buttons.
- The RAM variable storing controller state usually has: raw state, previous frame state, and "newly pressed" (current AND NOT previous). Find all three to understand input handling fully.
- After finding the polling routine, set a breakpoint after it returns and inspect the RAM variables to see the button mapping.
- Button bits (active low): Up=0, Down=1, Left=2, Right=3, B=4, C=5, A=6, Start=7.

## Workflow N: Game Loop and State Machine Analysis

Use this to understand the game's main loop structure and how it transitions between states (title screen, gameplay, pause, game over).

1. Find the VBlank handler using Workflow K
2. `set_breakpoint(device="Main 68000", address=VBLANK_VECTOR)` -- break at the start of each frame
3. `run_system()` -- let it hit
4. `read_cpu_registers(device="Main 68000")` -- inspect state
5. `disassemble(device="Main 68000", address=VBLANK_VECTOR, count=40)` -- look for a JSR/BSR to the main game logic, or a jump table indexed by a state variable
6. Look for a pattern like: `MOVE.b ($FFXXXX), D0` followed by a jump table (`ADD D0,D0; JMP (PC,D0)` or `LEA table(PC),A0; MOVEA.l (A0,D0),A0; JMP (A0)`) -- this is the state dispatcher
7. `read_memory(device="Main 68000", address="$FFXXXX", length=1)` -- read the current state variable
8. `write_memory(device="Main 68000", address="$FFXXXX", data=[XX])` -- write a different state value to test (e.g. jump to game over screen)
9. `remove_breakpoint(...)` and `run_system()` -- verify the state change

**Tips:**
- State variables are typically 1 byte (8-bit index) or 1 word (16-bit). Common states: 0=SEGA logo, 1=title screen, 2=options, 3=gameplay, 4=pause, 5=game over.
- The main loop is often outside VBlank: a `STOP #$2700` / wait-for-VBlank pattern followed by game logic. The VBlank handler just sets a flag and returns.
- If the VBlank handler is thin (just sets a flag), look for the main loop by disassembling the reset vector code and following the initialization to the first infinite loop.
- Jump tables using `JMP (A0,D0.w)` or `MOVE.w (PC,D0.w),D0; JMP (PC,D0.w)` are very common in 68K games for state dispatch.

## Workflow O: Tile Provenance -- Trace a Screen Pixel to Its Source

Use this to find where a specific on-screen element comes from in ROM and what code loaded it.

1. `stop_system()` -- pause during the frame you want to inspect
2. `screenshot()` -- capture the frame for visual reference
3. `query_pixel(x=PIXEL_X, y=PIXEL_Y)` -- point at the pixel you're interested in. This returns:
   - **Source**: which layer produced it (Sprite, Layer A, Layer B, Window)
   - **Tile index**: the VRAM pattern number
   - **Mapping VRAM addr**: where the nametable entry lives in VRAM
   - **Tile data VRAM addr**: where the tile pattern data lives (`tile_index * $20`)
   - **Palette**: which row and entry
   - For sprites: the sprite table entry number and cell position
4. `read_vram(address=TILE_DATA_ADDR, length=32)` -- read the raw tile pattern (32 bytes = one 8x8 4bpp tile)
5. `search_memory(device="Main 68000", hex="TILE_HEX", start="$000000", end="$3FFFFF")` -- search ROM for the tile data
   - If found: the tile is stored uncompressed in ROM at that address
   - If not found: the tile data is likely compressed or generated at runtime
6. If found in ROM, search for a pointer to that address using Workflow J to find the tileset table
7. To find what code loaded the tile into VRAM, use Workflow L (DMA tracing) or set a write watchpoint on the VDP data port:
   `set_watchpoint(device="Main 68000", address="$C00000", size=4, write=true)` -- catch writes that load tile data

**Tips:**
- `query_pixel` eliminates the manual guesswork of mapping screen coordinates to nametable rows/columns. Just point and click.
- If the source is "Sprite", cross-reference the sprite entry number with `read_sprite_table()` for full sprite attributes (position, size, link chain).
- If the tile is not found uncompressed in ROM, the game uses a decompression routine. Set a write watchpoint on the VRAM destination to catch the decompressor, then disassemble the code at that point.
- Pattern index $000 is often a blank/empty tile. Nonzero patterns are the interesting ones.
- Use `read_palette()` with the palette row from `query_pixel` to see the exact 16 colors used by that tile.

**Fallback -- nametable approach (when you don't have exact pixel coordinates or need to scan a region):**

1. `read_vdp_state()` -- get plane sizes and nametable base addresses
2. `read_nametable(plane="a", row_start=ROW, row_count=1)` -- read the row containing the tile. Note the `Pat` (tile index) for the target column
3. Calculate the VRAM address: `tile_vram_addr = pattern_index * $20`
4. Continue from step 4 above (read tile data, search ROM, etc.)

This is useful when scanning a whole row for non-blank tiles, comparing tile usage across rows, or when you know the nametable coordinates but not the screen pixel position (e.g. after accounting for scroll offsets).
