# Platform-Specific 6502 Information

## Table of Contents
- NES / Famicom
- Commodore 64
- Apple II
- Atari 2600
- Atari 8-Bit
- BBC Micro

## NES / Famicom

**CPU**: Ricoh 2A03 (NMOS 6502 core, no decimal mode, built-in audio)
**Clock**: 1.7897725 MHz (NTSC), 1.662607 MHz (PAL)
**RAM**: 2KB internal ($0000–$07FF, mirrored to $1FFF)

### Memory Map
```
$0000-$00FF  Zero Page (256 bytes)
$0100-$01FF  Stack (256 bytes)
$0200-$02FF  OAM DMA page (sprite data, conventionally)
$0300-$07FF  General purpose RAM
$0800-$1FFF  Mirrors of $0000-$07FF
$2000-$2007  PPU registers
$2008-$3FFF  Mirrors of PPU registers
$4000-$4017  APU and I/O registers
$4018-$401F  APU test registers (disabled)
$4020-$FFFF  Cartridge space (PRG ROM/RAM, mapper registers)
$FFFA-$FFFB  NMI vector
$FFFC-$FFFD  RESET vector
$FFFE-$FFFF  IRQ vector
```

### Key I/O Registers
```
PPU_CTRL   = $2000    ; NMI enable, sprite size, BG pattern table
PPU_MASK   = $2001    ; Rendering enable, clipping
PPU_STATUS = $2002    ; Vblank flag (read clears it), sprite 0 hit
OAM_ADDR   = $2003    ; OAM address for OAM_DATA
OAM_DATA   = $2004    ; OAM read/write
PPU_SCROLL = $2005    ; Scroll position (write twice: X then Y)
PPU_ADDR   = $2006    ; VRAM address (write twice: high then low)
PPU_DATA   = $2007    ; VRAM read/write
OAM_DMA    = $4014    ; Write page number to trigger 256-byte OAM DMA
JOYPAD1    = $4016    ; Controller 1 (write to strobe, read bits)
JOYPAD2    = $4017    ; Controller 2
```

### NES Startup Template
```asm
    .segment "HEADER"
    .byte "NES", $1A    ; iNES header magic
    .byte 2             ; 2 × 16KB PRG ROM
    .byte 1             ; 1 × 8KB CHR ROM
    .byte $01           ; Mapper 0, vertical mirroring
    .byte $00

    .segment "CODE"
reset:
    SEI                 ; disable IRQs
    CLD                 ; clear decimal mode (no effect on 2A03, but good practice)
    LDX #$40
    STX $4017           ; disable APU frame IRQ
    LDX #$FF
    TXS                 ; set up stack
    INX                 ; X = 0
    STX $2000           ; disable NMI
    STX $2001           ; disable rendering
    STX $4010           ; disable DMC IRQs

wait_vblank1:           ; wait for first vblank
    BIT $2002
    BPL wait_vblank1

clear_mem:              ; clear RAM
    LDA #$00
    STA $0000,X
    STA $0100,X
    STA $0200,X
    STA $0300,X
    STA $0400,X
    STA $0500,X
    STA $0600,X
    STA $0700,X
    INX
    BNE clear_mem

wait_vblank2:           ; wait for second vblank
    BIT $2002
    BPL wait_vblank2

    ; PPU is ready. Set up nametables, palettes, etc.
    ; Enable NMI and rendering.
    JMP main_loop

nmi:
    ; Called every vblank — do PPU updates here
    RTI

irq:
    RTI

    .segment "VECTORS"
    .word nmi
    .word reset
    .word irq
```

### Reading Controller Input
```asm
; Strobe and read 8 buttons into buttons variable
read_controller:
    LDA #$01
    STA JOYPAD1         ; begin strobe
    LDA #$00
    STA JOYPAD1         ; end strobe
    LDX #8
read_loop:
    LDA JOYPAD1         ; bit 0 = button state
    LSR                 ; shift into carry
    ROL buttons         ; rotate carry into buttons
    DEX
    BNE read_loop
    RTS
; buttons: A, B, Select, Start, Up, Down, Left, Right (bit 7 to bit 0)
```

## Commodore 64

**CPU**: MOS 6510 (6502 + I/O port at $00/$01)
**Clock**: 0.985 MHz (PAL), 1.023 MHz (NTSC)
**RAM**: 64KB (with bank switching via $01)

### Memory Map (Default Configuration)
```
$0000-$0001  6510 I/O port (DDR and data)
$0002-$00FF  Zero page (much used by BASIC/Kernal)
$0100-$01FF  Stack
$0200-$03FF  OS variables
$0400-$07FF  Screen RAM (default, 1000 bytes + spare)
$0800-$9FFF  BASIC program area
$A000-$BFFF  BASIC ROM (can be switched to RAM)
$C000-$CFFF  RAM (always accessible)
$D000-$D3FF  VIC-II video chip registers
$D400-$D7FF  SID sound chip registers
$D800-$DBFF  Color RAM (nybbles, always visible)
$DC00-$DCFF  CIA #1 (keyboard, joystick, timer)
$DD00-$DDFF  CIA #2 (serial, timer, VIC bank select)
$E000-$FFFF  Kernal ROM (can be switched to RAM)
```

### Key I/O Addresses
```
; VIC-II
$D020  Border color
$D021  Background color
$D011  Screen control (vertical scroll, screen height, mode)
$D016  Screen control (horizontal scroll, multicolor, 38/40 col)
$D012  Raster line counter (read/compare)
$D019  Interrupt status register
$D01A  Interrupt enable register

; SID
$D400  Voice 1 frequency low
$D401  Voice 1 frequency high
$D404  Voice 1 control (waveform, gate)
$D418  Volume and filter mode

; CIA #1
$DC00  Port A (keyboard column / joystick port 2)
$DC01  Port B (keyboard row / joystick port 1)
```

### C64 Raster Interrupt Setup
```asm
    SEI
    LDA #$7F
    STA $DC0D           ; disable CIA interrupts
    STA $DD0D
    LDA $DC0D           ; acknowledge pending CIA IRQs
    LDA $DD0D

    LDA #<irq_handler   ; set IRQ vector
    STA $0314
    LDA #>irq_handler
    STA $0315

    LDA #$01
    STA $D01A           ; enable VIC raster interrupt
    LDA #100            ; trigger at raster line 100
    STA $D012
    LDA $D011
    AND #$7F            ; clear bit 7 of $D011 (raster high bit)
    STA $D011
    CLI
    RTS

irq_handler:
    LDA $D019           ; acknowledge VIC interrupt
    STA $D019
    ; ... your raster code here ...
    JMP $EA31           ; jump to normal Kernal IRQ handler
```

## Apple II

**CPU**: MOS 6502 (original), 65C02 (IIe enhanced, IIc)
**Clock**: 1.023 MHz
**RAM**: 48KB (II+), 64KB–128KB (IIe/IIc)

### Memory Map
```
$0000-$00FF  Zero page
$0100-$01FF  Stack
$0200-$02FF  Input buffer
$0300-$03FF  Programmable area / DOS hooks
$0400-$07FF  Text page 1 / Lo-res graphics page 1
$0800-$0BFF  Text page 2 / Lo-res graphics page 2
$0C00-$1FFF  Free RAM
$2000-$3FFF  Hi-res graphics page 1
$4000-$5FFF  Hi-res graphics page 2
$6000-$95FF  Free RAM (typical)
$C000-$C0FF  Soft switches (I/O)
$C100-$C7FF  Peripheral card ROM (slot 1-7)
$C800-$CFFF  Shared peripheral ROM space
$D000-$FFFF  ROM (Monitor, Applesoft BASIC, or Integer BASIC)
```

### Soft Switches (Some Key Ones)
```
$C000  KBD — last key pressed (bit 7 = strobe)
$C010  KBDSTRB — clear keyboard strobe
$C030  SPKR — toggle speaker
$C050  TXTCLR — switch to graphics mode
$C051  TXTSET — switch to text mode
$C054  PAGE1 — display page 1
$C055  PAGE2 — display page 2
$C056  LORES — switch to lo-res
$C057  HIRES — switch to hi-res
```

## Atari 2600

**CPU**: MOS 6507 (6502 with 13 address lines, 8KB addressable)
**Clock**: 1.19 MHz (NTSC)
**RAM**: 128 bytes ($80–$FF)
**ROM**: 4KB typical (bank switching for more)

The 2600 has no frame buffer — the CPU must generate the display in real-time,
one scanline at a time ("racing the beam"). This is the most constrained 6502
platform, requiring cycle-exact code for all graphics.

### Memory Map
```
$00-$7F  TIA (Television Interface Adapter) registers
$80-$FF  RAM (128 bytes, also stack)
$F000-$FFFF  ROM (4KB cartridge)
$FFFC-$FFFD  RESET vector
$FFFE-$FFFF  IRQ vector (not normally used)
```

### Kernel Loop Pattern
```asm
; Basic 2600 frame structure
frame:
    ; --- Vertical Sync (3 scanlines) ---
    LDA #$02
    STA VSYNC           ; turn on VSYNC
    STA WSYNC           ; wait for end of scanline 1
    STA WSYNC           ; scanline 2
    STA WSYNC           ; scanline 3
    LDA #$00
    STA VSYNC           ; turn off VSYNC

    ; --- Vertical Blank (37 scanlines) ---
    ; Set timer for vblank period
    LDA #43
    STA TIM64T           ; ~37 scanlines worth
    ; ... do game logic during vblank ...
wait_vblank:
    LDA INTIM
    BNE wait_vblank
    STA WSYNC
    STA VBLANK           ; turn off vblank

    ; --- Visible Scanlines (192 lines) ---
    LDY #192
scanline:
    STA WSYNC            ; wait for start of scanline
    ; ... set TIA registers for this scanline ...
    DEY
    BNE scanline

    ; --- Overscan (30 scanlines) ---
    LDA #$02
    STA VBLANK           ; turn on vblank
    LDA #35
    STA TIM64T           ; ~30 scanlines
wait_overscan:
    LDA INTIM
    BNE wait_overscan
    JMP frame
```

## Atari 8-Bit (400/800/XL/XE)

**CPU**: MOS 6502
**Clock**: 1.79 MHz (NTSC), 1.77 MHz (PAL)
**RAM**: 48KB–130KB depending on model

### Memory Map
```
$0000-$00FF  Zero page (many locations used by OS)
$0100-$01FF  Stack
$0200-$04FF  OS variables and buffers
$0500-$9FFF  Free RAM (typical)
$A000-$BFFF  BASIC cartridge / RAM
$C000-$CFFF  RAM (XL/XE) or unused
$D000-$D0FF  GTIA graphics chip
$D200-$D2FF  POKEY sound/I/O chip
$D300-$D3FF  PIA parallel I/O
$D400-$D4FF  ANTIC display chip
$E000-$FFFF  OS ROM
```

## BBC Micro

**CPU**: MOS 6502A (2 MHz)
**RAM**: 32KB (Model B)

### Memory Map
```
$0000-$00FF  Zero page (shared with OS)
$0100-$01FF  Stack
$0200-$02FF  OS workspace
$0300-$03FF  OS workspace / vectors
$0400-$07FF  Language workspace (BASIC uses this)
$0800-$2FFF  User program space (varies with mode)
$3000-$7FFF  Screen memory (varies with mode)
$8000-$BFFF  Paged ROM (16KB sideways ROMs)
$C000-$FBFF  OS ROM
$FC00-$FEFF  I/O (SHEILA page, VIA, etc.)
$FF00-$FFFF  OS ROM (vectors)
```

### OSBYTE/OSWORD Calls
The BBC OS provides high-level calls through `JSR $FFF4` (OSBYTE) and
`JSR $FFF1` (OSWORD) with command number in A.
