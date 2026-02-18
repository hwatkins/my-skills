# 6502 Addressing Modes

## Table of Contents
- Overview Table
- Implied and Accumulator
- Immediate
- Zero Page Modes
- Absolute Modes
- Indirect Modes
- Relative (Branches)
- Choosing the Right Mode

## Overview Table

| Mode | Syntax | Example | Bytes | What it accesses |
|------|--------|---------|-------|------------------|
| Implied | `OPC` | `CLC` | 1 | Implicit register/flag |
| Accumulator | `OPC` | `ASL` | 1 | A register |
| Immediate | `OPC #val` | `LDA #$42` | 2 | Literal value |
| Zero Page | `OPC zp` | `LDA $80` | 2 | Memory[$00xx] |
| Zero Page,X | `OPC zp,X` | `LDA $80,X` | 2 | Memory[($00xx + X) & $FF] |
| Zero Page,Y | `OPC zp,Y` | `LDX $80,Y` | 2 | Memory[($00xx + Y) & $FF] |
| Absolute | `OPC addr` | `LDA $4400` | 3 | Memory[$xxxx] |
| Absolute,X | `OPC addr,X` | `LDA $4400,X` | 3 | Memory[$xxxx + X] |
| Absolute,Y | `OPC addr,Y` | `LDA $4400,Y` | 3 | Memory[$xxxx + Y] |
| Indirect | `OPC (addr)` | `JMP ($FFFC)` | 3 | Memory[Memory[$xxxx]] |
| (Indirect,X) | `OPC (zp,X)` | `LDA ($80,X)` | 2 | Memory[Memory[($zp+X)&$FF]] |
| (Indirect),Y | `OPC (zp),Y` | `LDA ($80),Y` | 2 | Memory[Memory[$zp] + Y] |
| Relative | `OPC offset` | `BEQ label` | 2 | PC + signed offset |

## Implied and Accumulator

No operand needed — the instruction implicitly operates on a specific register or flag.

**Implied**: CLC, SEC, CLD, SED, CLI, SEI, CLV, NOP, BRK, RTI, RTS,
TAX, TAY, TXA, TYA, TSX, TXS, PHA, PLA, PHP, PLP, INX, INY, DEX, DEY

**Accumulator**: ASL, LSR, ROL, ROR (when no operand is given, they operate on A)

```asm
    CLC             ; clear carry flag (implied)
    ASL             ; shift A left by 1 (accumulator)
    RTS             ; return from subroutine (implied)
```

## Immediate

The operand IS the value. Indicated by `#` prefix.

```asm
    LDA #$42        ; A ← $42 (the literal value 66)
    LDX #0          ; X ← 0
    AND #%11110000  ; mask low nibble (binary literal)
    CMP #'A'        ; compare A with ASCII 'A' (some assemblers)
```

Used for: loading constants, masking, comparing with known values.
Available for: LDA, LDX, LDY, ADC, SBC, AND, ORA, EOR, CMP, CPX, CPY, BIT (65C02)

## Zero Page Modes

Zero page ($0000–$00FF) is the 6502's "fast" memory. Instructions use only
a 1-byte address, saving 1 byte and 1 cycle vs absolute mode.

### Zero Page
```asm
    LDA $80         ; A ← Memory[$0080]   (3 cycles, 2 bytes)
    ; vs
    LDA $0080       ; A ← Memory[$0080]   (4 cycles, 3 bytes — absolute)
```
Most assemblers automatically choose zero page mode when the address fits in $00–$FF.

### Zero Page,X
```asm
    LDA $80,X       ; A ← Memory[($80 + X) & $FF]
```
**Wraps within zero page!** If $80 + X > $FF, it wraps around to the start
of zero page, NOT into page $01. Example: `LDA $F0,X` with X=$20 reads from
$10, not $110.

This is useful for arrays of variables in zero page:
```asm
; Zero page variable table for 4 objects:
; $40-$43: X positions
; $44-$47: Y positions
    LDX #2          ; object index 2
    LDA $40,X       ; load X position of object 2
    LDA $44,X       ; load Y position of object 2
```

### Zero Page,Y
Only available for LDX and STX. Same wrapping behavior as ZP,X.

```asm
    LDX $80,Y       ; X ← Memory[($80 + Y) & $FF]
    STX $80,Y       ; Memory[($80 + Y) & $FF] ← X
```

## Absolute Modes

Full 16-bit address. Can access any memory location.

### Absolute
```asm
    LDA $4400       ; A ← Memory[$4400]   (4 cycles, 3 bytes)
    STA $D021       ; Memory[$D021] ← A   (C64: border color register)
    JMP $C000       ; PC ← $C000
```

### Absolute,X and Absolute,Y
```asm
    LDA $4400,X     ; A ← Memory[$4400 + X]
    STA $0400,Y     ; Memory[$0400 + Y] ← A
```
**Page crossing penalty**: If adding X or Y causes the address to cross a page
boundary (e.g., $44FF + $01 = $4500), load instructions take an extra cycle.
Store instructions always take the extra cycle regardless.

These are the workhorse modes for array/table access:
```asm
; Read from a 256+ byte lookup table
    LDX offset
    LDA table,X     ; table is at an absolute address

; Write to screen memory
    LDY column
    LDA character
    STA SCREEN,Y    ; SCREEN = $0400 on C64
```

## Indirect Modes

### Indirect (JMP only)
```asm
    JMP ($FFFC)     ; PC ← 16-bit value stored at $FFFC/$FFFD
```
Reads a pointer from memory and jumps to that address.

**NMOS bug**: `JMP ($10FF)` reads low byte from $10FF and high byte from $1000
(not $1100). Never place an indirect jump pointer at a page boundary on NMOS 6502.

### (Indirect,X) — Pre-Indexed Indirect
```asm
    LDA ($80,X)     ; effective_addr = Memory[$80+X] | (Memory[$81+X] << 8)
                    ; A ← Memory[effective_addr]
```

X is added to the zero page address BEFORE reading the pointer. Both the pointer
low and high bytes are read from zero page (with wrapping).

Use case: selecting from a table of pointers in zero page.
```asm
; Jump table with pointers at $80, $82, $84
    LDX #4          ; select third entry (offset 4)
    LDA ($80,X)     ; load byte pointed to by $84/$85
```

This mode is less common than (Indirect),Y.

### (Indirect),Y — Post-Indexed Indirect
```asm
    LDA ($80),Y     ; ptr = Memory[$80] | (Memory[$81] << 8)
                    ; A ← Memory[ptr + Y]
```

The zero page pointer is read FIRST, then Y is added to form the final address.

**This is the most powerful addressing mode.** It turns any zero page pair into
a base pointer that Y can offset into. Essential for:

```asm
; Accessing a structure/array via pointer
    LDY #0
    LDA ($80),Y     ; read first byte at pointer
    LDY #3
    LDA ($80),Y     ; read byte at pointer+3

; String processing
    LDY #0
loop:
    LDA (str_ptr),Y ; read character from string
    BEQ done        ; null terminator
    JSR print_char
    INY
    BNE loop        ; up to 256 characters

; Iterating through a large array with pointer arithmetic
    LDY #0
page_loop:
    LDA (ptr),Y
    STA (dst),Y
    INY
    BNE page_loop
    INC ptr+1       ; advance pointer to next page
    INC dst+1
    DEX
    BNE page_loop
```

## Relative (Branches)

Used only by branch instructions. The operand is a signed 8-bit offset
from the address of the NEXT instruction (PC after fetching the branch).

```asm
    BEQ label       ; if Z=1, PC ← PC + signed_offset
```

Range: -128 to +127 bytes from the instruction after the branch.

In assembly, you write a label and the assembler calculates the offset:
```asm
    LDA $44
    BEQ is_zero     ; assembler calculates byte offset to is_zero
    ; ... code for non-zero case ...
    JMP continue
is_zero:
    ; ... code for zero case ...
continue:
```

## Choosing the Right Mode

**For constants**: Immediate (`LDA #$42`)

**For variables**: Zero page if possible (`LDA $80`), absolute if not (`LDA $4400`)

**For arrays with index < 256**: Absolute,X or Absolute,Y (`LDA table,X`)

**For zero page variable arrays**: Zero Page,X (`LDA $40,X`)

**For pointer-based access**: (Indirect),Y (`LDA ($80),Y`) — the go-to for
dynamic memory access, linked lists, string processing.

**For pointer tables**: (Indirect,X) (`LDA ($80,X)`) — less common, for selecting
among multiple pointers stored in zero page.

**For computed jumps**: Indirect (`JMP ($addr)`) — jump tables, dispatch.

**Performance priority**: Zero Page > Immediate > Absolute > Indexed > Indirect
(fewer cycles in that approximate order)
