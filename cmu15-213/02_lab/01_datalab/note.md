# datalab

> 参考

```text
1. 
2. 
3.
4. 
```



## 1. botXor

```text
bitXor - x^y using only ~ and & 
Example: 
	bitXor(4, 5) = 1
	Legal ops: ~ &
	Max ops: 14
	Rating: 1
```

$$
\begin{equation}
\begin{aligned}
\mathrm{a} \oplus \mathrm{b} 
&= \mathrm{a}\overline{\mathrm{b}} + \overline{\mathrm{a}}\mathrm{b} \\
&= \mathrm{a}\overline{\mathrm{a}} + \mathrm{a}\overline{\mathrm{b}} + \overline{\mathrm{a}}\mathrm{b} + \mathrm{b}\overline{\mathrm{b}} \\
&= (\mathrm{a} + \mathrm{b})(\overline{\mathrm{a}}+\overline{\mathrm{b}}) \\
&= \overline{(\overline{\mathrm{a}}\overline{\mathrm{b}})} (\overline{\mathrm{a}\mathrm{b}})
\end{aligned}
\end{equation}
$$

```c
int bitXor(int x, int y) {
  return ~(~x&~y)&(~(x&y));
}
```



## 2. tmin

```text
tmin - return minimum two's complement integer 
Legal ops: ! ~ & ^ | + << >>
Max ops: 4
Rating: 1
10000000000000000000000000000000
```

```c
int tmin(void) {
    return (1 << 31);
}
```



## 3. isTmax

```text
isTmax - returns 1 if x is the maximum, two's complement number, and 0 otherwise 
Legal ops: ! ~ & ^ | +
Max ops: 10
Rating: 1
```

```text
a = 0111111  ->  1

a + 1 = 10000000
~(a + 1) = 01111111
a ^(~(a + 1)) == 0 ? 1 : 0  -->  !(a ^ (~(a + 1)))

例外:     a = 0xFF
         a + 1 = 0x00 yichu
         ~(a + 1) = 0xFF
         !!((a + 1) ^ 0x0)
         
!(a ^(~(a + 1))) & (!!((a + 1) ^ 0x0))
```



## 4. allOddBits

```text
llOddBits - return 1 if all odd-numbered bits in word set to 1 where bits are numbered from 0 (least significant) to 31 (most significant)
xamples allOddBits(0xFFFFFFFD) = 0, allOddBits(0xAAAAAAAA) = 1
Legal ops: ! ~ & ^ | + << >>
Max ops: 12
Rating: 2
```

```text
1010 -> A  -> 1
(num & A) ^ A != 0 -> num != A
target：get 0xAAAAAAAA
```

```c
int allOddBits(int x) {
    int mask = 0xA;
    mask = mask | (mask << 4);  // 0xAA
    mask = mask | (mask << 8);  // 0xAAAA
    mask = mask | (mask << 16); // 0xAAAAAAAA
    return !((x & mask) ^ mask);
}
```



## 5. negate

```text
negate - return -x 
Example: negate(1) = -1.
Legal ops: ! ~ & ^ | + << >>
Max ops: 5
Rating: 2
```

```text
int negate(int x) {
	return ~x + 1;
}
```



## 6. isAsciiDigit

```text
isAsciiDigit - return 1 if 0x30 <= x <= 0x39 (ASCII codes for characters '0' to '9')
Example: isAsciiDigit(0x35) = 1.
		 isAsciiDigit(0x3a) = 0.
		 isAsciiDigit(0x05) = 0.
Legal ops: ! ~ & ^ | + << >>
Max ops: 15
```

```text
0x30 = 110000
0x39 = 111001
	return cond1 & cond2
cond1 -> 前26位都是0
	x >>= 6;
	cond1 = !x
		x = 0, !x = 1
		x = !0, !x = 0
cond2 -> 11xxxx
	x >>= 4;
	(x ^ 0b11) == 0 ? ! : 0;
cond3 -> xxxx <= 1001
	int c = x & 0xF;
	d = c - A < 0;   
	!!(d >>= 31)
```



## 7. conditional

```text
conditional - same as x ? y : z 
Example: conditional(2,4,5) = 4
Legal ops: ! ~ & ^ | + << >>
Max ops: 16
```

```text
return (mask & y) | (mask & z)   通过mask每次选取y或z中的一个  mask = 全1或者全0即可

取的全1：1111
a = 1 << 31;
b = a >> 31; ->  b = 111111, 这里 1 如何获取？
------------------------------------------

a = !!x
b = a << 31 >> 31;
x = 0 时：
	b = 0
x = 1 时：
	b = 11...111
```



## 8. isLessOrEqual

```text
isLessOrEqual - if x <= y  then return 1, else return 0
Example: isLessOrEqual(4,5) = 1.
Legal ops: ! ~ & ^ | + << >>
```

```text
x <= y ? 1 : 0

1. x == y
int cond1 = !(x ^ y);

   signX = x >> 31 & 1;
   signY = y >> 31 & 1;
2. x+ y-
   int cond2 = (!signX) & signY;
3. x- y+
   int cond3 = signX & !signY;
4 x- y- x+ y+
	x + ~y + 1
	int cond4 = (x + ~y + 1) >> 31 & 1;

```

```c
int isLessOrEqual(int x, int y) {
  // x == y
  int cond1 = !(x ^ y); 

  int signX = (x >> 31) & 1;
  int signY = (y >> 31) & 1;

  // x+ y-
  int cond2 = !((!signX) & signY);

  // x- y+
  int cond3 = signX & (!signY);

  // x- y- x+ y+
  int cond4 = ((x + (~y) + 1) >> 31) & 0b1;
  return cond1 || (cond2 && (cond3 || cond4));
}
```



## 9. logicalNeg

```text
logicalNeg - implement the ! operator, using all of the legal operators except !
Examples: logicalNeg(3) = 0, logicalNeg(0) = 1
Legal ops: ~ & ^ | + << >>
Max ops: 12
```

```text
0和!0的区别：
 !0的数字 有正负的区别
 int a = 0111
 int -a = 1001
 int sign = a | (-a) >> 31 = 11...11
 0 的符号位永远为0
```



## 10. howManyBits

```text
howManyBits - return the minimum number of bits required to represent x in two's complement
Examples: howManyBits(12) = 5
		  howManyBits(298) = 10
		  howManyBits(-5) = 4
		  howManyBits(0)  = 1
		  howManyBits(-1) = 1
		  howManyBits(0x80000000) = 32
Legal ops: ! ~ & ^ | + << >>
Max ops: 90
```

```text
if (x == 0)
	return 1;
else {
	x > 0
		1 + highBit // 0100
	x < 0  // 11110001  ->00001110
}

int isZero = !x;

如何获取hightBit
int flag = x >> 31; // 11...11 or 00...00
x = (~flag & x) | flag & (~x)  // get highest bits
```

```c
int howManyBits(int x) {
    int isZero = !x;
    int flag = x >> 31;
    int mask = ((!!x) << 31) >> 31;     // 全0或全1
    x = (flag & (~x)) | ((~flag) & x);  // 截取需要的部分
    
    int bit_16, bit_8, bit_4, bit_2, bit_1, bit_0;
    
    bit_16 = (!((!!(x >> 16)) ^ (0x1))) << 4; // 16
    x >>= bit_16;
    
    bit_8 = (!((!!(x >> 8)) ^ (0x1))) << 3; 
    x >>= bit_8;
    
    bit_4 = (!((!!(x >> 4)) ^ (0x1))) << 2;
    x >>= bit_4;
    
    bit_2 = (!((!!(x >> 2)) ^ (0x1))) << 1;
    x >>= bit_2;
    
    bit_1 = (!((!!(x >> 1)) ^ (0x1)));
    x >>= bit_1;
    
    bit_0 = x;
    
    int res = 1 + bit_0 + bit_1 + bit_2 + bit_4 + bit_8 + bit_16;
    
    return isZero | (mask & res);
}
```



## 11. floatScale2

```text
floatScale2 - Return bit-level equivalent of expression 2*f for floating point argument f.
Both the argument and result are passed as unsigned int's, but  they are to be interpreted as the bit-level representation of single-precision floating point values.
When argument is NaN, return argument
Legal ops: Any integer/unsigned operations incl. ||, &&. also if, while
Max ops: 30
```

```c
unsigned floatScale2(unsigned uf) {
  // expr, s, frac
  unsigned s = (uf >> 31) & (0x1);
  unsigned expr = (uf >> 23) & (0xFF);
  unsigned frac = (uf & 0x7FFFFF);

  //0
  if (expr == 0 && frac ==0)
      return uf;

  // inifity or not a number
  if (expr == 0xFF)
      return uf;

  // denormalize
  if (expr == 0) {
      // E = expr - 127 = -127
      // frac
      frac <<= 1;
      return (s << 31) | frac;
  }

  // normalize
  expr++;
  // E = expr - 127
  return (s << 31) | (expr << 23) | frac;
}
```



## 12. floatFloat2Int

```text
floatFloat2Int - Return bit-level equivalent of expression (int) f for floating point argument f.
Argument is passed as unsigned int, but it is to be interpreted as the bit-level representation of a single-precision floating point value.
Anything out of range (including NaN and infinity) should return 0x80000000u.

Legal ops: Any integer/unsigned operations incl. ||, &&. also if, while

Max ops: 30
```

```c
int floatFloat2Int(unsigned uf) {
  unsigned s = (uf >> 31) & (0x1);
  unsigned expr = (uf >> 23) & (0xFF);
  unsigned frac = (uf & 0x7FFFFF);

  //0
  if (expr == 0 && frac == 0)
        return 0;
  // inifity or NaN
  if (expr == 0xFF)
        return 1 << 31;
  // denormalize
  if (expr == 0) {
      // M 0.1111 < 1
      // E = 1 - 127 = -126
      return 0;
  }

  // normalize
  int E = expr - 127;
  frac = frac | (1 << 23);
  if (E > 31) // 1.XXXX
      return 1 << 31;
  else if (E < 0) {
    return 0;    
  }


  if (E >= 23) {
      frac <<= (E -23);
  } else {
      frac >>= (23 - E);
  }

  if (s) 
      return ~frac + 1;
  return frac;
}
```



## 13. floatPower2

```text
floatPower2 - Return bit-level equivalent of the expression 2.0^x (2.0 raised to the power x) for any 32-bit integer x.

The unsigned value that is returned should have the identical bit representation as the single-precision floating-point number 2.0^x.

If the result is too small to be represented as a denorm, return  0. If too large, return +INF.
 
Legal ops: Any integer/unsigned operations incl. ||, &&. Also if, while 
Max ops: 30 
```

```text
unsigned floatPower2(int x) {
  if (x < -149) {
    return 0;
  } else if (x < -126) {
    int shift = 23 + (x + 126);
    return 1 << shift;
  } else if (x <= 127) {
    int expr = x + 127;
    return expr << 23;
  } else {
    return (0xFF) << 23;
  }
}
```

