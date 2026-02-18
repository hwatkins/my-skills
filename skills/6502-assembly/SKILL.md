---
name: 6502-assembly
description: >
  Write, read, debug, and optimize 6502 assembly language code. Use this skill whenever
  the user asks about 6502 assembly, MOS 6502, 6510, 65C02, or 65C816 programming,
  or is working on projects for the NES, SNES, Commodore 64, Apple II, Atari 2600/8-bit,
  VIC-20, BBC Micro, or any other 6502-based platform. Also trigger when the user mentions
  opcodes, addressing modes, zero page, stack operations, or assembly language concepts
  in the context of 8-bit/retro computing. Covers instruction set, addressing modes,
  common algorithms (16-bit math, multiplication, division), memory layout, optimization,
  and platform-specific patterns. Even if the user just says "assembly" in a retro
  computing context, use this skill.
---

# 6502 Assembly Language

Complete reference for programming the MOS 6502 and its derivatives. Covers the
instruction set, addressing modes, common algorithms, optimization techniques, and
platform-specific considerations.

## Architecture Overview

The 6502 is an 8-bit processor with a 16-bit address bus (64KB addressable memory).

### Registers

| Register | Width | Purpose |
|----------|-------|---------|
| **A** (Accumulator) | 8-bit | Primary arithmetic/logic register |
| **X** (Index X) | 8-bit | Index register, counter, transfers |
| **Y** (Index Y) | 8-bit | Index register, counter, transfers |
| **SP** (Stack Pointer) | 8-bit | Points into page $01 ($0100–$01FF) |
| **PC** (Program Counter) | 16-bit | Address of next instruction |
| **P** (Processor Status) | 8-bit | Flag register: `NV-BDIZC` |

### Status Flags (P register): `NV-BDIZC`

| Bit | Flag | Name | Set when |
|-----|------|------|----------|
| 7 | **N** | Negative | Result bit 7 is set |
| 6 | **V** | Overflow | Signed arithmetic overflow |
| 5 | - | (Unused) | Always 1 |
| 4 | **B** | Break | BRK instruction (on stack only) |
| 3 | **D** | Decimal | BCD mode enabled (SED/CLD) |
| 2 | **I** | Interrupt Disable | IRQ disabled (SEI/CLI) |
| 1 | **Z** | Zero | Result is zero |
| 0 | **C** | Carry | Unsigned overflow / borrow inverse |

### Memory Map (General)

| Range | Purpose |
|-------|---------|
| `$0000–$00FF` | **Zero Page** — fast access, used as pseudo-registers |
| `$0100–$01FF` | **Stack** — grows downward from $01FF |
| `$0200–$FFF9` | **General RAM/ROM** — platform-specific |
| `$FFFA–$FFFB` | **NMI vector** — address of NMI handler |
| `$FFFC–$FFFD` | **RESET vector** — address of startup code |
| `$FFFE–$FFFF` | **IRQ/BRK vector** — address of IRQ handler |

Read `references/instruction-set.md` for the complete instruction reference.
Read `references/addressing-modes.md` for all 13 addressing modes with examples.
Read `references/algorithms.md` for 16-bit math, multiply, divide, and common patterns.
Read `references/optimization.md` for cycle counting and optimization techniques.
Read `references/platforms.md` for NES, C64, Apple II, and Atari specifics.

## Addressing Modes Quick Reference

The 6502 has 13 addressing modes. The mode determines where the operand comes from:

```
Implied         CLC              ; operand is implicit (carry flag)
Accumulator     ASL              ; operates on A register
Immediate       LDA #$42         ; literal value follows opcode
Zero Page       LDA $80          ; 8-bit address in page $00
Zero Page,X     LDA $80,X        ; zero page + X offset (wraps within $00)
Zero Page,Y     LDX $80,Y        ; zero page + Y offset (only LDX, STX)
Absolute        LDA $4400        ; full 16-bit address
Absolute,X      LDA $4400,X      ; 16-bit address + X
Absolute,Y      LDA $4400,Y      ; 16-bit address + Y
Indirect        JMP ($FFFC)      ; read 16-bit address from pointer (JMP only)
(Indirect,X)    LDA ($80,X)      ; zero page pointer + X pre-index
(Indirect),Y    LDA ($80),Y      ; zero page pointer, then + Y post-index
Relative        BEQ label        ; signed 8-bit offset from PC (-128 to +127)
```

**Key distinction**: `(Indirect,X)` adds X *before* reading the pointer.
`(Indirect),Y` adds Y *after* reading the pointer. The `,Y` form is far more
common and useful — it lets zero page hold a base pointer that Y indexes into.

## Essential Patterns

### CLC before ADC, SEC before SBC

The carry flag participates in every add/subtract. Always set it explicitly:

```asm
; Addition: clear carry first
    CLC
    LDA num1
    ADC num2
    STA result

; Subtraction: set carry first (carry = inverse borrow)
    SEC
    LDA num1
    SBC num2
    STA result
```

### 16-Bit Addition

Add low bytes, then high bytes. The carry propagates automatically:

```asm
    CLC
    LDA num1_lo      ; low byte of first number
    ADC num2_lo      ; add low bytes
    STA result_lo    ; store low byte result
    LDA num1_hi      ; high byte of first number
    ADC num2_hi      ; add high bytes + carry
    STA result_hi    ; store high byte result
```

### 16-Bit Subtraction

```asm
    SEC
    LDA num1_lo
    SBC num2_lo
    STA result_lo
    LDA num1_hi
    SBC num2_hi
    STA result_hi
```

### Comparison and Branching

CMP sets flags as if a subtraction occurred (but doesn't store the result):

```asm
    LDA score
    CMP #100          ; compare A with 100
    BEQ equal         ; branch if A == 100
    BCS greater_eq    ; branch if A >= 100 (unsigned)
    BCC less_than     ; branch if A < 100 (unsigned)
```

For signed comparison, check the N flag (with overflow awareness):

```asm
    LDA signed_val
    CMP #$80
    BCC positive      ; bit 7 clear = positive (0 to 127)
    ; negative path   ; bit 7 set = negative (-128 to -1)
```

### Loop Patterns

**Count down to zero** (most efficient — no CMP needed):

```asm
    LDX #10           ; loop 10 times
loop:
    ; ... loop body ...
    DEX
    BNE loop          ; DEX sets Z flag when X reaches 0
```

**Count up with comparison**:

```asm
    LDX #0
loop:
    ; ... loop body using X as index ...
    INX
    CPX #10
    BNE loop
```

**Iterate over array** (Y-indexed, indirect):

```asm
    LDY #0
loop:
    LDA (ptr),Y       ; ptr is a zero-page 16-bit pointer
    BEQ done           ; stop at null terminator
    JSR process_byte
    INY
    BNE loop           ; loop up to 256 bytes (Y wraps)
done:
```

### Subroutine Call

```asm
    JSR my_subroutine  ; push return address, jump
    ; ... continues here after RTS

my_subroutine:
    ; ... do work ...
    ; save/restore registers if needed:
    PHA                ; save A
    TXA
    PHA                ; save X
    ; ... work ...
    PLA
    TAX                ; restore X
    PLA                ; restore A
    RTS                ; pull return address, jump back
```

**Important**: JSR pushes the address of the *last byte* of the JSR instruction
(PC-1), and RTS increments it by 1. This is why RTS goes to the right place but
RTI (which doesn't add 1) is used for interrupts instead.

### Interrupt Handler Template

```asm
irq_handler:
    PHA                ; save A
    TXA
    PHA                ; save X
    TYA
    PHA                ; save Y

    ; ... handle interrupt ...

    PLA
    TAY                ; restore Y
    PLA
    TAX                ; restore X
    PLA                ; restore A
    RTI                ; return from interrupt
```

### Multiply by Small Constants

No MUL instruction exists. Use shifts and adds:

```asm
; Multiply A by 2
    ASL                ; shift left = ×2

; Multiply A by 4
    ASL
    ASL                ; ×2 × 2 = ×4

; Multiply A by 10 (for decimal conversion)
    ASL                ; A×2
    STA temp
    ASL
    ASL                ; A×8
    CLC
    ADC temp           ; A×8 + A×2 = A×10

; Multiply A by 5
    STA temp
    ASL
    ASL                ; A×4
    CLC
    ADC temp           ; A×4 + A = A×5
```

See `references/algorithms.md` for full 8×8 and 16×16 multiply routines.

### String/Memory Copy

```asm
; Copy 256 bytes from src to dst
    LDY #0
copy_loop:
    LDA (src),Y
    STA (dst),Y
    INY
    BNE copy_loop      ; loops 256 times (Y wraps 0→FF→0)
```

For copies > 256 bytes, increment the high byte of src/dst pointers:

```asm
; Copy pages (256-byte blocks)
    LDX #num_pages
    LDY #0
page_loop:
    LDA (src),Y
    STA (dst),Y
    INY
    BNE page_loop
    INC src+1          ; increment high byte of source pointer
    INC dst+1          ; increment high byte of dest pointer
    DEX
    BNE page_loop
```

## Common Gotchas

1. **Forgetting CLC/SEC.** The #1 bug. ADC always includes carry. If you forget
   CLC, a stale carry from a previous operation corrupts your math.

2. **JMP ($xxFF) bug.** On NMOS 6502, `JMP ($10FF)` reads the low byte from $10FF
   and the high byte from $1000 (not $1100). The high byte of the pointer address
   doesn't increment. Fixed on 65C02.

3. **Branch range.** Branches are relative, ±127 bytes. If you need to branch
   further, invert the condition and JMP:
   ```asm
       BEQ skip        ; can't reach far_away directly
       JMP far_away
   skip:
   ```

4. **Decimal mode.** If D flag is set (SED), ADC/SBC operate in BCD. Always CLD
   at startup. NES's 2A03 variant ignores the D flag entirely.

5. **Stack overflow.** The stack is only 256 bytes ($0100–$01FF). Deep recursion
   or unbalanced PHA/PLA will wrap around and corrupt data.

6. **Zero page pointer alignment.** Indirect addressing reads two bytes from zero
   page. A pointer at $FF wraps to $00 (not $100) for its high byte. Keep pointers
   at even addresses to avoid confusion.

7. **Read-modify-write double write.** On NMOS 6502, instructions like INC, DEC,
   ASL, LSR, ROL, ROR write the original value back before writing the modified value.
   This can trigger hardware side effects on I/O addresses. Fixed on 65C02.

## 6502 Variants

| Chip | Used in | Notable differences |
|------|---------|---------------------|
| **MOS 6502** | Apple II, Atari 800, VIC-20 | Original NMOS. JMP indirect bug. |
| **MOS 6510** | Commodore 64 | 6502 + I/O port at $00/$01 |
| **Ricoh 2A03** | NES/Famicom | 6502 without decimal mode + audio |
| **WDC 65C02** | Apple IIe enhanced, BBC Master | Fixes JMP bug, adds BRA, PHX/PLX, STZ, etc. |
| **WDC 65C816** | Apple IIGS, SNES | 16-bit extension, 24-bit address bus |
| **HuC6280** | TurboGrafx-16 | 65C02 + extra I/O, block transfer, timers |

When writing code, specify which variant you're targeting. Instructions like BRA,
PHX, PLX, PHY, PLY, STZ, TRB, TSB are 65C02+ only.

## Code Style Conventions

- **Labels**: lowercase with underscores: `game_loop`, `update_sprites`
- **Constants**: UPPERCASE: `SCREEN_WIDTH = 32`, `PPU_CTRL = $2000`
- **Hex prefix**: `$` for hex (`$FF`), `#` for immediate (`#$FF`), `%` for binary (`%10110011`)
- **Comments**: Use `;` — comment every non-obvious instruction
- **Zero page**: Document your ZP allocation at the top of the file
- **Vectors**: Always set up RESET, NMI, and IRQ vectors at the end of ROM

```asm
; --- Zero Page Variables ---
score_lo    = $00
score_hi    = $01
player_x    = $02
player_y    = $03
temp        = $04
ptr_lo      = $10      ; general-purpose pointer
ptr_hi      = $11

; --- Constants ---
SCREEN_BASE = $0400    ; C64 screen memory
COLOR_BASE  = $D800    ; C64 color memory
```
