---
title: Arithmetic
weight: 6
published: true
---

이 책에서 여러 차례 보여주었듯이, 명령어 집합의 사각지대를 살펴보는 것은 특히 [CISC](/hpc/architecture/isa)플랫폼, 예를 들어 현재 명령어 수가 계산 방식에 따라 [약 1000개에서 4000개](https://stefanheule.com/blog/how-many-x86-64-instructions-are-there-anyway/)에 이르는 x86의 경우 매우 유익할 수 있습니다.

이러한 명령어들의 대부분은 산술 연산과 관련이 있으며, 이들을 효율적으로 활용해 산술 연산을 최적화하려면 풍부한 지식은 물론, 높은 수준의 기술과 창의성까지 요구됩니다. 따라서 이번 장에서는 수의 표현 방식과 이를 수치 알고리즘에서 어떻게 활용할 수 있는지를 다루겠습니다.

<!--

Knowing darker corners of the instruction set can be very fruitful, especially in the case of [CISC](/hpc/architecture/isa) platforms like x86, which currently has [somewhere between 1000 and 4000](https://stefanheule.com/blog/how-many-x86-64-instructions-are-there-anyway/) distinct instructions, depending on how you count.

In this chapter, we will discuss number representations and their use in numerical algorithms, as well as some core mathematical concepts in algebra and number theory that are often overlooked in computer science curricula.

-->
