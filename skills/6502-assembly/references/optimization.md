# 6502 Optimization Techniques

## Table of Contents
- Cycle Counting Basics
- Zero Page Optimization
- Loop Optimization
- Self-Modifying Code
- Unrolling
- Common Peephole Optimizations
- Size vs Speed Tradeoffs

## Cycle Counting Basics

The 6502 runs at 1 cycle per clock. At 1 MHz (Apple II, C64 PAL), 1 cycle = 1 μs.
At 1.79 MHz (NES NTSC), 1 cycle ≈ 559 ns.

A typical NES frame is ~29,780 CPU cycles (NTSC). Every cycle matters in
time-critical code like raster effects or audio processing.

Key cycle costs:
- Zero page access: 3 cycles (load/store)
- Absolute access: 4 cycles (load), 4-5 cycles (store)
- Indexed access: 4-5 cycles (adds 1 if page boundary crossed)
- Indirect,Y access: 5-6 cycles
- Branch not taken: 2 cycles
- Branch taken, same page: 3 cycles
- Branch taken, page cross: 4 cycles
- JSR: 6 cycles. RTS: 6 cycles. Total overhead: 12 cycles per call.

## Zero Page Optimization

Zero page is the most valuable memory on the 6502. Accessing zero page saves
1 byte and 1 cycle per instruction vs absolute addressing.

```asm
; Absolute: 3 bytes, 4 cycles
    LDA $0300       ; AD 00 03

; Zero page: 2 bytes, 3 cycles
    LDA $80         ; A5 80
```

**Rules**:
- Put your most-accessed variables in zero page
- Temporary loop variables and pointers belong in zero page
- Reserve contiguous pairs for 16-bit pointers (required for indirect modes)
- Document your zero page allocation — it's a scarce resource

**Platform ZP budgets** (approximate free bytes):
- NES: ~$00–$FF (most available, no BASIC/OS claims)
- C64: ~$02–$7F after BASIC/Kernal (more with BASIC off)
- Apple II: ~$06–$FF (some used by monitor/DOS)

## Loop Optimization

### Count Down Instead of Up

Counting down to zero eliminates the CMP instruction:

```asm
; Slow: 7 cycles per iteration
    LDX #0
loop:
    ; ... body ...
    INX             ; 2 cycles
    CPX #100        ; 2 cycles
    BNE loop        ; 3 cycles (taken)

; Fast: 5 cycles per iteration (saves 2 cycles × 100 = 200 cycles)
    LDX #100
loop:
    ; ... body ...
    DEX             ; 2 cycles (sets Z flag when X=0)
    BNE loop        ; 3 cycles (taken)
```

### Avoid Page-Crossing Branches

Branches that cross a 256-byte page boundary cost an extra cycle. Align
hot loops to avoid this:

```asm
    .align 256       ; some assemblers support this
loop:
    ; ... keep loop body < 128 bytes if branching back to start
    DEX
    BNE loop
```

### Unrolled Loops

Trading code size for speed by repeating the loop body:

```asm
; Unrolled ×4: fill 256 bytes with A
; Standard: ~7 cycles × 256 = 1792 cycles
; Unrolled ×4: ~3.5 cycles × 256 = 896 cycles
    LDY #0
fill_unrolled:
    STA (dest),Y
    INY
    STA (dest),Y
    INY
    STA (dest),Y
    INY
    STA (dest),Y
    INY
    BNE fill_unrolled
```

### Duff's Device (Partial Unrolling)

When the count isn't a multiple of the unroll factor, jump into the middle:

```asm
; Copy X bytes (X = 1-4), partially unrolled
    DEX
    BEQ copy1
    DEX
    BEQ copy2
    DEX
    BEQ copy3
copy4:  LDA (src),Y
        STA (dest),Y
        INY
copy3:  LDA (src),Y
        STA (dest),Y
        INY
copy2:  LDA (src),Y
        STA (dest),Y
        INY
copy1:  LDA (src),Y
        STA (dest),Y
        RTS
```

## Self-Modifying Code

Since the 6502 has no instruction cache, code in RAM can modify itself.
This is a powerful optimization but makes code harder to read and
impossible to run from ROM.

### Patching an Address

```asm
; Instead of using a ZP pointer + indirect addressing (5+ cycles),
; modify the absolute address in the instruction itself (4 cycles):
    LDA data_addr
    STA load_instr+1    ; patch low byte of address
    LDA data_addr+1
    STA load_instr+2    ; patch high byte of address
load_instr:
    LDA $0000           ; this address gets overwritten!
```

### Patching an Immediate Value

```asm
; Store a computed value directly into an instruction
    LDA computed_value
    STA cmp_instr+1     ; patch the immediate operand
cmp_instr:
    CMP #$00            ; this $00 gets overwritten
    BEQ match
```

### Patching a Branch Target

```asm
; Toggle between two behaviors by changing a branch
    LDA #$D0            ; BNE opcode
    STA branch_instr    ; enable the branch
    ; or
    LDA #$EA            ; NOP opcode
    STA branch_instr    ; disable the branch
branch_instr:
    BNE target          ; (or NOP, depending on patch)
```

**Caution**: Self-modifying code cannot run from ROM. If your code is in
cartridge ROM, this technique is unavailable.

## Common Peephole Optimizations

### Replace JSR+RTS with JMP

If a JSR is followed immediately by RTS, replace with JMP:

```asm
; Before (12 cycles overhead):
    JSR subroutine
    RTS

; After (3 cycles):
    JMP subroutine      ; subroutine's RTS returns to our caller
```

### Use TAX/TAY Instead of STA+LDX/LDY

```asm
; Before (6 cycles):
    STA temp
    LDX temp

; After (2 cycles):
    TAX
```

### Avoid Redundant Loads

```asm
; Before:
    LDA #$00
    STA var1
    LDA #$00        ; A already contains 0!
    STA var2

; After:
    LDA #$00
    STA var1
    STA var2        ; A still contains 0
```

### Use BIT for Flag Testing Without Destroying A

```asm
; BIT reads bits 7→N and 6→V from memory without modifying A
    BIT port_status
    BMI bit7_set        ; N flag = bit 7 of port_status
    BVS bit6_set        ; V flag = bit 6 of port_status
```

### Use EOR #$FF + CLC + ADC #$01 for Negation (or SEC + EOR #$FF + ADC #$00)

```asm
; Negate A (two's complement)
    EOR #$FF
    CLC
    ADC #1
; Or equivalently:
    EOR #$FF
    SEC
    ADC #0          ; carry acts as the +1
```

### Replace CMP #0 with Nothing

LDA, AND, ORA, EOR, TAX, TAY, TXA, TYA, INX, DEX, etc. all set the Z and N flags.
You often don't need a CMP:

```asm
; Before:
    LDA value
    CMP #0
    BEQ is_zero

; After (saves 2 bytes, 2 cycles):
    LDA value
    BEQ is_zero     ; LDA already set Z flag
```

### Use DEC/BNE as a Combined Decrement-and-Test

```asm
    DEC counter     ; decrement AND set Z flag
    BNE not_zero    ; branch if result wasn't zero
```

## Size vs Speed Tradeoffs

| Technique | Saves Time | Costs Space |
|-----------|-----------|-------------|
| Unrolled loops | ~50% cycle reduction | 2-8× code size |
| Lookup tables | Replaces computation | 256-1024 bytes ROM |
| Self-modifying code | Removes indirection | Only works in RAM |
| Inlined subroutines | Saves 12 cycles/call | Duplicated code |
| Zero page variables | 1 cycle per access | Scarce ZP slots |

For NES/game development, speed usually wins in rendering code (executed
every frame) while size wins in game logic (executed occasionally).
For C64/Apple II demos, cycle-exact timing often requires both.

## Timing-Critical Code

For raster effects and cycle-exact work, you need to count every cycle.
Use NOPs and branch tricks to burn precise numbers of cycles:

```asm
; Burn exactly 2 cycles
    NOP

; Burn exactly 3 cycles (branch always taken, same page)
    BIT $00         ; 3 cycles (reads ZP, result ignored)
; or
    CLC             ; 2 cycles
    NOP             ; 2 cycles  = 4 total
; Need 5 cycles? NOP NOP + half? Use a dummy zero-page read:
    LDA $00         ; 3 cycles
    NOP             ; 2 cycles  = 5 total
```

For the NES, you have exactly 341 PPU dots per scanline (113.667 CPU cycles
at NTSC). Getting effects lined up requires counting every cycle through
your rendering loop.
