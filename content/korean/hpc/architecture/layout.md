---
title: Machine Code Layout
weight: 10
published: true
---

컴퓨터 엔지니어들은 CPU의 파이프라인을 [두 부분](/hpc/pipelining)으로 나누어 생각하곤 합니다. 메모리에서 명령어를 가져와 디코딩하는 **프론트엔드**와, 명령어를 스케줄링하고 실제로 실행하는 **백엔드** 입니다. 일반적으로 성능 병목은 실행 단계에서 발생하며, 이러한 이유로 이 책의 대부분은 백엔드 최적화의 초점을 맞출 것입니다.

하지만 프론트엔드가 명령어를 충분히 빠르게 공급하지 못해 백엔드가 포화 상태에 도달하지 못하는 경우도 있습니다. 이러한 현상은 여러 이유로 발생할 수 있으며, 결국에는 기계어 코드가 메모리에 어떻게 배치되어 있는지와 밀접한 관련이 있습니다. 예를 들어, 사용되지 않는 코드를 제거하거나, `if` 분기의 순서를 바꾸거나, 함수 선언 순서만 바꿔도 성능이 향상되거나 저하되는 사례가 존재합니다.

### CPU 프론트엔드

기계어가 명령어로 해석되어 CPU가 프로그래머의 의도를 이해하기 전에, 먼저 우리가 주목할 두 가지 중요한 단계를 거쳐야 합니다. 바로 **fetch**와 **decode** 단계입니다.

fetch 단계에서는, CPU가 메인 메모리로부터 일정 크기의 바이트 블록을 단순히 읽어옵니다. 이 블록은 여러 명령어의 바이너리 인코딩을 담고 있습니다. 이 블록의 크기는 x86 기준 일반적으로 32 바이트이며, 다른 시스템에서는 다를 수 있습니다. 중요한 점은 이 블록이 반드시 [정렬(aligned)](/hpc/cpu-cache/cache-lines)되어야 한다는 점입니다. 즉, 이 바이트 묶음의 주소는 블록 크기(여기선 32바이트)의 배수여야 합니다.

<!-- todo: what happens when an instruction crosses the boundary? -->

다음은 decode 단계입니다. CPU는 이 바이트 묶음을 보고, 명령어 포인터 이전의 내용은 버리고 나머지를 명령어 단위로 분리합니다. 기계어 명령어는 가변 길이 바이트로 인코딩 되어 있습니다. 예를 들어, `inc rax`처럼 단순하고 자주 쓰이는 명령어는 1바이트만 차지하지만, 인코딩된 상수나 동작을 바꾸는  접두어가 포함된 복잡한 명령어는 최대 15바이트까지 차지할 수 있습니다. 따라서 하나의 32바이트 블록에서 디코딩되는 명령어 수는 가변적이지만, 각 CPU마다 정해진 한계인 decode width를 넘을 수는 없습니다. 예를 들어, 필자가 사용하는 [Zen 2](https://en.wikichip.org/wiki/amd/microarchitectures/zen_2) 아키텍처에서는 decode width가 4이므로, 한 사이클당 최대 4개의 명령어만 디코딩되어 다음 단계로 넘어갈 수 있습니다.

이 단계들은 파이프라인 방식으로 동작합니다. CPU가 다음에 어떤 명령어 블록이 필요한지 파악하거나 [예측](/hpc/pipelining/branching/)할 수 있다면, fetch 단계는 현재 블록의 마지막 명령어가 디코딩되기를 기다리지 않고 즉시 다음 블록을 불러옵니다.

<!--

Decoded Stream Buffer (DSB)

Loop Stream Detector (LSD)

-->

### 코드 정렬

다른 조건이 동일하다면, 컴파일러는 일반적으로 더 짧은 기계어 명령어를 선호합니다. 그 이유는 더 많은 명령어를 단일 32바이트 fetch 블록에 담을 수 있고, 전체 바이너리 크기를 줄일 수 있기 때문입니다. 하지만 fetch된 명령어 블록이 반드시 정렬되어야 한다는 제약 때문에, 반대의 선택이 더 나은 경우도 있습니다.

예를 들어, 32바이트 정렬된 블록의 마지막 바이트에서 시작하는 명령어 시퀀스를 실행해야 한다고 상상해봅시다. 첫 번째 명령어는 지연 없이 실행할 수 있겠지만, 그 이후의 명령어들은 다음 블록을 fetch 하기 위해 한 사이클을 더 기다려야 합니다. 반면, 코드 블록이 32바이트 경계에 맞춰 정렬되어 있었다면, 최대 4개의 명령어를 동시에 디코딩하고 실행할 수 있었을 것입니다(물론 명령어가 너무 길거나 서로 의존하는 경우는 예외입니다).

이러한 점을 고려해, 컴파일러는 겉보기에는 비효율적으로 보이는 최적화를 수행하기도 합니다. 중요한 점프 위치를 2의 제곱 경계에 정렬시키기 위해, 더 긴 기계어 명령어를 선택하거나, 아무 기능도 하지 않는[^nop] 더미 명령어를 삽입하기도 합니다.

[^nop]: 이러한 명령어들은 no-op 혹은 NOP 명령어라고 불립니다. x86 아키텍처에서 공식적으로 "아무 것도 하지 않는" 방법은 `xchg rax, rax`(레지스터 자신을 교환)이며, CPU는 이를 특별히 인식해서 디코드 단계를 제외하면 별도의 실행 사이클을 소모하지 않습니다. `nop`는 이와 동일한 기계어로 매핑되는 축약형 명령어입니다.

GCC에서는 `-falign-labels=n` 플래그를 사용하여 특정 정렬 기준을 지정할 수 있습니다. 보다 세분화하고 싶다면 `-labels` 대신 `-functions`, `-loops`, `-jumps`로 [대체](https://gcc.gnu.org/onlinedocs/gcc/Optimize-Options.html)할 수 있습니다. 최적화 레벨 `-O2`나 `O3`에서는 이 기능이 기본적으로 활성화되어 있으며, 별도의 정렬 기준을 지정하지 않으면 시스템에 따라 적절하게 설정된 기본값을 사용합니다.

<!-- Having to decode a bunch of extra NOPs is usually not a problem. -->

### 명령어 캐시

명령어는 대체로 데이터와 동일한 [메모리 시스템](/hpc/cpu-cache)을 사용하여 저장되고 fetch됩니다. 다만, 낮은 계층의 캐시는 별도의 명령어 캐시로 대체될 수 있습니다. 왜냐하면 임의의 데이터 읽기가 데이터를 처리하는 코드를 캐시에서 밀어내는 상황을 원하지 않기 때문입니다.

명령어 캐시는 다음과 같은 상황에서 중요합니다.

- 무슨 명령어가 다음에 실행될 지 모르고, 다음 블록을 [낮은 지연 시간](/hpc/cpu-cache/latency)으로 fetch해야 할 때
- 혹은 긴 흐름의 장황하지만 빠르게 실행해야 할 명령어를 실행하고 있으며, [높은 대역폭](/hpc/cpu-cache/bandwidth)이 필요할 때

The memory system can therefore become the bottleneck for programs with large machine code. This consideration limits the applicability of the optimization techniques we've previously discussed:
따라서 메모리 시스템은 긴 기계어 코드를 가진 프로그램에서 병목 현상을 일으킬 수 있습니다. 이러한 점은 이전에 논의했던 최적화 기법들의 적용 가능성을 제한합니다.

- [함수 인라이닝](../functions)은 항상 최적화된 방법은 아닙니다. 왜냐하면 코드 공유를 줄이고 바이너리 크기를 증가시켜 더 많은 명령어 캐시를 요구하기 때문입니다.
- [반복문을 푸는 것](../loops)은 반복문의 수를 컴파일 시점에 알 수 있을 때에만 유리합니다. 일정 시점 이후 CPU는 명령어와 데이터를 메인 메모리에서 fetch해야 하기 때문에 메모리 대역폭에서 병목 현상이 발생할 가능성이 큽니다.
- 큰 [코드 정렬](#code-alignment)은 바이너리 크기를 증가시키며, 이로 인해 더 많은 명령어 캐시를 요구합니다. fetch에 한 사이클을 더 소모하는 것은 메인 메모리에서 명령어를 fetch하는 것과 캐시 미스에 비해 상대적으로 작은 단점입니다. 

또 다른 측면은 자주 사용되는 명령어 흐름을 동일한 [캐시 라인](/hpc/cpu-cache/cache-lines)과 [메모리 페이지](/hpc/cpu-cache/paging) 에 캐시하면 캐시 지역성이 향상된다는 것입니다. 명령어 [캐시 활용도](/hpc/external-memory/locality)를 높이기 위해, 자주 사용되는 코드는 자주 사용되는 코드끼리, 그렇지 않은 코드는 그렇지 않은 코드끼리 묶고, 사용되지 않는(dead) 코드는 가능한 제거해야 합니다. 이 아이디어에 대해 더 알고 싶다면, 최근 LLVM에 [합병](https://github.com/llvm/llvm-project/commit/4c106cfdf7cf7eec861ad3983a3dd9a9e8f3a8ae)된 Facebook의 [바이너리 최적화 및 레이아웃 툴](https://engineering.fb.com/2018/06/19/data-infrastructure/accelerate-large-scale-applications-with-bolt/)을 확인해보세요.

### 불일치 분기

Suppose that for some reason you need a helper function that calculates the length of an integer interval. It takes two arguments, $x$ and $y$, but for convenience, it may correspond to either $[x, y]$ or $[y, x]$, depending on which one is non-empty. In plain C, you would probably write something like this:
정수 간격의 길이를 계산하는 함수가 필요하다고 가정해보겠습니다. 이 함수는 두 인자, `x`와 `y`를 받지만, 편의상 `[x, y]` 또는 `[y, x]`의 형태로 비어있지 않은 구간을 처리할 수 있습니다. C에서는 다음과 같은 코드를 작성할 수 있을 것입니다.

```c++
int length(int x, int y) {
    if (x > y)
        return x - y;
    else
        return y - x;
}
```

하지만 x86 어셈블리에서는 구현 방식에 따라 성능에 더 큰 영향을 미칠 수 있습니다. 이를 어셈블리로 변환해봅시다.

```nasm
length:
    cmp  edi, esi
    jle  less
    ; x > y
    sub  edi, esi
    mov  eax, edi
done:
    ret
less:
    ; x <= y
    sub  esi, edi
    mov  eax, esi
    jmp  done
```

초기 C 코드가 대칭적인 반면, 어셈블리 코드는 그렇지 않습니다. 이로 인해 흥미로운 특성이 발생하는데, 한 가지 분기가 다른 분기보다 더 빠르게 실행될 수 있다는 점입니다. 예를 들어, `x > y`일 경우, `cmp`와 `ret` 사이의 5개의 명령어만 실행됩니다. 이 명령어들은 함수가 정렬되어 있다면 한 번에 모두 fetch될 것입니다. 반면, `x <= y`인 경우에는 두 번의 점프가 추가로 필요합니다. 

`x > y`가 거의 발생하지 않는다고 가정하면, 이 경우는 예외 상황에 가까우며, 우리는 단순히 `x`와 `y`를 바꿔버리면 됩니다.

```c++
int length(int x, int y) {
    if (x > y)
        swap(x, y);
    return y - x;
}
```

어셈블리는 다음과 같이 변환됩니다. 이는 보통 `if`문에서 `else` 없이 조건을 처리하는 방식입니다.

```nasm
length:
    cmp  edi, esi
    jle  normal     ; if x <= y, no swap is needed, and we can skip the xchg
    xchg edi, esi
normal:
    sub  esi, edi
    mov  eax, esi
    ret
```

이제 명령어 길이는 6으로 줄어들었고, 이전에는 8이었습니다. 하지만 여전히 최적화가 충분하지 않습니다. `x > y`가 결코 발생하지 않는다고 가정하면, `xchg edi, esi` 명령어를 불러오는 부분이 낭비가 됩니다. 이를 해결하려면 일반적인 실행 경로에서 이 부분을 빼면 됩니다.

```nasm
length:
    cmp  edi, esi
    jg   swap
normal:
    sub  esi, edi
    mov  eax, esi
    ret
swap:
    xchg edi, esi
    jmp normal
```

이 기법은 예외 상황을 처리할 때 유용합니다. 고수준 언어에서는 컴파일러에 특정 분기가 다른 분기보다 더 자주 실행될 것이라는 [힌트](/hpc/compilation/situational)힌트를 줄 수 있습니다.

```c++
int length(int x, int y) {
    if (x > y) [[unlikely]]
        swap(x, y);
    return y - x;
}
```

이 최적화는 분기가 매우 드물게 실행될 때만 유리합니다. 그렇지 않은 경우, 코드 레이아웃보다 더 중요한 [다른 요소](/hpc/pipelining/hazards)들이 있습니다. 이로 인해 컴파일러는 분기를 피하고, 대신 특수한 조건부 이동 명령어로  대체합니다. 이는 대체로 삼항 연산자 `x > y ? y - x : x - y`나 `abs(x - y)` 호출에 해당합니다.

```nasm
length:
    mov   edx, edi
    mov   eax, esi
    sub   edx, esi
    sub   eax, edi
    cmp   edi, esi
    cmovg eax, edx  ; "mov if edi > esi"
    ret
```


분기를 제거하는 것은 중요한 주제이며, 이를 [다음 장](/hpc/pipelining/branching)에서 이를 좀 더 자세히 다룰 것입니다.

<!--

This architecture peculiarity

When you have branches in your code, there is a variability in how you can place their instruction sequences in the memory — and surprisingly, .

```nasm
length:
    mov   edx, edi
    mov   eax, esi
    sub   edx, esi
    sub   eax, edi
    cmp   edi, esi
    cmovg eax, edx  ; "mov if edi > esi"
    ret
```

Granted that `x > y` never or almost never happens, the branchy variant will be 2 instructions shorter.

https://godbolt.org/z/bb3a3ahdE

(The compiler can't optimize it because it's technically [not allowed to](/hpc/compilation/contracts): despite `y - x` being valid, `x - y` could over/underflow, causing undefined behavior. Although fully correct, I guess the compiler just doesn't date executing it.)

We will spend [much of the next chapter](/hpc/pipelining/branching) discussing it in more detail.

You don't have to decode the things you are not going to execute anyway.

In general, you want to, and put rarely executed code away — even in the case of if-without-else patterns.

-->
