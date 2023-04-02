---
layout: post
title:  "Basic CPU Design in Xilinx Spartan 3E FPGA"
date:   2018-8-28 21:46:39 -0700
# categories: jekyll update
---
This basic CPU runs on Spartan 3E FPGA with basic instructions. It is Von Neumann architecture. Each instruction is 32 bit. ALU(Arithmetic Logic Unit) is capable of 32-bit if operands are loaded from memory. Otherwise direct operations only support 14-bit. [My github repo](https://github.com/hasanunlu/simple_cpu) has all necessary files (Design files, binary download, memory dump and example bubble sort assembly file). Unfortunately no interrupt vector support yet. But I will add that soon.

**Instruction word bit breakdown**
```
| opcode  |   i   |     A     |     B     |
|  3-bit  | 1-bit |   14-bit  |   14-bit  |
```
\
**ADD** -> unsigned add
```
{opcode, i} = {0, 0}
*A <- (*A) + (*B)
```
\
**ADDi** -> unsigned add immediate
```
{opcode, i} = {0, 1}
*A <- (*A) + B
```
\
**MUL** -> unsigned multiply
```
{opcode, i} = {7, 0}
*A <- (*A) * (*B)
```
\
**MULi** -> unsigned multiply
```
{opcode, i} = {7, 1}
*A <- (*A) * B
```
\
**NAND** -> bitwise NAND
```
{opcode, i} = {1, 0}
*A <- ~((*A) & (*B))
```
\
**NANDi** -> bitwise NAND immediate
```
{opcode, i} = {1, 1}
*A <- ~((*A) & B)
```
\
**SRL** -> shift right if the shift amount `*B` is less than 32, otherwise shift left
```
{opcode, i} = {2, 0}
*A <- (*B) < 32 ? (*A) >> (*B) : (*A) << ((*B) – 32)
```
\
**SRLi** -> Shift Right if the shift amount `B` is less than 32, otherwise Shift Left
```
{opcode, i} = {2, 1}
*A <- B < 32 ? (*A) >> B : (*A) << (B – 32)
```
\
**LT** -> if `*A` is less than `*B` then `*A` is set to 1, otherwise to 0.
```
{opcode, i} = {3, 0}
*A <- (*A) < (*B)
```
\
**LTi** -> if `*A` is less than `B` then `*A` is set to 1, otherwise to 0.
```
{opcode, i} = {3, 1}
*A <- (*A) < B
```
\
**CP** -> copy `*B` to `*A`
```
{opcode, i} = {4, 0}
*A <- *B
```
\
**CPi** -> copy `B` to `*A`
```
{opcode, i} = {4, 1}
*A <- B
```
\
**CPI** -> copy `**B` to `*A`
```
{opcode, i} = {5, 0}
*A <- **B
```
\
**CPIi** -> copy *B to **A
```
{opcode, i} = {5, 1}
**A <- *B
```
\
**BZJ** -> branch on zero
```
{opcode, i} = {6, 0}
PC <- (*B) == 0 ? (*A) : (PC+1)
```
\
**BZJi** -> jump
```
{opcode, i} = {6, 1}
PC <- (*A) + B
```
\
Bubble sort algorithm written in this assembly code: [bubble_sort.asm](https://raw.githubusercontent.com/hasanunlu/simple_cpu/master/bubble_sort.simplecpuasm.txt)

**References**
* The instruction is outlined by H. Fatih Ugurdag in one his graduate FPGA design courses.