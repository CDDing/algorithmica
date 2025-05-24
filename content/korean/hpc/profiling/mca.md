---
title: Machine Code Analyzers
weight: 4
---

A *machine code analyzer* is a program that takes a small snippet of assembly code and [simulates](../simulation) its execution on a particular microarchitecture using information available to compilers, and outputs the latency and throughput of the whole block, as well as cycle-perfect utilization of various resources within the CPU.
기계어 분석기는 어셈블리 코드의 일부를 받아 특정 마이크로 아키텍처에서 컴파일러에서 사용할 수 있는 정보를 사용해 시뮬레이션하고 전체 블럭의 지연과 처리량에 더해 CPU의 다양한 자원의 사이클 완벽 유틸을 출력합니다. 

### `llvm-mca` 사용하기

There are many different machine code analyzers, but I personally prefer `llvm-mca`, which you can probably install via a package manager together with `clang`. You can also access it through a web-based tool called [UICA](https://uica.uops.info) or in the [Compiler Explorer](https://godbolt.org/) by selecting "Analysis" as the language.
다양한 기계어 분석기가 존재하지만, 필자 개인적으로는 `llvm-mca`를 선호합니다. 이는 아마 `clang`과 함께있는 패키지 매니저를 통해 설치할 수 있을 것입니다. 또한 웹 기반 도구인 UICA 혹은 Compiler Explorer를 사용할 수도 있습니다.

What `llvm-mca` does is it runs a set number of iterations of a given assembly snippet and computes statistics about the resource usage of each instruction, which is useful for finding out where the bottleneck is.
`llvm-mca`가 하는 일은 주어진 어셈블리 스니펫을 특정 횟수만큼 반복해 실행하고 각 명령간 사용되는 자원의 정보들을 계산합니다. 이는 병목을 찾는데 유용합니다.

We will consider the array sum as our simple example:
단순한 예제로 배열 합 예제를 살펴보겠습니다.

```asm
loop:
    addl (%rax), %edx
    addq $4, %rax
    cmpq %rcx, %rax
    jne	 loop
````

Here is its analysis with `llvm-mca` for the Skylake microarchitecture:
여기 `llvm-mca`를 통한 Skylake 마이크로 아키텍처에서의 분석이 있습니다.

```yaml
Iterations:        100
Instructions:      400
Total Cycles:      108
Total uOps:        500

Dispatch Width:    6
uOps Per Cycle:    4.63
IPC:               3.70
Block RThroughput: 0.8
```

First, it outputs general information about the loop and the hardware:
먼저, 반복문과 하드웨어에 관한 일반적인 정보를 출력합니다.

- It "ran" the loop 100 times, executing 400 instructions in total in 108 cycles, which is the same as executing $\frac{400}{108} \approx 3.7$ [instructions per cycle](/hpc/complexity/hardware) on average (IPC).
- The CPU is theoretically capable of executing up to 6 instructions per cycle ([dispatch width](/hpc/architecture/layout)).
- Each cycle in theory can be executed in 0.8 cycles on average ([block reciprocal throughput](/hpc/pipelining/tables)).
- The "uOps" here are the micro-operations that the CPU splits each instruction into (e.g., fused load-add is composed of two uOps).
- 반복문을 100회 실행했고, 400 명령어를 총 108 사이클 동안 수행했으며, 이는 $\frac{400}{108} \approx 3.7$에 수렴하므로 평균(IPC)에 근사합니다.
- CPU는 이론적으로 사이클당 6개의 명령어를 실행할 수 있습니다.
- 이론적으로는 각 사이클은 평균적으로 0.8사이클동안 실행됩니다.
- 여기서의 uOps는 CPU가 각 명령어를 나눈 단위인 미세 명령입니다. 예를 들어 혼합된 load-add를 2개의 uOps로 분해합니다. 

Then it proceeds to give information about each individual instruction: 
그러면 각 개별 명령어에 대한 정보를 전달합니다.

```yaml
Instruction Info:
[1]: uOps
[2]: Latency
[3]: RThroughput
[4]: MayLoad
[5]: MayStore
[6]: HasSideEffects (U)

[1]    [2]    [3]    [4]    [5]    [6]    Instructions:
 2      6     0.50    *                   addl	(%rax), %edx
 1      1     0.25                        addq	$4, %rax
 1      1     0.25                        cmpq	%rcx, %rax
 1      1     0.50                        jne	-11
```

There is nothing there that there isn't in the [instruction tables](/hpc/pipelining/tables):
이들이 명령어 테이블에 없기 때문에 아무것도 없습니다.

- how many uOps each instruction is split into;
- how many cycles each instruction takes to complete (latency);
- how many cycles each instruction takes to complete in the amortized sense (reciprocal throughput), considering that several copies of it can be executed simultaneously.
- 명령어마다 얼마나 많은 uOps로 나뉘어지는지.
- 명령어마다 얼마나 많은 사이클을 필요로 하는지(지연).
- 명령어마다 얼마나 많은 처리를 하는지(처리량), 동시에 얼마나 많은 복사본을 실행할 수 있는지를 논합니다.

Then it outputs probably the most important part — which instructions are executing when and where:
그러면 이는 아마 가장 중요한 부분인 명령어가 어디에서 언제 실행되는지를 출력합니다.

```yaml
Resource pressure by instruction:
[0]    [1]    [2]    [3]    [4]    [5]    [6]    [7]    [8]    [9]    Instructions:
 -      -     0.01   0.98   0.50   0.50    -      -     0.01    -     addl (%rax), %edx
 -      -      -      -      -      -      -     0.01   0.99    -     addq $4, %rax
 -      -      -     0.01    -      -      -     0.99    -      -     cmpq %rcx, %rax
 -      -     0.99    -      -      -      -      -     0.01    -     jne  -11
```

As the contention for execution ports causes [structural hazards](/hpc/pipelining/hazards), ports often become the bottleneck for throughput-oriented loops, and this chart helps diagnose why. It does not give you a cycle-perfect Gantt chart of something like that, but it gives you the aggregate statistics of the execution ports used for each instruction, which lets you find which one is overloaded.
실행 위치에 대한 경쟁이 구조적 위험을 초래하기 때문에, 실행 위치는 처리량 기준 반복문을 위한 병목이 종종 됩니다. 또한 이 차트는 왜인지를 분석합니다. 이는 사이클 완벽 Gantt 차트같은 것을 보여주지는 않지만, 각 명령어에 사용되는 실행 위치의 집계 상태를 줍니다. 이는 어디가 오버로딩되었는지 찾을 수 있게 합니다.

<!--

A CPU is a very complicated thing, but in essence, there are several "ports" that specialize on particular kinds of instructions. These ports often become the bottleneck, and the chart above helps in diagnosing why.

We are not ready to discuss how this works yet, but will talk about it in detail in the last chapter.

-->
