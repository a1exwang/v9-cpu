# Instruction Set for v9

## Registers

| Name | Usage | Width | Index |
| :---:| :-----| :----:| :----:|
| T0 | general purpose (caller saved) | 32 bits | 000 |
| T1 | general purpose (caller saved) | 32 bits | 001 |
| T2 | general purpose (caller saved) | 32 bits | 010 |
| S0 | general purpose (callee saved) | 32 bits | 011 |
| S1 | general purpose (callee saved) | 32 bits | 100 |
| FP | frame pointer | 32 bits | 101 |
| SP | stack pointer | 32 bits | 110 |
| PC | program counter | 32 bits | 111 |
| F0 | floating-point numeric computation | 64 bits ||
| F1 | floating-point numeric computation | 64 bits ||
| FLAGS | flag bits | 32 bits ||

Notice that floating-point registers have 64 bits, i.e, they are represented in type `double`. All `float` values will be represented in type `double`.

## Axioms

- Memory must be accessed by **byte**, following **small** endian.
- Stack is 8-byte-aligned, and a 4-byte value is located at the lower four bytes.
- We use ra, rb, rc to represent register T0, T1, T2, S0, S1, FP, SP or PC.
- We use fa, fb, fc to represent register F0 or F1.

## Instruction Format

```
+----------+--------+--------+--------+-------+
|  31..25  | 24..22 | 21..19 | 18..16 | 15..0 |
+----------+--------+--------+--------+-------+
|  op_code |   ra   |   rb   |   rc   |  imm  |
+----------+--------+--------+--------+-------+
```

## Instructions

### Instruction Control

| Name | Machine Code | Meaning |
| :----- | :----------- | :-------- |
| NOP | 0000000 000 000 000 000000000000000 | do nothing |

### Arithmetic/Logic

| Name | Machine Code | Meaning |
| :----- | :----------- | :-------- |
| ADD  | 0000001 ra rb rc ... | ra := rb + rc |
| ADDI | 0000010 ra rb ... imm | ra := rb + imm |
| SUB  | 0000101 ra rb rc ... | ra := rb - rc |
| SUBI | 0000110 ra rb ... imm | ra := rb - imm |
| MUL  | 0000111 ra rb rc ... | ra := rb * rc |
| MULI | 0001000 ra rb ... imm | ra := rb * imm |
| DIV  | 0001011 ra rb rc ... | ra := rb / rc |
| DIVI | 0001100 ra rb ... imm | ra := rb / imm |
| MOD  | 0001111 ra rb rc ... | ra := rb % rc |
| MODI | 0010000 ra rb ... imm | ra := rb % imm |
| | | |
| Name | Machine Code | Meaning |
| :----- | :----------- | :-------- |
| AND  | 0010011 ra rb rc ... | ra := rb and rc |
| OR   | 0010101 ra rb rc ... | ra := rb or rc |
| XOR  | 0010111 ra rb rc ... | ra := rb xor rc |
| NOT  | 0011000 ra rb ... ... | ra := not rb |
| SHL  | 0011001 ra rb rc ... | ra := unsigned(rb) << unsigned(rc) |
| SLR  | 0011011 ra rb rc ... | ra := unsigned(rb) >> unsigned(rc) |
| SAR  | 0011101 ra rb rc ... | ra := rb >> rc |
| EQ   | 0011111 ra rb rc ... | ra := if rb = rc then 1 else 0 |
| NE   | 0100001 ra rb rc ... | ra := if rb != rc then 1 else 0 |
| LT   | 0100011 ra rb rc ... | ra := if rb < rb then 1 else 0 |
| GT   | 0100111 ra rb rc ... | ra := if rb > rb then 1 else 0 |
| LE   | 0101011 ra rb rc ... | ra := if rb <= rb then 1 else 0 |
| GE   | 0101111 ra rb rc ... | ra := if rb >= rb then 1 else 0 |

### Branch/Jump

| Name | Machine Code | Meaning |
| :----- | :----------- | :-------- |
| BNZ  | 1010010 ra ... ... imm | if ra != 0 then PC := PC + offset(imm) |
| BE   | 1010100 ra rb ... imm | if ra = rb then PC := PC + offset(imm) |
| BNE  | 1010110 ra rb ... imm | if ra != rb then PC := PC + offset(imm) |
| BLT  | 1011000 ra rb ... imm | if ra .< rb then PC := PC + offset(imm) |
| BGT  | 1011011 ra rb ... imm | if ra > rb then PC := PC + offset(imm) |
| | | |
| Name | Machine Code | Meaning |
| :----- | :----------- | :-------- |
| J    | 1011110 ... ... ... imm | PC := PC + offset(imm) |
| JR   | 1011111 ra ... ... ... | PC := PC + ra |
| CALL | 1100010 000 ... ... imm | ? PC := PC + offset(imm) |
| RET  | 1100010 001 ... ... ... | ? |

### Load/Store

| Name | Machine Code | Meaning |
| :----- | :----------- | :-------- |
| LW  | 1100011 ra rb ... imm | ra := word(rb + imm) |
| LI  | 1101011 ra ... ... imm | ra := imm |
| LIU | 1101100 ra ... ... imm | ra := unsigned(imm) |
| LIH | 1101101 ra ... ... imm | ra := imm << 16 |
| SW  | 1110000 ra rb ... imm | word(rb + imm) := ra |

### Notations

| Notation | Meaning |
| :------- | :-------- |
| \_ := \_ | assign value of the right operand to the left operand (a register or a memory space) |
| \_ + \_ | add signed integers |
| unsigned(\_) + unsigned(\_) | add unsigned integers |
| \_ - \_ | subtract signed integers |
| \_ * \_ | multiply signed integers |
| unsigned(\_) * unsigned(\_) | multiply unsigned integers |
| \_ / \_ | divide signed integers |
| unsigned(\_) / unsigned(\_) | divide unsigned integers |
| \_ % \_ | mod signed integers |
| unsigned(\_) % unsigned(\_) | mod unsigned integers |
| \_ and \_ | bitwise and unsigned integers |
| \_ or \_ | bitwise or unsigned integers |
| \_ xor \_ | bitwise xor unsigned integers |
| not \_ | bitwise negate unsigned integer |
| unsigned(\_) << unsigned(\_) | shift left unsigned integer, the second operand must be in range [0, 32] |
| unsigned(\_) << unsigned(\_) | logically shift right unsigned integer, the second operand must be in range [0, 32] |
| \_ >> \_ | arithmetically shift right signed integer, the second operand must be in range [0, 32] |
| \_ = \_ | whether two integers are equal |
| \_ != \_ | whether two integers are not equal |
| \_ < \_ | whether the first signed integer is less than the second signed integer |
| unsigned(\_) < unsigned(\_) | whether the first unsigned integer is less than the second unsigned integer |
| \_ > \_ | whether the first signed integer is greater than the second signed integer |
| unsigned(\_) > unsigned(\_) | whether the first unsigned integer is greater than the second unsigned integer |
| \_ <= \_ | whether the first signed integer is less or equal than the second signed integer |
| unsigned(\_) <= unsigned(\_) | whether the first unsigned integer is less or equal than the second unsigned integer |
| \_ >= \_ | whether the first signed integer is greater or equal than the second signed integer |
| unsigned(\_) >= unsigned(\_) | whether the first unsigned integer is greater or equal than the second unsigned integer |
| \_ .+ \_ | add floating-point numbers |
| \_ .- \_ | subtract floating-point numbers |
| \_ .* \_ | multiply floating-point numbers |
| \_ ./ \_ | divide floating-point numbers |
| \_ .= \_ | whether two floating-point numbers are equal |
| \_ .!= \_ | whether two floating-point numbers are not equal |
| \_ .< \_ | whether the first floating-point numbers is less than the second floating-point numbers |
| \_ .> \_ | whether the first floating-point numbers is greater than the second floating-point numbers |
| \_ .<= \_ | whether the first floating-point numbers is less or equal than the second floating-point numbers |
| \_ .>= \_ | whether the first floating-point numbers is greater or equal than the second floating-point numbers |
| \_ .% \_ | mod floating-point numbers |
| float(\_) | convert signed integer into floating-point number |
| float(unsigned(\_)) | convert unsigned integer into floating-point number |
| signed(\_) | convert floating-point number into signed number |
| word(\_) | 32 bits from the starting memory address |
| half(\_) | 16 bits from the starting memory address |
| byte(\_) | 8 bits from the starting memory address |
| offset(\_) | shift left the immediate integer and then do signed extension |
|
