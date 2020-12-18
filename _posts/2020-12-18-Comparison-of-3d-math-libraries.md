---
layout: post
title: Comparison of 3d Math libraries
---

In this article I want to compare performance of basic operations of different popular (and less popular) math libraries. I will focus on stuff related to 3d math used in geometry processing, so I tested Vector4 and 4x4 Matrix implementations. For GLM and Mango library I also tested swizzle implementations as this libraries have this feature.

I tested presented libraries on 3 major compilers (MSVC, Gcc, Clang) using google benchmark. Benchmark results are quite interesting as it tells us which library is most performant, however dissassemblies are most interesting because we can see how each implementation compiles on different compilers.

Few months ago I've tried to improve implementation of the GLM library and discovered that run across into 