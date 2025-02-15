## Quick summary

|Instruction|General theme|Writemask|Optional special features|
|---|---|---|---|
|`mac16`&nbsp;(63=0)|`z[j][i] += x[i] * y[j]`|7 bit X, 7 bit Y|X/Y/Z input disable, right shift|
|`mac16`&nbsp;(63=1)|<code>z[_][i]&nbsp;+=&nbsp;x[i]&nbsp;*&nbsp;y[i]</code>|7 bit|X/Y/Z input disable, right shift|


## Instruction encoding

|Bit|Width|Meaning|Notes|
|---:|---:|---|---|
|10|22|[A64 reserved instruction](aarch64.md)|Must be `0x201000 >> 10`|
|5|5|Instruction|Must be `14`|
|0|5|5-bit GPR index|See below for the meaning of the 64 bits in the GPR|

## Operand bitfields

|Bit|Width|Meaning|Notes|
|---:|---:|---|---|
|63|1|Vector mode (`1`) or matrix mode (`0`)|
|62|1|Z is i32 (`1`) or Z is i16 (`0`) |Ignored in vector mode; Z is always i16 there|
|61|1|X is i8 (`1`) or X is i16 (`0`)|
|60|1|Y is i8 (`1`) or Y is i16 (`0`)|
|55|5|Right shift amount|Applied to `x*y`. When zero, sign of `x` and `y` inputs is less relevant.|
|48|7|Ignored|
|46|2|X enable mode|
|41|5|X enable value|Meaning dependent upon associated mode|
|39|2|Ignored|
|37|2|Y enable mode|Ignored in vector mode|
|32|5|Y enable value|Ignored in vector mode<br/>Meaning dependent upon associated mode|
|30|2|Ignored|
|29|1|Skip X input (`1`) or use X input (`0`)|
|28|1|Skip Y input (`1`) or use Y input (`0`)|
|27|1|Skip Z input (`1`) or use Z input (`0`)|
|26|1|Ignored|
|20|6|Z row|High bits ignored in matrix mode|
|19|1|Ignored|
|10|9|X offset|
|9|1|Ignored|
|0|9|Y offset|

Combinations of bits 27-29 result in various integer ALU operations:

|Operation|29 (X)|28 (Y)|27 (Z)|
|---|---|---|---|
|`z+((x*y)>>s)`|`0`|`0`|`0`|
|<code>&nbsp;&nbsp;&nbsp;(x*y)>>s&nbsp;</code>|`0`|`0`|`1`|
|<code>z+(&nbsp;x&nbsp;&nbsp;&nbsp;>>s)</code>|`0`|`1`|`0`|
|<code>&nbsp;&nbsp;&nbsp;&nbsp;x&nbsp;&nbsp;&nbsp;>>s&nbsp;</code>|`0`|`1`|`1`|
|<code>z+(&nbsp;&nbsp;&nbsp;y&nbsp;>>s)</code>|`1`|`0`|`0`|
|<code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;y&nbsp;>>s&nbsp;</code>|`1`|`0`|`1`|
|<code>z&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code>|`1`|`1`|`0`|
|`0`|`1`|`1`|`1`|

Combinations of bits 60-63 result in various widths / signs for X / Y / Z:

|Mode|X|Y|Z|63 (M)|62 (Z)|61 (X)|60 (Y)|
|---|---|---|---|---|---|---|---|
|Matrix|i16 <sup>[a]</sup>|i16 <sup>[a]</sup>|i16 or u16 (one row from each two)|`0`|`0`|`0`|`0`|
|Matrix|i16 <sup>[a]</sup>|i8 (even lanes)|i16 or u16 (one row from each two)|`0`|`0`|`0`|`1`|
|Matrix|i8 (even lanes)|i16 <sup>[a]</sup>|i16 or u16 (one row from each two)|`0`|`0`|`1`|`0`|
|Matrix|i8 (even lanes)|i8 (even lanes)|i16 or u16 (one row from each two)|`0`|`0`|`1`|`1`|
|Matrix|i16|i16|i32 or u32 (all rows, interleaved pairs)|`0`|`1`|`0`|`0`|
|Matrix|i16|i8 (even lanes)|i32 or u32 (all rows, interleaved pairs)|`0`|`1`|`0`|`1`|
|Matrix|i8 (even lanes)|i16|i32 or u32 (all rows, interleaved pairs)|`0`|`1`|`1`|`0`|
|Matrix|i8 (even lanes)|i8 (even lanes)|i32 or u32 (all rows, interleaved pairs)|`0`|`1`|`1`|`1`|
|Vector|i16 <sup>[a]</sup>|i16 <sup>[a]</sup>|i16 or u16 (one row)|`1`||`0`|`0`|
|Vector|i16 <sup>[a]</sup>|i8 (even lanes)|i16 or u16 (one row)|`1`||`0`|`1`|
|Vector|i8 (even lanes)|i16 <sup>[a]</sup>|i16 or u16 (one row)|`1`||`1`|`0`|
|Vector|i8 (even lanes)|i8 (even lanes)|i16 or u16 (one row)|`1`||`1`|`1`|

<sup>[a]</sup> Or u16 if shift amount is zero.

X/Y enable modes:
|Mode|Meaning of value (N)|
|---:|---|
|`0`|Enable all lanes (`0`), or odd lanes only (`1`), or even lanes only (`2`), or no lanes (anything else) |
|`1`|Only enable lane #N|
|`2`|Only enable the first N lanes, or all lanes when N is zero|
|`3`|Only enable the last N lanes, or all lanes when N is zero|

## Description

In vector mode, takes an X vector of type i8 or i16, a Y vector of type i8 or i16, and a Z vector of type i16, and performs pointwise: multiply X by Y, right shift (truncating) by some amount, then add on to Z. Variants of this pointwise operation remove the X and/or Y and/or Z inputs. When X or Y have type i8, the 8 bits are taken from the low 8 bits of each 16-bit lane.

In matrix mode, takes an X vector of type i8 or i16, a Y vector of type i8 or i16, and a 2D grid of Z values of type i16 or i32, and performs an outer product of X and Y followed by pointwise right shift (truncating) by some amount, and then pointwise addition onto Z. Variants of this remove the X and/or Y and/or Z inputs. When X or Y have type i8, the 8 bits are taken from the low 8 bits of each 16-bit lane. When Z has type i32, the entire 64x64 byte grid of Z is used, with even lanes of X going into even Z registers and odd lanes of X going into odd Z registers (see [Mixed lane widths](RegisterFile.md#mixed-lane-widths)).

## Emulation code

See [mac16.c](mac16.c).

A representative sample is:
```c
void emulate_AMX_MAC16(amx_state* state, uint64_t operand) {
    uint64_t y_offset = operand & 0x1FF;
    uint64_t x_offset = (operand >> 10) & 0x1FF;
    uint64_t z_row = (operand >> 20) & 63;
    uint64_t x_enable = parse_writemask(operand >> 41, 2, 7);
    uint64_t y_enable = parse_writemask(operand >> 32, 2, 7);

    int16_t x[32];
    int16_t y[32];
    load_xy_reg(x, state->x, x_offset);
    load_xy_reg(y, state->y, y_offset);

    for (int i = 0; i < 32; i++) {
        if (!((x_enable >> (i * 2)) & 1)) continue;
        if (operand & FMA_VECTOR_PRODUCT) {
            int16_t* z = &state->z[z_row].i16[i];
            *z = mac32_alu(x[i], y[i], *z, operand);
        } else {
            for (int j = 0; j < 32; j++) {
                if (!((y_enable >> (j * 2)) & 1)) continue;
                if (operand & FMA_WIDEN_16_32) {
                    int32_t* z = &state->z[(j * 2) + (i & 1)].i32[i >> 1];
                    *z = mac32_alu(x[i], y[j], *z, operand);
                } else {
                    int16_t* z = &state->z[(j * 2) + (z_row & 1)].i16[i];
                    *z = mac32_alu(x[i], y[j], *z, operand);
                }
            }
        }
    }
}

int64_t mac32_alu(int64_t x, int64_t y, int64_t z, uint64_t operand) {
    if (operand & MAC_X_INT8) x = (int8_t)x;
    if (operand & MAC_Y_INT8) y = (int8_t)y;
    int64_t val;
    switch ((operand >> 28) & 3) {
    default: val = x * y; break;
    case  1: val = x; break;
    case  2: val = y; break;
    case  3: val = 0; break;
    }
    uint32_t shift = (operand >> 55) & 0x1f;
    val >>= shift;
    if (!(operand & MAC_SKIP_Z_INPUT)) {
        val += z;
    }
    return val;
}
```
