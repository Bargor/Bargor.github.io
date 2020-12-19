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
    

| Xeon E8450          | MSVC       | GCC        | CLANG      | i7 8850H            | MSVC       | GCC        | CLANG      |
| ------------------- | ---------- | ---------- | ---------- | ------------------- | ---------- | ---------- | ---------- |
| Add                 | 5.09 ns    | 3.03 ns    | 1.90 ns    | Add                 | 3.48 ns    | -          | 0.459 ns   |
| Multiply            | 5.31 ns    | 3.02 ns    | 1.86 ns    | Multiply            | 3.24 ns    | -          | 0.510 ns   |
| Multiply scalar     | 4.43 ns    | 2.01 ns    | 1.39 ns    | Multiply scalar     | 2.49 ns    | -          | 0.514 ns   |
| Compute 1           | 2.98 ns    | 1.00 ns    | 1.76 ns    | Compute 1           | 1.37 ns    | -          | 1.01 ns    |
| Compute 2           | 2.45 ns    | 1.00 ns    | 1.18 ns    | Compute 2           | 0.642 ns   | -          | 0.424 ns   |
| Compute 3           | 5.96 ns    | 3.02 ns    | 1.85 ns    | Compute 3           | 3.24 ns    | -          | 0.437 ns   |
    
Default configured GLM wasn't auto-vectorized by MSVC and GCC but Clang managed to do it and thats why its winning the add and multiply tests on benchmark.

### 2. GLM SIMD

| Xeon E8450          | MSVC       | GCC        | CLANG      | i7 8850H            | MSVC       | GCC        | CLANG      |
| ------------------- | ---------- | ---------- | ---------- | ------------------- | ---------- | ---------- | ---------- |
| Add                 | 1.40 ns    | 1.01 ns    | 1.12 ns    | Add                 | 0.618 ns   | -          | 0.532 ns   |
| Multiply            | 1.37 ns    | 1.00 ns    | 1.01 ns    | Multiply            | 0.524 ns   | -          | 0.439 ns   |
| Multiply scalar     | 1.01 ns    | 1.00 ns    | 1.01 ns    | Multiply scalar     | 0.528 ns   | -          | 0.429 ns   |
| Compute 1           | 2.22 ns    | 1.00 ns    | 2.36 ns    | Compute 1           | 1.25 ns    | -          |  1.25 ns   |
| Compute 2           | 1.53 ns    | 1.00 ns    | 1.18 ns    | Compute 2           | 0.578 ns   | -          | 0.503 ns   |
| Compute 3           | 2.10 ns    | 1.00 ns    | 1.02 ns    | Compute 3           | 0.535 ns   | -          | 0.519 ns   |

### 3. Eigen

| Xeon E8450          | MSVC       | GCC        | CLANG      | i7 8850H            | MSVC       | GCC        | CLANG      |
| ------------------- | ---------- | ---------- | ---------- | ------------------- | ---------- | ---------- | ---------- |
| Add                 | 1.59 ns    | 1.00 ns    | 1.12 ns    | Add                 | 0.586 ns   | -          | 0.528 ns   |
| Multiply            | 1.88 ns    | 1.00 ns    | 1.01 ns    | Multiply            | 0.506 ns   | -          | 0.419 ns   |
| Multiply scalar     | 1.11 ns    | 1.00 ns    | 1.01 ns    | Multiply scalar     | 0.533 ns   | -          | 0.428 ns   |
| Compute 1           | 8.75 ns    | 9.68 ns    | 2.35 ns    | Compute 1           | 1.78 ns    | -          | 1.24 ns    |
| Compute 2           | 1.35 ns    | 7.08 ns    | 1.18 ns    | Compute 2           | 0.567 ns   | -          | 0.422 ns   |
| Compute 3           | 7.84 ns    | 1.30 ns    | 1.01 ns    | Compute 3           | 2.10 ns    | -          | 0.429 ns   |

### 4. Blaze

| Xeon E8450          | MSVC       | GCC        | CLANG      | i7 8850H            | MSVC       | GCC        | CLANG      |
| ------------------- | ---------- | ---------- | ---------- | ------------------- | ---------- | ---------- | ---------- |
| Add                 | 1.43 ns    | 1.00 ns    | 1.11 ns    | Add                 | 0.551 ns   | -          | 0.476 ns   |
| Multiply            | 1.43 ns    | 1.00 ns    | 1.01 ns    | Multiply            | 0.513 ns   | -          | 0.434 ns   |
| Multiply scalar     | 1.53 ns    | 1.00 ns    | 1.01 ns    | Multiply scalar     | 0.580 ns   | -          | 0.435 ns   |
| Compute 1           | 16.0 ns    | 4.44 ns    | 2.36 ns    | Compute 1           | 10.2 ns    | -          | 1.25 ns    |
| Compute 2           | 8.47 ns    | 7.01 ns    | 1.27 ns    | Compute 2           | 6.73 ns    | -          | 0.785 ns   |
| Compute 3           | 2.39 ns    | 1.00 ns    | 1.03 ns    | Compute 3           | 0.776 ns   | -          | 0.546 ns   |

### 5. Mathfu

| Xeon E8450          | MSVC       | GCC        | CLANG      | i7 8850H            | MSVC       | GCC        | CLANG      |
| ------------------- | ---------- | ---------- | ---------- | ------------------- | ---------- | ---------- | ---------- |
| Add                 | 5.02 ns    | 1.00 ns    | 1.06 ns    | Add                 | 1.91 ns    | -          | 0.458 ns   |
| Multiply            | 4.05 ns    | 1.00 ns    | 1.01 ns    | Multiply            | 1.75 ns    | -          | 0.425 ns   |
| Multiply scalar     | 3.30 ns    | 1.00 ns    | 1.01 ns    | Multiply scalar     | 1.75 ns    | -          | 0.415 ns   |
| Compute 1           | 2.22 ns    | 1.75 ns    | 2.37 ns    | Compute 1           | 1.25 ns    | -          |  1.24 ns   |
| Compute 2           | 1.58 ns    | 6.97 ns    | 1.18 ns    | Compute 2           | 0.744 ns   | -          | 0.421 ns   |
| Compute 3           | 1.94 ns    | 1.00 ns    | 1.01 ns    | Compute 3           | 0.593 ns   | -          | 0.416 ns   |

### 6. Mango

| Xeon E8450          | MSVC       | GCC        | CLANG      | i7 8850H            | MSVC       | GCC        | CLANG      |
| ------------------- | ---------- | ---------- | ---------- | ------------------- | ---------- | ---------- | ---------- |
| Add                 | 3.48 ns    | 1.00 ns    | 1.08 ns    | Add                 | 0.600 ns   | -          | 0.452 ns   |
| Multiply            | 1.35 ns    | 1.00 ns    | 1.01 ns    | Multiply            | 0.506 ns   | -          | 0.419 ns   |
| Multiply scalar     | 1.01 ns    | 1.00 ns    | 1.01 ns    | Multiply scalar     | 0.510 ns   | -          | 0.433 ns   |
| Compute 1           | 2.91 ns    | 1.00 ns    | 2.36 ns    | Compute 1           | 1.27 ns    | -          |  1.02 ns   |
| Compute 2           | 2.02 ns    | 1.00 ns    | 1.22 ns    | Compute 2           | 0.589 ns   | -          | 0.502 ns   |
| Compute 3           | 1.89 ns    | 1.00 ns    | 1.01 ns    | Compute 3           | 0.525 ns   | -          | 0.528 ns   |

## Swizzle tests

## Martix 4x4 tests

