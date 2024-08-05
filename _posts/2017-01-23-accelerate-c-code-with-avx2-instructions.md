---
layout: post
title: Accelerate C Code with AVX2 Instructions
---

### Instruction Set Extension

Today’s modern CPU such as Intel’s Boardwell and Skylake usually has some instruction set extensions, for example, SSE2 and AVX2. These instruction sets provide complex, and usually multi-cycle instructions to make it possible for programmers to speed up their code, in the hardware level.

Among these instructions the AVX instructions may be the most useful ones. Intel AVX, the abbreviation of Intel Advanced Vector Extensions, provides CPU the ability of doing the same thing multiple times at the same time. This feature is super useful in vector operation, so it is granted the name AVX. Intel AVX is first introduced in Sandy Bridge, and it’s improved to AVX2 when Haswell Architecture is published by Intel. This extension is still improving, today’s Skylake CPU in the Xeon family has already support AVX-512, which means the operational capability is doubled then its predecessor, Haswell.

### Registers

AVX2 uses 8 `ymm` registers, which is an extension of `xmm`. Each `ymm` register has 256 bits, and the lowest 128 bits belongs to `xmm` register. As for AVX-512, `zmm` registers are introduced. Just like what `ymm` is for `xmm`, `ymm` register is the lowest 256 bits of `zmm` register.

### Instructions

There’s a lot of them, and they are all in an official guide of Intel CPU, which contains more than 3000 pages, and you can find it on [01.org](https://01.org/).
And here’s some examples:

```c
vpxor ymm, ymm, ymm // 256 bit xor
vpaddq ymm, ymm, ymm // add 4 64bit integer
vpaddd ymm, ymm, ymm // add 8 32bit integer
vpcmpeqq ymm, ymm, ymm // compare 4 64bit integer, and store the result into a ymm
vmovdqu ymm, m256 // load a in-memory value to register or store it back
```

### Bad News & Good News

Few compilers support these instructions. This result is kind of reasonable, because as a compiler you have take the compatibility into consideration, for example, you have to support different CPU from different vendor, although they have the common base, x86_64 Instruction Set.

Good news is that although compilers like GCC refuses to use these instructions to speed up your program automatically, you can force them to use it. Both of GCC and Visual C++ provide a group of headers to make programming with these new instruction easier. So there’s no need to directly write Assembly or inline Assembly anymore.

### Work with GCC

GCC has several headers which provide encapsulation of these instruction, for example `immintrin.h`, `emmintrin.h`, `xmmintrin.h` and so on, but you can only include `immintrin.h` because most other headers are included by it.
To use AVX instructions with these header is quiet simple. But before that, we have to know how to store the data in those special registers, in other word, how to represent these registers.

#### Data Type

The data types used to represent ymm should be `__m256`, `__m256i` and `__m256d`. There’s also types like `__m128`, representing `xmm`, and `__m512`, representing `zmm`.

#### Functions & Micros

Most functions and be find [here](https://software.intel.com/sites/landingpage/IntrinsicsGuide/).
Here’s some example:

```c
__m256i _mm256_loadu_si256 (__m256i const * mem_addr) // to load 256bit integer data from memory.
void _mm256_storeu_si256 (__m256i * mem_addr, __m256i a) // to store data in a register to memory.
__m256i _mm256_xor_si256 (__m256i a, __m256i b) // to calculate a xor b
```

Note that registers begins with two `_`, and functions begins with one `_`. And while programming, although the compiler will allocate registers and spill variable atomically, programmer would better decide these operation themselves, because these functions or micros are at a very low level.

### Example

```c
#include <immintrin.h>
void xor4ll(void* addr_a, void* addr_b, void* addr_target) {
    __m256i a = _mm256_loadu_si256((__m256i*)addr_a);
    __m256i b = _mm256_loadu_si256((__m256i*)addr_b);
    __m256i c = _mm256_xor_si256(a, b);
    _mm256_storeu_si256((__m256i*)addr_target, c);
}
```
