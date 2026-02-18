# 6502 Common Algorithms

## Table of Contents
- Multi-Byte Arithmetic
- Multiplication
- Division
- Bit Manipulation
- Memory Operations
- Comparisons
- Jump Tables
- BCD Arithmetic
- Random Numbers

## Multi-Byte Arithmetic

### 16-Bit Addition
```asm
; result = num1 + num2 (all 16-bit, little-endian in zero page)
add16:
    CLC
    LDA num1        ; low byte
    ADC num2
    STA result
    LDA num1+1      ; high byte
    ADC num2+1      ; carry propagates from low byte add
    STA result+1
    RTS
```

### 16-Bit Subtraction
```asm
; result = num1 - num2
sub16:
    SEC
    LDA num1
    SBC num2
    STA result
    LDA num1+1
    SBC num2+1
    STA result+1
    RTS
```

### 16-Bit Increment
```asm
; Increment 16-bit value at addr/addr+1
inc16:
    INC addr
    BNE done        ; if low byte didn't wrap to 0, skip
    INC addr+1      ; only increment high byte if low wrapped
done:
    RTS
```

### 16-Bit Decrement
```asm
; Decrement 16-bit value at addr/addr+1
dec16:
    LDA addr
    BNE skip        ; if low byte isn't 0, just decrement it
    DEC addr+1      ; borrow from high byte
skip:
    DEC addr
    RTS
```

### 16-Bit Comparison
```asm
; Compare num1 (16-bit) with num2 (16-bit), unsigned
; After: C=1 and Z=0 → num1 > num2
;        C=1 and Z=1 → num1 = num2
;        C=0         → num1 < num2
cmp16:
    LDA num1+1      ; compare high bytes first
    CMP num2+1
    BNE done        ; if not equal, flags are set correctly
    LDA num1        ; high bytes equal, compare low bytes
    CMP num2
done:
    RTS
```

### 16-Bit Negate (Two's Complement)
```asm
; Negate 16-bit value at addr/addr+1
negate16:
    SEC
    LDA #0
    SBC addr
    STA addr
    LDA #0
    SBC addr+1
    STA addr+1
    RTS
```

### 16-Bit Shift Left (×2)
```asm
asl16:
    ASL addr        ; shift low byte left, bit 7 → carry
    ROL addr+1      ; rotate high byte left, carry → bit 0
    RTS
```

### 16-Bit Shift Right (÷2, unsigned)
```asm
lsr16:
    LSR addr+1      ; shift high byte right, bit 0 → carry
    ROR addr        ; rotate low byte right, carry → bit 7
    RTS
```

## Multiplication

### 8×8 → 16-Bit Multiply (Shift-and-Add)
```asm
; Multiply A × Y → result (16-bit at product/product+1)
; Destroys: A, X, Y, carry
multiply_8x8:
    STA multiplicand
    STY multiplier
    LDA #0
    STA product
    STA product+1
    LDX #8          ; 8 bits to process
mult_loop:
    LSR multiplier  ; shift multiplier right, bit 0 → carry
    BCC no_add      ; if bit was 0, skip addition
    CLC
    LDA product
    ADC multiplicand
    STA product
    BCC no_add
    INC product+1   ; propagate carry to high byte
no_add:
    ASL multiplicand ; shift multiplicand left (double it)
    DEX
    BNE mult_loop
    RTS             ; result in product/product+1
```

### Multiply by Constant (Shift-and-Add Inline)
```asm
; A × 10 (for decimal-to-binary conversion)
; A×10 = A×8 + A×2
mul10:
    ASL             ; A×2
    STA temp
    ASL             ; A×4
    ASL             ; A×8
    CLC
    ADC temp        ; A×8 + A×2 = A×10
    RTS

; A × 40 (e.g., 40-column screen row offset)
; A×40 = A×32 + A×8
mul40:
    STA temp        ; save original
    ASL             ; ×2
    ASL             ; ×4
    ASL             ; ×8
    STA temp2       ; save ×8
    ASL             ; ×16
    ASL             ; ×32
    CLC
    ADC temp2       ; ×32 + ×8 = ×40
    RTS
```

### Table-Based Multiply (Fast, Memory-Intensive)

Using the identity: `a*b = ((a+b)² - (a-b)²) / 4`

Store a table of squares/4 and do two lookups + subtraction.
This trades ~512 bytes of ROM for much faster multiplication.

```asm
; Requires: sqr_lo/sqr_hi tables (256 bytes each)
; where sqr[n] = (n*n)/4
; Multiply: A × X → result (16-bit)
fast_multiply:
    STA sm1         ; self-modify: factor 1
    STX sm2         ; self-modify: factor 2
    ; ... (complex setup, consult platform-specific implementations)
```

## Division

### 8-Bit Divide (A ÷ Y → Quotient in A, Remainder in X)
```asm
; Unsigned A ÷ Y → A=quotient, X=remainder
divide_8:
    STY divisor
    LDX #0          ; remainder
    LDY #8          ; bit counter
div_loop:
    ASL             ; shift dividend left, MSB → carry
    TXA
    ROL             ; shift carry into remainder
    TAX
    CMP divisor     ; remainder >= divisor?
    BCC div_skip
    SBC divisor     ; subtract divisor from remainder
    TAX
div_skip:
    DEY
    BNE div_loop
    ; A = quotient (accumulated in the shifted-out bits)
    ; Actually: quotient bits are shifted into A's low bits
    RTS
```

A simpler (slower) approach — repeated subtraction:
```asm
; Simple divide: A ÷ divisor → A=quotient, remainder in temp
simple_divide:
    LDX #0          ; quotient counter
    SEC
sub_loop:
    SBC divisor
    BCC div_done    ; went negative, stop
    INX             ; count successful subtractions
    BCS sub_loop    ; always branch (C is set by SBC if positive)
div_done:
    ADC divisor     ; undo last subtraction to get remainder
    STA temp        ; remainder in temp
    TXA             ; quotient in A
    RTS
```

## Bit Manipulation

### Test a Specific Bit
```asm
; Test bit 3 of memory location
    LDA value
    AND #%00001000  ; mask bit 3
    BNE bit_is_set

; Using BIT (preserves A, tests bits 7 and 6 specially)
    LDA #%00001000  ; mask for bit 3
    BIT value       ; Z flag = !(A & value)
    BEQ bit_is_clear
```

### Set / Clear / Toggle a Bit
```asm
; Set bit 5
    LDA value
    ORA #%00100000
    STA value

; Clear bit 5
    LDA value
    AND #%11011111  ; inverse of bit 5 mask
    STA value

; Toggle bit 5
    LDA value
    EOR #%00100000
    STA value
```

### Count Set Bits (Popcount)
```asm
; Count bits set in A → result in X
popcount:
    LDX #0
    TAY             ; save in Y for the loop
pop_loop:
    TYA
    BEQ pop_done
    SEC
    SBC #1
    AND Y           ; clear lowest set bit (n & (n-1))
    TAY
    INX
    JMP pop_loop
pop_done:
    RTS             ; count in X
```

## Memory Operations

### Fill Memory Block
```asm
; Fill 256 bytes at page_addr with value in A
fill_page:
    LDY #0
fill_loop:
    STA (dest),Y
    INY
    BNE fill_loop
    RTS

; Fill arbitrary count (count = 16-bit length)
fill_block:
    LDY #0
    LDX count+1     ; number of full pages
    BEQ partial      ; no full pages
full_pages:
    STA (dest),Y
    INY
    BNE full_pages
    INC dest+1       ; next page
    DEX
    BNE full_pages
partial:
    LDX count        ; remaining bytes
    BEQ done
partial_loop:
    STA (dest),Y
    INY
    DEX
    BNE partial_loop
done:
    RTS
```

### Copy Memory Block
```asm
; Copy count bytes from src to dest (forward, ascending)
memcpy:
    LDY #0
    LDX count+1     ; full pages
    BEQ copy_partial
copy_page:
    LDA (src),Y
    STA (dest),Y
    INY
    BNE copy_page
    INC src+1
    INC dest+1
    DEX
    BNE copy_page
copy_partial:
    LDX count       ; remaining bytes
    BEQ copy_done
copy_rem:
    LDA (src),Y
    STA (dest),Y
    INY
    DEX
    BNE copy_rem
copy_done:
    RTS
```

## Jump Tables

### Indexed Jump Table (Dispatch)
```asm
; Jump to handler based on command in A (0-N)
dispatch:
    ASL             ; multiply by 2 (table has 16-bit addresses)
    TAX
    LDA jump_table,X
    STA ptr
    LDA jump_table+1,X
    STA ptr+1
    JMP (ptr)       ; indirect jump to handler

jump_table:
    .word handler_0
    .word handler_1
    .word handler_2
    .word handler_3
```

### RTS Trick (Push Address, RTS)
A common trick: push the target address minus 1 onto the stack, then RTS.
RTS pops the address and adds 1, effectively jumping there.

```asm
; Jump to handler based on command in A
dispatch_rts:
    ASL
    TAX
    LDA jump_table+1,X  ; high byte first (stack order)
    PHA
    LDA jump_table,X    ; low byte
    PHA
    RTS                  ; "returns" to the pushed address

; Table stores address-1 of each handler
jump_table:
    .word handler_0-1
    .word handler_1-1
    .word handler_2-1
```

This avoids needing a zero page pointer for JMP (indirect).

## BCD Arithmetic

Binary-Coded Decimal mode makes ADC/SBC operate in decimal. Each nibble
represents a digit 0–9.

```asm
; Add two BCD numbers
    SED             ; set decimal mode
    CLC
    LDA #$15        ; represents decimal 15
    ADC #$27        ; represents decimal 27
    ; A = $42       ; represents decimal 42 (correct!)
    CLD             ; ALWAYS clear decimal mode when done

; Note: On NES (2A03), decimal mode is disabled in hardware.
; The D flag can be set but ADC/SBC still operate in binary.
```

## Random Numbers

### Linear Feedback Shift Register (LFSR)
```asm
; 8-bit LFSR pseudo-random number generator
; Returns random byte in A. Call repeatedly.
; seed must be non-zero, stored in zero page.
random:
    LDA seed
    ASL
    BCC no_eor
    EOR #$1D        ; polynomial for maximal-length 8-bit LFSR
no_eor:
    STA seed
    RTS

; 16-bit Galois LFSR (better period: 65535)
random16:
    LDA seed
    ASL
    ROL seed+1
    BCC no_eor16
    ; EOR with $002D for maximal 16-bit LFSR
    EOR #$2D
no_eor16:
    STA seed
    RTS             ; random byte in A, full state in seed/seed+1
```
