---
layout: post
title: Comparison of 3d Math libraries
---

In this article I want to compare performance of basic operations of different popular (and less popular) math libraries. I will focus on stuff related to 3d math used in geometry processing, so I tested Vector4 and 4x4 Matrix implementations. For GLM and Mango library I also tested swizzle implementations as this libraries have this feature.

I tested presented libraries on 3 major compilers (MSVC, Gcc, Clang) using google benchmark. Benchmark results are quite interesting as it tells us which library is most performant, however dissassemblies are most interesting because we can see how each implementation compiles on different compilers. Important thing to note: MSVC and Clang tests were run on Windows 10, and Gcc tests on Ubuntu 20.04, it seems that on GCC results are very strange on some tests and I will explain that later, but it seems that MSVC and Clang results are comparable and sometimes GCC results are strangely biased and benchmark timings are unrealistic and should be taken with big grain of salt.

For benchmark I took GLM, Mathfu and Mango ad typical 3d game math libraries and Eigen and Blaze as state of the art general purpose math libraries. GLM library is tested in two modes "out of the box" and SIMD configured mode. for other libraries I'm not very familiar with them so I took default settings. Tests were made on two processor I had access to: old Xeon E5450 (SSE 4.1 capabilities) and i7-8850H, on second one I also run benchmarks compiled with AVX2 instrucions where is was possible.

# Vector4 tests

I set up few tests on which I will do the comparisons:
    
1. Multiply and multiply by scalar tests:
   
   ```c++
    for (auto _ : state) {
        benchmark::ClobberMemory();
        res = testData[0] * testData[1];
        benchmark::ClobberMemory();
    }
    ```
    
    ```c++    
    for (auto _ : state) {
        benchmark::ClobberMemory();
        res = testData[0] * testData[1].y;
        benchmark::ClobberMemory();
    }
    ```   
    
I will combine anazying these tests as they are fairly similar. 
    
Benchmark results:  

| Xeon E8450      | MSVC       | GCC        | CLANG      | i7 8850H        | MSVC      | GCC        | CLANG     |
| ----------------| ---------- | ---------- | ---------- | --------------- | --------- | ---------- | --------- |
| GLM             | 5.05 ns    | 3.02 ns    | 1.86 ns    | GLM             | 2.24 ns   | -          | 0.510 ns  |
| GLM SIMD        | 1.08 ns    | 1.00 ns    | 1.01 ns    | GLM SIMD        | 0.372 ns  | -          | 0.412 ns  |
| Eigen           | 1.02 ns    | 1.00 ns    | 1.02 ns    | Eigen           | 0.506 ns  | -          | 0.414 ns  |
| Blaze           | 1.43 ns    | 1.00 ns    | 1.02 ns    | Blaze           | 0.513 ns  | -          | 0.499 ns  |
| Mathfu          | 2.72 ns    | 1.00 ns    | 1.02 ns    | Mathfu          | 1.75 ns   | -          | 0.425 ns  |
| Mango           | 1.03 ns    | 1.00 ns    | 1.01 ns    | Mango           | 0.498 ns  | -          | 0.413 ns  | 

| Xeon E8450      | MSVC       | GCC        | CLANG      | i7 8850H        | MSVC      | GCC        | CLANG     |
| ----------------| ---------- | ---------- | ---------- | --------------- | --------- | ---------- | --------- |
| GLM             | 3.98 ns    | 2.01 ns    | 1.40 ns    | GLM             | 1.50 ns   | -          | 0.514 ns  |
| GLM SIMD        | 1.02 ns    | 1.00 ns    | 1.01 ns    | GLM SIMD        | 0.498 ns  | -          | 0.418 ns  |
| Eigen           | 1.04 ns    | 1.00 ns    | 1.01 ns    | Eigen           | 0.533 ns  | -          | 0.412 ns  |
| Blaze           | 1.53 ns    | 1.00 ns    | 1.01 ns    | Blaze           | 0.580 ns  | -          | 0.408 ns  |
| Mathfu          | 2.82 ns    | 1.00 ns    | 1.02 ns    | Mathfu          | 1.75 ns   | -          | 0.415 ns  |
| Mango           | 1.70 ns    | 1.00 ns    | 1.01 ns    | Mango           | 0.495 ns  | -          | 0.410 ns  | 

Default configured GLM wasn't auto-vectorized by MSVC and GCC but Clang managed to do it and thats why its winning the multiply tests on benchmark.

GLM SIMD implementation of simd operations is based on often seen implementation that exploits type-punning:

```c++    
struct vec4 {
    union {
        float x,y,z,w;
        __m128 data;
    };
};
```

Unfortunately this is UB in C++. In fact all compilers support this technique properly, and there are no errors it is interesting how it is influencing optimization of such code.

For GLM SIMD, Eigen, Blaze and Mango resulted in same assembly code while compiling it with MSVC. This is what we expect as probably we can't have anything better.

```assembly    
mov         rax,qword ptr [testData]  
movups      xmm0,xmmword ptr [rax+10h]  
mulps       xmm0,xmmword ptr [rax]  
movaps      xmmword ptr [res],xmm0 
```
    
Also GLM SIMD and Eigen have best code for multiply scalar test:
    
```assembly    
mov         rax,qword ptr [testData]  
movss       xmm0,dword ptr [rax+14h]  
shufps      xmm0,xmm0,0  
mulps       xmm0,xmmword ptr [rax]  
movaps      xmmword ptr [res],xmm0  
```
   
Mango and Blaze implementations results in extra instruction which cost us a bit of performance in multiply by scalar test.

Mango:

```assembly    
mov         rax,qword ptr [testData]  
movups      xmm0,xmmword ptr [rax+10h]  
shufps      xmm0,xmm0,55h  
shufps      xmm0,xmm0,0  
mulps       xmm0,xmmword ptr [rax]  
movaps      xmmword ptr [res],xmm0
```
    
Blaze:
    
```assembly  
mov         rax,qword ptr [testData]  
movss       xmm1,dword ptr [rax+14h]  
shufps      xmm1,xmm1,0  
movups      xmm0,xmmword ptr [rax]  
mulps       xmm0,xmm1  
movaps      xmmword ptr [res],xmm0  
```
    
GLM and Mathfu implementations weren't vectorized by the compiler and I consider them uninteresing and won't comment on their assembly.

GCC didn't vectorized GLM code, for GLM SIMD and Mango produced:

```assembly  
mov    0x10(%rsp),%rax
movaps (%rax),%xmm1
movaps 0x10(%rax),%xmm0

sub    $0x1,%rbx
jne    0x55555555e6c0 <vec4_mult_simd(benchmark::State&)+144>

mulps  %xmm0,%xmm1
movaps %xmm1,(%rsp)
```
    
Interesting thing is to have loop control instructions (from benchmark library) in the middle of the loop - for other implementations I didn't included them on listings as they are after part which is doing actual work. On Travis CI where measuring time has better resolution this implementation seems to be a little better (around 0.499 ns vs 0.540 ns with version presented below for multiplication code). It seems to be best code overall.
    
Alternative assembly was produced by Eigen, Blaze and Mathfu libraries which is same as assemby produced by MSVC for Eigen and GLM SIMD:
 
```assembly  
mov    (%rsp),%rax
movaps 0x10(%rax),%xmm0
mulps  (%rax),%xmm0
movaps %xmm0,0x20(%rsp)
```

```assembly 
mov    (%rsp),%rax
movss  0x14(%rax),%xmm0
shufps $0x0,%xmm0,%xmm0
mulps  (%rax),%xmm0
movaps %xmm0,0x20(%rsp)
```

Clang done best job overall, for vanilla GLM was vectorized to following codes:

```assembly  
mov         rax,qword ptr [testData]  
movups      xmm0,xmmword ptr [rax]  
movups      xmm1,xmmword ptr [rax+10h]  
mulps       xmm1,xmm0  
movaps      xmmword ptr [res],xmm1
```
    
and 
    
```assembly
mov         rax,qword ptr [testData]  
movups      xmm0,xmmword ptr [rax]  
movss       xmm1,dword ptr [rax+14h]  
shufps      xmm1,xmm1,0  
mulps       xmm1,xmm0  
movaps      xmmword ptr [res],xmm1
```
    
Rest of libraries were compiled to this assembly which is better one and we already seen it:   
    
```assembly 
mov         rax,qword ptr [testData]  
movaps      xmm0,xmmword ptr [rax+10h]  
mulps       xmm0,xmmword ptr [rax]  
movaps      xmmword ptr [res],xmm0
```

```assembly 
mov         rax,qword ptr [testData]  
movss       xmm0,dword ptr [rax+10h]  
shufps      xmm0,xmm0,0  
mulps       xmm0,xmmword ptr [rax]  
movaps      xmmword ptr [res],xmm0 
```
     
## Compute tests results

That was easy part, as we measured only very simple operations like component-wise multiplication and multiplying vector by scalar value which very easily translates to SIMD operations. Now lets test little bit more complicated expressions. 

### Compute 1 test:

```c++    
glm::vec4 compute_1(float a, float b)
{
    glm::vec4 const av(a, b, b, a);
    glm::vec4 const bv(a, b, a, b);

    glm::vec4 const cv(bv * av);
    glm::vec4 const dv(av + cv);

    return dv;
}
```
    
Benchmark results:  

| Xeon E8450      | MSVC       | GCC        | CLANG      | i7 8850H        | MSVC      | GCC        | CLANG     |
| ----------------| ---------- | ---------- | ---------- | --------------- | --------- | ---------- | --------- |
| GLM             | 2.48 ns    | 1.00 ns    | 1.78 ns    | GLM             | 1.07 ns   | -          | 1.01 ns   |
| GLM SIMD        | 1.86 ns    | 1.00 ns    | 2.36 ns    | GLM SIMD        | 1.24 ns   | -          | 1.24 ns   |
| Eigen           | 6.08 ns    | 9.68 ns    | 2.38 ns    | Eigen           | 1.78 ns   | -          | 1.25 ns   |
| Blaze           | 16.0 ns    | 9.68 ns    | 2.37 ns    | Blaze           | 10.2 ns   | -          | 1.25 ns   |
| Mathfu          | 1.86 ns    | 1.75 ns    | 2.38 ns    | Mathfu          | 1.25 ns   | -          | 1.24 ns   |
| Mango           | 3.00 ns    | 1.00 ns    | 2.37 ns    | Mango           | 1.24 ns   | -          | 0.991 ns  |  

Lets analyze the results see the assembly code.

MSVC compiled code has two anomalies Eigen and Blaze implementations which we would expect to be much better. Eigen assembly is:

```assembly 
mov         rax,qword ptr [testData]  
movss       xmm3,dword ptr [rax+14h]  
movss       xmm4,dword ptr [rax]  
movaps      xmm2,xmm3  
movaps      xmm5,xmm4  
unpcklps    xmm2,xmm4  
unpcklps    xmm5,xmm3  
movlhps     xmm5,xmm2  
movaps      xmm1,xmm4  
unpcklps    xmm1,xmm3  
movlhps     xmm1,xmm1  
mulps       xmm1,xmm5  
addps       xmm1,xmm5  
movaps      xmmword ptr [res],xmm1 
```

unfortunately I don't know why this is being so slow and Blaze:

```assembly
mov         rax,qword ptr [rbp-39h]  
movss       xmm1,dword ptr [rax+14h]  
movss       xmm2,dword ptr [rax]  
movss       dword ptr [rbp-21h],xmm2  
movss       dword ptr [rbp-1Dh],xmm1  
movss       dword ptr [rbp-19h],xmm1  
movss       dword ptr [rbp-15h],xmm2  
xorps       xmm0,xmm0  
movups      xmmword ptr [rbp+17h],xmm0  
lea         rcx,[rbp-21h]  
lea         rdx,[rbp+17h]  
nop         dword ptr [rax]  
mov         eax,dword ptr [rcx]  
mov         dword ptr [rdx],eax  
lea         rdx,[rdx+4]  
add         rcx,4  
lea         rax,[rbp-11h]  
cmp         rcx,rax  
jne         vec4_compute_1+0A0h (07FF7644C72D0h)  
movss       dword ptr [rbp-11h],xmm2  
movss       dword ptr [rbp-0Dh],xmm1  
movss       dword ptr [rbp-9],xmm2  
movss       dword ptr [rbp-5],xmm1  
xorps       xmm0,xmm0  
movups      xmmword ptr [rbp+7],xmm0  
lea         rcx,[rbp-11h]  
lea         rdx,[rbp+7]  
nop         dword ptr [rax+rax]  
mov         eax,dword ptr [rcx]  
mov         dword ptr [rdx],eax  
lea         rdx,[rdx+4]  
add         rcx,4  
lea         rax,[rbp-1]  
cmp         rcx,rax  
jne         vec4_compute_1+0E0h (07FF7644C7310h)  
movaps      xmm0,xmmword ptr [rbp+7]  
movaps      xmm1,xmmword ptr [rbp+17h]  
mulps       xmm0,xmm1  
addps       xmm1,xmm0  
movaps      xmmword ptr [rbp+27h],xmm1  
```
    
It seems to be a real distaster. It seems that it has two loops where the vectors are constructed.

GLM SIMD and Mathfu resulted it following code:

```x86asm
mov         rax,qword ptr [testData]  
movss       xmm3,dword ptr [rax+14h]  
movss       xmm4,dword ptr [rax]  
movaps      xmm2,xmm3  
movaps      xmm5,xmm4  
unpcklps    xmm2,xmm4  
unpcklps    xmm5,xmm3  
movlhps     xmm5,xmm2  
movaps      xmm1,xmm4  
unpcklps    xmm1,xmm3  
movlhps     xmm1,xmm1  
movaps      xmm0,xmm5  
mulps       xmm0,xmm1  
addps       xmm5,xmm0  
movdqa      xmmword ptr [res],xmm5  
```

Mango code looks identical to Eigen but result is very different, I run measurements many times and timins were always the sameand I don't know why the Eigen results is biased. I checked that on Appveyor CI results were very similar anyway it is worse code than GLM SIMD / Mathfu.   

```x86asm   
mov         rax,qword ptr [testData]  
movups      xmm4,xmmword ptr [rax+10h]  
shufps      xmm4,xmm4,55h  
movups      xmm3,xmmword ptr [rax]  
movaps      xmm2,xmm4  
movaps      xmm5,xmm3  
unpcklps    xmm2,xmm3  
unpcklps    xmm5,xmm4  
movlhps     xmm5,xmm2  
movaps      xmm1,xmm3  
unpcklps    xmm1,xmm4  
movlhps     xmm1,xmm1  
mulps       xmm1,xmm5  
addps       xmm1,xmm5  
movaps      xmmword ptr [res],xmm1
```

GCC in case of this test inserted loop control instructions in the middle of the loop in four out of six times and it is harder to compare the best library in this case. GLM implementation is again not vectorized and generated worst code. Eigen and Blaze are identical and probably not best code but better than GLM. Shorter code was generated for GLM SIMD and Mathfu:

```x86asm 
mov    rax,QWORD PTR [rsp+0x10]
movss  xmm0,DWORD PTR [rax+0x14]
movss  xmm1,DWORD PTR [rax]
sub    rbx,0x1
jne    0x55555555e6c0 <vec4_compute_1(benchmark::State&)+144>
movaps xmm2,xmm1
unpcklps xmm2,xmm0
movaps xmm3,xmm2
unpcklps xmm0,xmm1
movlhps xmm3,xmm2
addps  xmm3,XMMWORD PTR [rip+0x3aa56]        # 0x555555599140
movlhps xmm2,xmm0
mulps  xmm3,xmm2
movaps XMMWORD PTR [rsp],xmm3
```

Shortest and probably best code was generated for Mango, this is one instruction less than previous one:

```x86asm 
mov    rax,QWORD PTR [rsp+0x10]
movaps xmm0,XMMWORD PTR [rax+0x10]
movaps xmm1,XMMWORD PTR [rax]
shufps xmm0,xmm0,0x55
sub    rbx,0x1
jne    0x55555555e600 <vec4_compute_1(benchmark::State&)+144>
unpcklps xmm1,xmm0
movaps xmm0,xmm1
movlhps xmm0,xmm1
addps  xmm0,XMMWORD PTR [rip+0x3aaea]        # 0x555555599110
shufps xmm1,xmm1,0x14
mulps  xmm0,xmm1
movaps XMMWORD PTR [rsp],xmm0
```

Clang again produces most consistent code, however best is produced for GLM library without SIMD support enabled, it is best result of all tested libraries:

```x86asm   
mov         rax,qword ptr [testData]  
movss       xmm0,dword ptr [rax]  
movss       xmm1,dword ptr [rax+14h]  
movaps      xmm2,xmm0  
mulss       xmm2,xmm1  
unpcklps    xmm0,xmm1  
movaps      xmm1,xmm0  
mulps       xmm1,xmm0  
shufps      xmm1,xmm2,4  
shufps      xmm0,xmm0,14h  
addps       xmm0,xmm1  
movaps      xmmword ptr [res],xmm0  
```
    
and other version for all other libraries:
    
```x86asm 
mov         rax,qword ptr [testData]  
movss       xmm0,dword ptr [rax]  
movss       xmm1,dword ptr [rax+14h]  
movaps      xmm2,xmm0  
unpcklps    xmm2,xmm1  
movaps      xmm3,xmm2  
shufps      xmm3,xmm1,84h  
shufps      xmm1,xmm0,0  
shufps      xmm0,xmm3,20h  
movaps      xmm3,xmm2  
shufps      xmm3,xmm0,24h  
shufps      xmm2,xmm1,24h  
mulps       xmm2,xmm3  
addps       xmm2,xmm3  
movaps      xmmword ptr [res],xmm2  
```


### 2. Compute 2 test:

```c++    
glm::vec4 compute_2(float a, float b)
{
    glm::vec4 const c(b * a);
    glm::vec4 const d(a + c);

    return d;
}
```
    
Benchmark results:  

| Xeon E8450      | MSVC       | GCC        | CLANG      | i7 8850H        | MSVC      | GCC        | CLANG     |
| ----------------| ---------- | ---------- | ---------- | --------------- | --------- | ---------- | --------- |
| GLM             | 2.37 ns    | 1.00 ns    | 1.22 ns    | GLM             | 0.498 ns  | -          | 0.424 ns  |
| GLM SIMD        | 1.02 ns    | 1.00 ns    | 1.18 ns    | GLM SIMD        | 0.498 ns  | -          | 0.497 ns  |
| Eigen           | 1.02 ns    | 7.08 ns    | 1.18 ns    | Eigen           | 0.567 ns  | -          | 0.496 ns  |
| Blaze           | 8.47 ns    | 7.01 ns    | 1.18 ns    | Blaze           | 6.73 ns   | -          | 0.528 ns  |
| Mathfu          | 1.33 ns    | 5.71 ns    | 1.19 ns    | Mathfu          | 0.744 ns  | -          | 0.422 ns  |
| Mango           | 1.51 ns    | 1.00 ns    | 1.23 ns    | Mango           | 0.500 ns  | -          | 0.406 ns  | 

Again as we see Clang is pretty stable and optimizing all libaries to similar code. In fact there were only two versions of disassembly:

```x86asm 
mov         rax,qword ptr [testData]  
movss       xmm0,dword ptr [rax]  
movss       xmm1,dword ptr [rax+14h]  
mulss       xmm1,xmm0  
addss       xmm1,xmm0  
shufps      xmm1,xmm1,0  
movaps      xmmword ptr [res],xmm1
```

and other for Mango:

```x86asm 
mov         rax,qword ptr [testData]  
movaps      xmm0,xmmword ptr [rax+10h]  
shufps      xmm0,xmm0,0E5h  
movss       xmm1,dword ptr [rax]  
mulss       xmm0,xmm1  
addss       xmm0,xmm1  
shufps      xmm0,xmm0,0  
movaps      xmmword ptr [res],xmm0  
```

Clang and as we will see most of other implementations understand that in fact in this case we don't need all data and in fact only two values are needed to compute the result and later this result is broadcasted to other elements of the vector. Nice!

MSVC code for GLM SIMD and Eigen is even one instruction less:

```x86asm 
mov         rax,qword ptr [testData]  
movss       xmm0,dword ptr [rax+14h]  
addss       xmm0,xmm1  
mulss       xmm0,dword ptr [rax]  
shufps      xmm0,xmm0,0  
movaps      xmmword ptr [res],xmm0
``` 

Mathfu and Mango resulted in slightly worse implementation:

```x86asm 
mov         rax,qword ptr [testData]  
movss       xmm1,dword ptr [rax]  
movaps      xmm0,xmm1  
mulss       xmm0,dword ptr [rax+14h]  
addss       xmm0,xmm1  
movaps      xmm1,xmm0  
shufps      xmm1,xmm1,0  
movdqa      xmmword ptr [res],xmm1
```  

Blaze assembly is much worse. Gcc in this case done worst job. Only for GLM assembly is optimal, and for other libraries it isn't as good, and for Eigen and Blaze is really awful, doing a lot of useless ```mov``` instructions. In some cases even it didn't use single instructions but vector instructions.
 
### 3. Compute 3 test:

```c++    
glm::vec4 compute_3(glm::vec4 a, glm::vec4 b)
{
    return a * b + a * b;
}
```
    
Benchmark results:  

| Xeon E8450      | MSVC       | GCC        | CLANG      | i7 8850H        | MSVC      | GCC        | CLANG     |
| ----------------| ---------- | ---------- | ---------- | --------------- | --------- | ---------- | --------- |
| GLM             | 5.15 ns    | 3.02 ns    | 2.05 ns    | GLM             | 2.23 ns   | -          | 0.437 ns  |
| GLM SIMD        | 1.47 ns    | 1.00 ns    | 1.02 ns    | GLM SIMD        | 0.497 ns  | -          | 0.498 ns  |
| Eigen           | 6.75 ns    | 1.30 ns    | 1.01 ns    | Eigen           | 2.10 ns   | -          | 0.498 ns  |
| Blaze           | 2.39 ns    | 1.00 ns    | 1.03 ns    | Blaze           | 0.776 ns  | -          | 0.534 ns  |
| Mathfu          | 1.52 ns    | 1.00 ns    | 1.02 ns    | Mathfu          | 0.593 ns  | -          | 0.416 ns  |
| Mango           | 1.49 ns    | 1.00 ns    | 1.02 ns    | Mango           | 0.496 ns  | -          | 0.408 ns  | 

Again Clang generated following code in most cases - it used common subexpression elimination optimization and uses only one addition and one multiplication instead of two:

```x86asm 
mov         rax,qword ptr [testData]  
movaps      xmm0,xmmword ptr [rax]  
addps       xmm0,xmm0  
mulps       xmm0,xmmword ptr [rax+10h]  
movaps      xmmword ptr [res],xmm0
```

GCC also optimized code well outside of Eigen (8 instructions) and GLM (unvectorized) in other cases it used common subexpression elimination. MSVC best disassembly was for Mathfu - the only one where it used one addition and one multiplication, second best was for GCC and Mango (one more instruction then Mathfu).

## Swizzle tests

GLM and Mango support useful functionality known as swizzling for accessing vector members. I as before I tested two modes of GLM library and Mango.
Both MSVC and GCC compiled code much better for SIMD libraries.

1. Swizzle test 1:

```c++    
inline glm::vec4 test_swizzle_1(glm::vec4 a, glm::vec4 b, glm::vec4 c) 
{
    return a.wwww() * b.xxyy() + (c.xxzz() - a).zzzz() * b.w;
}
```

Benchmark results:  

| Xeon E8450      | MSVC       | GCC        | CLANG      | i7 8850H        | MSVC      | GCC        | CLANG     |
| ----------------| ---------- | ---------- | ---------- | --------------- | --------- | ---------- | --------- |
| GLM             | 5.55 ns    | 2.40 ns    | 2.59 ns    | GLM             | 1.93 ns   | -          | 1.10 ns   |
| GLM SIMD        | 4.10 ns    | 2.00 ns    | 2.24 ns    | GLM SIMD        | 1.63 ns   | -          | 1.24  ns  |
| Mango           | 4.21 ns    | 2.01 ns    | 2.10 ns    | Mango           | 1.08 ns   | -          | 1.26  ns  | 

2. Swizzle test 2:

```c++    
inline glm::vec4 test_swizzle_2(glm::vec4 a, glm::vec4 b) 
{
    return a.xyyz() * b.wxxw() + a * b.w;
}
```

Benchmark results:

| Xeon E8450      | MSVC       | GCC        | CLANG      | i7 8850H        | MSVC      | GCC        | CLANG     |
| ----------------| ---------- | ---------- | ---------- | --------------- | --------- | ---------- | --------- |
| GLM             | 7.21 ns    | 2.39 ns    | 3.58 ns    | GLM             | 1.83 ns   | -          | 1.87 ns   |
| GLM SIMD        | 3.35 ns    | 1.34 ns    | 1.96 ns    | GLM SIMD        | 1.06 ns   | -          | 1.14 ns   |
| Mango           | 3.31 ns    | 1.34 ns    | 1.83 ns    | Mango           | 0.880 ns  | -          | 1.25 ns   | 

Generally GLM non - SIMD implementation is usually much worse than SIMD / Mango which looks same, almost always for the same compiler. Only test_swizzle_1 GLM implementation resulted with better while compiling with Clang and GCC than SIMD version. GCC time results aren't worth anything again and we have to compare assembly code. And here we see the usual pattern Clang > GCC > MSVC. I won't paste all assemblies (they are available in my repo) but just will compare number of instructions. Clang has 12/14/14 and 21/11/11 instructions in assembly, GCC has 14/15/15 and 19/13/13, MSVC 24/17/17 and 31/13/13 instructions in test_swizzle_1 and test_swizzle_2 respectively.

## Martix 4x4 tests

For matrices I tested add and multiply operations. From 3d math graphics library perhaps most interesting operation for us is matrix multiplication. After a while of thinking maybe operation like ```transpose()``` would also interesting or specialized methods for constructing projection or view matrices would also be worth testing. I don't know if general purpose libraries like Eigen or Blaze support that out of the box.
Mango library doesn't support adding matrices, so this result isn't available.

1. Add test:

```c++    
for (auto _ : state) {
    benchmark::ClobberMemory();
    res = testData[0] + testData[1];
    benchmark::ClobberMemory();
}
```
    
2. Multiply test:

```c++    
for (auto _ : state) {
    benchmark::ClobberMemory();
    res = testData[0] * testData[1];
    benchmark::ClobberMemory();
}
```

### 1. GLM

| Xeon E8450  | MSVC       | GCC        | CLANG      | i7 8850H   | MSVC       | GCC        | CLANG      |
| ----------- | ---------- | ---------- | ---------- | ---------- | ---------- | ---------- | ---------- |
| Add         | 18.3 ns    | 11.7 ns    | 9.26  ns   | Add        | 10.3 ns    | -          | 1.02 ns    |
| Multiply    | 68.7 ns    | 11.7 ns    | 32.8  ns   | Multiply   | 21.8 ns    | -          | 7.01 ns    |

### 2. GLM SIMD

| Xeon E8450  | MSVC       | GCC        | CLANG      | i7 8850H   | MSVC       | GCC        | CLANG      |
| ----------- | ---------- | ---------- | ---------- | ---------- | ---------- | ---------- | ---------- |
| Add         | 5.47 ns    | 3.03 ns    | 4.63  ns   | Add        | 5.81 ns    | -          | 1.74 ns    |
| Multiply    | 14.8 ns    | 7.18 ns    | 17.4  ns   | Multiply   | 9.33 ns    | -          | 3.82 ns    |

### 3. Eigen

| Xeon E8450  | MSVC       | GCC        | CLANG      | i7 8850H   | MSVC       | GCC        | CLANG      |
| ----------- | ---------- | ---------- | ---------- | ---------- | ---------- | ---------- | ---------- |
| Add         | 5.97 ns    | 3.17 ns    | 4.52  ns   | Add        | 0.955 ns   | -          | 0.867 ns   |
| Multiply    | 18.1 ns    | 8.55 ns    | 15.0  ns   | Multiply   |  8.06 ns   | -          |  4.47 ns   |

### 4. Blaze

| Xeon E8450  | MSVC       | GCC        | CLANG      | i7 8850H   | MSVC       | GCC        | CLANG      |
| ----------- | ---------- | ---------- | ---------- | ---------- | ---------- | ---------- | ---------- |
| Add         | 10.2 ns    | 3.17 ns    | 6.71  ns   | Add        | 3.11 ns    | -          | 1.79 ns    |
| Multiply    | 24.5 ns    | 8.66 ns    | 19.3  ns   | Multiply   | 18.4 ns    | -          | 6.07 ns    |

### 5. Mathfu

| Xeon E8450  | MSVC       | GCC        | CLANG      | i7 8850H   | MSVC       | GCC        | CLANG      |
| ----------- | ---------- | ---------- | ---------- | ---------- | ---------- | ---------- | ---------- |
| Add         | 12.3 ns    | 3.17 ns    | 5.08  ns   | Add        | 7.41 ns    | -          | 1.63 ns    |
| Multiply    | 32.9 ns    | 43.1 ns    | 50.9  ns   | Multiply   | 16.4 ns    | -          | 21.3 ns    |

### 6. Mango

| Xeon E8450  | MSVC       | GCC        | CLANG      | i7 8850H   | MSVC       | GCC        | CLANG      |
| ----------- | ---------- | ---------- | ---------- | ---------- | ---------- | ---------- | ---------- |
| Multiply    | 15.2 ns    | 9.68 ns    | 15.9  ns   | Add        | 4.30 ns    | -          | 4.74 ns    |

