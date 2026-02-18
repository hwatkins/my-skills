# 6502 Instruction Set Reference

## Table of Contents
- Load and Store
- Arithmetic
- Logical
- Shift and Rotate
- Increment and Decrement
- Compare and Branch
- Jump and Subroutine
- Stack
- Transfer
- Flag Control
- System
- 65C02 Extensions

## Notation

- **A** = Accumulator, **X** = X register, **Y** = Y register
- **M** = Memory operand, **C** = Carry flag
- **Flags**: N=Negative, V=Overflow, Z=Zero, C=Carry
- Cycle counts marked with **+** add 1 if page boundary crossed
- Cycle counts marked with **++** add 1 if branch taken, 2 if page crossed

## Load and Store

### LDA — Load Accumulator
```
A ← M                          Flags: N Z
Immediate    LDA #$44    $A9   2 bytes  2 cycles
Zero Page    LDA $44     $A5   2 bytes  3 cycles
Zero Page,X  LDA $44,X   $B5   2 bytes  4 cycles
Absolute     LDA $4400   $AD   3 bytes  4 cycles
Absolute,X   LDA $4400,X $BD   3 bytes  4+ cycles
Absolute,Y   LDA $4400,Y $B9   3 bytes  4+ cycles
(Indirect,X) LDA ($44,X) $A1   2 bytes  6 cycles
(Indirect),Y LDA ($44),Y $B1   2 bytes  5+ cycles
```

### LDX — Load X Register
```
X ← M                          Flags: N Z
Immediate    LDX #$44    $A2   2 bytes  2 cycles
Zero Page    LDX $44     $A6   2 bytes  3 cycles
Zero Page,Y  LDX $44,Y   $B6   2 bytes  4 cycles
Absolute     LDX $4400   $AE   3 bytes  4 cycles
Absolute,Y   LDX $4400,Y $BE   3 bytes  4+ cycles
```

### LDY — Load Y Register
```
Y ← M                          Flags: N Z
Immediate    LDY #$44    $A0   2 bytes  2 cycles
Zero Page    LDY $44     $A4   2 bytes  3 cycles
Zero Page,X  LDY $44,X   $B4   2 bytes  4 cycles
Absolute     LDY $4400   $AC   3 bytes  4 cycles
Absolute,X   LDY $4400,X $BC   3 bytes  4+ cycles
```

### STA — Store Accumulator
```
M ← A                          Flags: (none)
Zero Page    STA $44     $85   2 bytes  3 cycles
Zero Page,X  STA $44,X   $95   2 bytes  4 cycles
Absolute     STA $4400   $8D   3 bytes  4 cycles
Absolute,X   STA $4400,X $9D   3 bytes  5 cycles
Absolute,Y   STA $4400,Y $99   3 bytes  5 cycles
(Indirect,X) STA ($44,X) $81   2 bytes  6 cycles
(Indirect),Y STA ($44),Y $91   2 bytes  6 cycles
```

### STX — Store X Register
```
M ← X                          Flags: (none)
Zero Page    STX $44     $86   2 bytes  3 cycles
Zero Page,Y  STX $44,Y   $96   2 bytes  4 cycles
Absolute     STX $4400   $8E   3 bytes  4 cycles
```

### STY — Store Y Register
```
M ← Y                          Flags: (none)
Zero Page    STY $44     $84   2 bytes  3 cycles
Zero Page,X  STY $44,X   $94   2 bytes  4 cycles
Absolute     STY $4400   $8C   3 bytes  4 cycles
```

## Arithmetic

### ADC — Add with Carry
```
A ← A + M + C                  Flags: N V Z C
Immediate    ADC #$44    $69   2 bytes  2 cycles
Zero Page    ADC $44     $65   2 bytes  3 cycles
Zero Page,X  ADC $44,X   $75   2 bytes  4 cycles
Absolute     ADC $4400   $6D   3 bytes  4 cycles
Absolute,X   ADC $4400,X $7D   3 bytes  4+ cycles
Absolute,Y   ADC $4400,Y $79   3 bytes  4+ cycles
(Indirect,X) ADC ($44,X) $61   2 bytes  6 cycles
(Indirect),Y ADC ($44),Y $71   2 bytes  5+ cycles
```
**Always CLC before first ADC in a chain.** In decimal mode (D flag set),
operates as BCD addition.

### SBC — Subtract with Carry (Borrow)
```
A ← A - M - (1-C)              Flags: N V Z C
Immediate    SBC #$44    $E9   2 bytes  2 cycles
Zero Page    SBC $44     $E5   2 bytes  3 cycles
Zero Page,X  SBC $44,X   $F5   2 bytes  4 cycles
Absolute     SBC $4400   $ED   3 bytes  4 cycles
Absolute,X   SBC $4400,X $FD   3 bytes  4+ cycles
Absolute,Y   SBC $4400,Y $F9   3 bytes  4+ cycles
(Indirect,X) SBC ($44,X) $E1   2 bytes  6 cycles
(Indirect),Y SBC ($44),Y $F1   2 bytes  5+ cycles
```
**Always SEC before first SBC in a chain.** C=1 means no borrow. After SBC,
C=0 means a borrow occurred.

## Logical

### AND — Bitwise AND
```
A ← A & M                      Flags: N Z
Same addressing modes as ADC. Opcodes: $29, $25, $35, $2D, $3D, $39, $21, $31
```

### ORA — Bitwise OR
```
A ← A | M                      Flags: N Z
Same addressing modes as ADC. Opcodes: $09, $05, $15, $0D, $1D, $19, $01, $11
```

### EOR — Bitwise Exclusive OR
```
A ← A ^ M                      Flags: N Z
Same addressing modes as ADC. Opcodes: $49, $45, $55, $4D, $5D, $59, $41, $51
```

### BIT — Bit Test
```
Z ← !(A & M), N ← M bit 7, V ← M bit 6    Flags: N V Z
Zero Page    BIT $44     $24   2 bytes  3 cycles
Absolute     BIT $4400   $2C   3 bytes  4 cycles
```
Useful for testing specific bits without modifying A. The N and V flags reflect
bits 7 and 6 of the *memory* value, not the AND result.

## Shift and Rotate

All shift/rotate operations work on either A (accumulator mode) or memory.

### ASL — Arithmetic Shift Left
```
C ← [7][6][5][4][3][2][1][0] ← 0     Flags: N Z C
Accumulator  ASL         $0A   1 byte   2 cycles
Zero Page    ASL $44     $06   2 bytes  5 cycles
Zero Page,X  ASL $44,X   $16   2 bytes  6 cycles
Absolute     ASL $4400   $0E   3 bytes  6 cycles
Absolute,X   ASL $4400,X $1E   3 bytes  7 cycles
```
Multiplies by 2. Bit 7 goes to carry. Bit 0 becomes 0.

### LSR — Logical Shift Right
```
0 → [7][6][5][4][3][2][1][0] → C      Flags: N(=0) Z C
Same addressing modes as ASL. Opcodes: $4A, $46, $56, $4E, $5E
```
Unsigned divide by 2. Bit 0 goes to carry. Bit 7 becomes 0.

### ROL — Rotate Left through Carry
```
C ← [7][6][5][4][3][2][1][0] ← C      Flags: N Z C
Same addressing modes as ASL. Opcodes: $2A, $26, $36, $2E, $3E
```
Old carry rotates into bit 0. Bit 7 rotates into carry.

### ROR — Rotate Right through Carry
```
C → [7][6][5][4][3][2][1][0] → C       Flags: N Z C
Same addressing modes as ASL. Opcodes: $6A, $66, $76, $6E, $7E
```
Old carry rotates into bit 7. Bit 0 rotates into carry.

## Increment and Decrement

### INC / DEC — Increment / Decrement Memory
```
M ← M ± 1                      Flags: N Z
              INC   DEC
Zero Page     $E6   $C6   2 bytes  5 cycles
Zero Page,X   $F6   $D6   2 bytes  6 cycles
Absolute      $EE   $CE   3 bytes  6 cycles
Absolute,X    $FE   $DE   3 bytes  7 cycles
```

### INX / DEX — Increment / Decrement X
```
INX: X ← X + 1    $E8   1 byte  2 cycles   Flags: N Z
DEX: X ← X - 1    $CA   1 byte  2 cycles   Flags: N Z
```

### INY / DEY — Increment / Decrement Y
```
INY: Y ← Y + 1    $C8   1 byte  2 cycles   Flags: N Z
DEY: Y ← Y - 1    $88   1 byte  2 cycles   Flags: N Z
```

## Compare and Branch

### CMP / CPX / CPY — Compare
```
Sets flags based on (Register - M), result discarded.
C=1 if Register >= M, Z=1 if Register == M, N reflects bit 7 of result.
```
CMP has the same 8 addressing modes as LDA.
CPX/CPY have Immediate, Zero Page, and Absolute modes.

### Branch Instructions
All branches: 2 bytes, 2 cycles (not taken), 3 cycles (taken), 4 if page crossed.

```
BCC  $90  Branch if Carry Clear        (C=0)  — unsigned less than
BCS  $B0  Branch if Carry Set          (C=1)  — unsigned greater or equal
BEQ  $F0  Branch if Equal (Zero set)   (Z=1)
BNE  $D0  Branch if Not Equal          (Z=0)
BMI  $30  Branch if Minus (Negative)   (N=1)
BPL  $10  Branch if Plus               (N=0)
BVS  $70  Branch if Overflow Set       (V=1)
BVC  $50  Branch if Overflow Clear     (V=0)
```

Operand is a signed byte offset from the *next* instruction (-128 to +127).

## Jump and Subroutine

```
JMP $4400      $4C  3 bytes  3 cycles  — Absolute jump
JMP ($4400)    $6C  3 bytes  5 cycles  — Indirect jump (BUG: see below)
JSR $4400      $20  3 bytes  6 cycles  — Jump to Subroutine
RTS            $60  1 byte   6 cycles  — Return from Subroutine
RTI            $40  1 byte   6 cycles  — Return from Interrupt
```

**JMP Indirect Bug (NMOS 6502 only)**: If the pointer address is $xxFF,
the high byte is fetched from $xx00 instead of $(xx+1)00. Example:
`JMP ($10FF)` reads low from $10FF, high from $1000. Fixed on 65C02.

**JSR/RTS detail**: JSR pushes PC-1 (the address of the last byte of the JSR
instruction). RTS pulls the address and adds 1. This means the return goes to
the byte *after* the JSR.

## Stack Instructions

The stack lives at $0100–$01FF and grows downward (SP decrements on push).

```
PHA  $48  Push A to stack              1 byte  3 cycles
PLA  $68  Pull A from stack            1 byte  4 cycles  Flags: N Z
PHP  $08  Push Processor Status        1 byte  3 cycles
PLP  $28  Pull Processor Status        1 byte  4 cycles  Flags: all
```

## Transfer Instructions

```
TAX  $AA  A → X    Flags: N Z     TAY  $A8  A → Y    Flags: N Z
TXA  $8A  X → A    Flags: N Z     TYA  $98  Y → A    Flags: N Z
TSX  $BA  SP → X   Flags: N Z     TXS  $9A  X → SP   Flags: (none)
```

Note: TXS does NOT affect flags. TSX does.

## Flag Instructions

```
CLC  $18  Clear Carry         SEC  $38  Set Carry
CLD  $D8  Clear Decimal       SED  $F8  Set Decimal
CLI  $58  Clear Interrupt     SEI  $78  Set Interrupt Disable
CLV  $B8  Clear Overflow      (no SOV instruction)
```

## System

```
BRK  $00  Force Interrupt     1 byte*  7 cycles
NOP  $EA  No Operation        1 byte   2 cycles
```

*BRK is technically 1 byte but the byte after it is skipped (PC pushed is BRK+2).
This allows BRK to replace a 2-byte instruction for debugging.

## 65C02 Extensions

These instructions are available on the WDC 65C02 and later, NOT on the
original NMOS 6502 or 6510:

```
BRA label    $80  Branch Always (unconditional relative branch)
PHX          $DA  Push X             PLX  $FA  Pull X
PHY          $5A  Push Y             PLY  $7A  Pull Y
STZ addr     $64/$74/$9C/$9E  Store Zero to memory
TRB addr     $14/$1C  Test and Reset Bits
TSB addr     $04/$0C  Test and Set Bits
INC          $1A  Increment A       DEC  $3A  Decrement A
```

Also: `BIT #imm`, `BIT zp,X`, `BIT abs,X` addressing modes added.
JMP indirect bug is fixed.
BBR/BBS (bit branch) instructions available on Rockwell/WDC 65C02.
