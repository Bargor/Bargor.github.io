---
layout: post
title: Comparison of 3d Math libraries
---

In this article I want to compare performance of basic operations of different popular (and less popular) math libraries. I will focus on stuff related to 3d math used in geometry processing, so I tested Vector4 and 4x4 Matrix implementations. For GLM and Mango library I also tested swizzle implementations as this libraries have this feature.

I tested presented libraries on 3 major compilers (MSVC, Gcc, Clang) using google benchmark. Benchmark results are quite interesting as it tells us which library is most performant, however dissassemblies are most interesting because we can see how each implementation compiles on different compilers.

For benchmark I took GLM, Mathfu and Mango ad typical 3d game math libraries and Eigen and Blaze as state of the art general purpose math libraries. GLM library is tested in two modes "out of the box" and SIMD configured mode. for other libraries I'm not very familiar with them so I took default settings. Tests were made on two processor I had access to: old Xeon E5450 and i7-8850H, on second one I also run benchmarks compiled with AVX2 instrucions where is was possible.

## Vector4 tests

I set up few tests on which I will do the comparisons:
    
1. Addition test:
    
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
        
3. Multiply by scalar:

```c++    
for (auto _ : state) {
    benchmark::ClobberMemory();
    res = testData[0] * testData[1].y;
    benchmark::ClobberMemory();
}
```       

4. Compute 1 test:

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
    
5. Compute 2 test:

```c++    
glm::vec4 compute_2(float a, float b)
{
    glm::vec4 const c(b * a);
    glm::vec4 const d(a + c);

    return d;
}
```
    
6. Compute 3 test:

```c++    
glm::vec4 compute_3(glm::vec4 a, glm::vec4 b)
{
    return a * b + a * b;
}
```

### 1. GLM
    
|                  | MSVC    | GCC     | CLANG   |
| ---------------- | ------- | ------- | ------- |
| Add              | 5.09 ns | 3.03 ns | 1.90 ns |
| Multiply         | 5.31 ns | 3.02 ns | 1.86 ns |
| Multiply scalar  | 4.43 ns | 2.01 ns | 1.39 ns |
| Compute 1        | 2.98 ns | 1.00 ns | 1.76 ns |
| Compute 2        | 2.45 ns | 1.00 ns | 1.18 ns |
| Compute 3        | 5.96 ns | 3.02 ns | 1.85 ns |
    
Default configured GLM wasn't auto-vectorized by MSVC and GCC but Clang managed to do it and thats why its winning the add and multiply tests on benchmark.

### 2. GLM SIMD

### 3. Eigen

### 4. Blaze

### 5. Mathfu

### 6. Mango

## Swizzle tests

## Martix 4x4 tests

