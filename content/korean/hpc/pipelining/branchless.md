---
title: Branchless Programming
weight: 3
published: true
---

[이전 글](../branching)에서 언급했듯이, CPU가 효과적으로 예측하지 못하는 분기는 분기 예측 실패 시 새로운 명령어를 가져오기까지 긴 파이프라인 정지가 발생하기 때문에 매우 비용이 큽니다. 이 글에서는 분기를 제거하는 방법에 대해 다룹니다.

### 예측

이전 글에서 다뤘던 동일한 사례를 계속 이어가 봅시다. 우리는 무작위 숫자로 이루어진 배열을 생성하고, 그중 50이하인 원소들의 합을 구합니다.

```c++
for (int i = 0; i < N; i++)
    a[i] = rand() % 100;

volatile int s;

for (int i = 0; i < N; i++)
    if (a[i] < 50)
        s += a[i];
```

우리의 목표는 위 코드에서 `if` 문으로 인한 분기를 제거하는 것입니다. 다음과 같이 바꿔볼 수 있습니다.

```c++
for (int i = 0; i < N; i++)
    s += (a[i] < 50) * a[i];
```

이제 이 루프는 원래 약 14사이클이 걸리던 것에 비해, 원소당 약 7 사이클만 소요됩니다. 또한 `50`을 다른 임계값으로 바꾸더라도 성능이 유지되므로, 분기 확률에 영향을 받지 않습니다.

하지만 잠깐.. 여전히 분기가 존재하는 것 아닐까요? `(a[i] < 50)`을 어셈블리에서는 어떻게 처리할까요?

어셈블리에는 Boolean 타입이 없고, 비교 결과에 따라 1이나 0을 반환하는 명령어도 없습니다. 그러나 간접적으로 다음과 같은 방식으로 계산할 수 있습니다. `(a[i] - 50) >> 31)`. 이 트릭은 [정수의 이진 표현](/hpc/arithmetic/integer)을 활용합니다. 특히 `a[i] < 50`이 음수라면, 그 결과의 최상위 비트가 1이 되므로, 이를 오른쪽 시프트 연산으로 추출할 수 있습니다.

```nasm
mov  ebx, eax   ; t = x
sub  ebx, 50    ; t -= 50
sar  ebx, 31    ; t >>= 31
imul  eax, ebx   ; x *= t
```

조금 더 복잡하지만, 이 과정을 사인 비트를 마스크로 변환하고, 곱셈 대신 비트 연산 `and`를 사용하는 방식도 있습니다. `((a[i] - 50) >> 31 - 1) & a[i]`. `imul` 명령어는 다른 명령어와 달리 3 사이클이 소요되므로, 이 방식은 전체 연산을 1사이클 더 빠르게 만듭니다.

```nasm
mov  ebx, eax   ; t = x
sub  ebx, 50    ; t -= 50
sar  ebx, 31    ; t >>= 31
; imul  eax, ebx ; x *= t
sub  ebx, 1     ; t -= 1 (causing underflow if t = 0)
and  eax, ebx   ; x &= t
```

하지만 이 최적화는 컴파일러 관점에서는 기술적으로 안전하지 않다는 점에 주의해야 합니다. 표현 가능한 가장 낮은 50개의 정수들(예: $[-2^{31}, - 2^{31} + 49]$)에 대해 언더 플로우가 발생해 잘못된 결과를 낼 수 있습니다. 물론 우리는 이 모든 값이 0과 100 사이에 있다는 것을 알고 있기 때문에 문제가 없지만, 컴파일러는 그 사실을 알지 못합니다.

실제로 컴파일러는 이러한 산술 트릭을 사용하는 대신, `cmov`라는 특수한 명령어를 선택합니다. `cmov`는 조건에 따라 값을 할당하는 명령어로, 조건은 점프와 마찬가지로 플래그 레지스터를 통해 평가됩니다.

```nasm
mov     ebx, 0      ; cmov doesn't support immediate values, so we need a zero register
cmp     eax, 50
cmovge  eax, ebx    ; eax = (eax >= 50 ? eax : ebx=0)
```

따라서 이 코드는 사실상 다음과 같은 삼항 연산자를 사용하는 것과 유사합니다.

```c++
for (int i = 0; i < N; i++)
    s += (a[i] < 50 ? a[i] : 0);
```

두 방식 모두 컴파일러에 의해 최적화되며 다음과 같은 어셈블리를 생성합니다.

```nasm
    mov     eax, 0
    mov     ecx, -4000000
loop:
    mov     esi, dword ptr [rdx + a + 4000000]  ; load a[i]
    cmp     esi, 50
    cmovge  esi, eax                            ; esi = (esi >= 50 ? esi : eax=0)
    add     dword ptr [rsp + 12], esi           ; s += esi
    add     rdx, 4
    jnz     loop                                ; "iterate while rdx is not zero"
```

이 기본적인 기법은 **예측(prediction)**이라 불리며, 대략 다음과 같은 대수적 트릭과 유사합니다.

$$
x = c \cdot a + (1 - c) \cdot b
$$

이렇게 하면 분기를 제거할 수 있지만, 두 분기와 `cmov` 자체를 계산하는 비용이 발생합니다. ">=" 분기를 평가하는 데는 비용이 들지 않기 때문에, 성능은 분기형 코드에서 ["항상 참"](../branching/#branch-prediction)인 경우와 정확히 동일합니다.

### 예측이 유의미한 경우

예측을 사용하면 [제어 위험](../hazards)을 제거하지만, 데이터 위험이 발생합니다. 여전히 파이프라인 지연이 있지만, 조금 더 저렴한 버전입니다. 예측 실패 시 전체 파이프라인을 폐기하는 대신 `cmov`만 해결되기를 기다리면 됩니다.

하지만 분기형 코드를 그대로 두는 것이 더 효율적인 경우도 많습니다. 이는 하나의 분기만 계산하는 것보다 두 분기를 계산하는 비용이 잠재적인 분기 예측 실패로 인한 패널티보다 더 클 때 해당됩니다.

예제에서는 분기가 약 75% 이상의 확률로 예측될 수 있을 때, 분기형 코드가 유리합니다.

![](../img/branchy-vs-branchless.svg)

This 75% threshold is commonly used by the compilers as a heuristic for determining whether to use the `cmov` or not. Unfortunately, this probability is usually unknown at the compile time, so it needs to be provided in one of several ways:
이 75% 임계값은 컴파일러가 `cmov`를 사용할지 말지를 결정할 때 흔히 사용하는 휴리스틱입니다. 아쉽게도, 이 확률은 컴파일 시점에서 보통 알 수 없으므로 여러 방법으로 이를 제공해야 합니다.

- 예측을 사용할지 말지 스스로 결정할 [프로필 가이드 최적화](/hpc/compilation/situational/#profile-guided-optimization)를 사용할 수 있습니다.
- 분기의 발생 가능성에 대한 힌트를 주기 위해 [likeliness 속성](../branching#hinting-likeliness-of-branches)이나 [컴파일러 고유 내장 함수](/hpc/compilation/situational)를 사용할 수 있습니다. 예를 들어 GCC의 `__builtin_expect_with_probability`나 Clang의 `__builtin_unpredictable`이 있습니다.
- 삼항 연산자나 다양한 산술 트릭을 사용해 분기형 코드를 재작성할 수 있습니다. 이는 프로그래머와 컴파일러 간의 암묵적인 계약처럼 작동합니다. 즉, 프로그래머가 코드를 이런 방식으로 작성했다면, 아마도 분기가 없는 방식으로 처리되기를 의도한 것입니다.

"옳은 방법"은 분기 힌트를 사용하는 것이지만, 아쉽게도 이에 대한 지원이 부족합니다. 현재 이 [힌트](https://bugs.llvm.org/show_bug.cgi?id=40027)는 컴파일러 백엔드가 `cmov`가 더 유리한지 결정할 때 사라지는 것처럼 보입니다. 이를 가능하게 하는 [과정](https://discourse.llvm.org/t/rfc-cmov-vs-branch-optimization/6040)이 일부 있지만, 현재로서는 컴파일러에게 분기 없는 코드를 생성하도록 강제하는 좋은 방법이 없으므로, 때때로 가장 좋은 방법은 그냥 작은 어셈블리 코드 조각을 작성하는 것입니다.

<!--

Because this is very architecture-specific.

in the absence of branch likeliness hints

While any program that uses a ternary operator is equivalent to a program that uses an `if` statement

The codes seem equivalent. My guess is that the compiler doesn't know that `s + a[i]` does not cause integer overflow.

(The compiler can't optimize it because it's technically [not allowed to](/hpc/compilation/contracts): despite `y - x` being valid, `x - y` could over/underflow, causing undefined behavior. Although fully correct, I guess the compiler just doesn't date executing it.)

Branchless computing tricks like this one are especially important in all sorts of parallel algorithms.

The `cmov` variant doesn't care about probabilities of branches. It only wins if the branch probability if 75% chance, which usually is the heuristic threshold set in compilers.

This is a legal optimization, but I guess an implicit contract has evolved between application programmers and compiler engineers that if you write a ternary operator, then you kind of telling that it is likely going to be an unpredictable branch.

The general technique is called *branchless* or *branch-free* programming. Predication is the main tool of it, but there are more complicated ways.

-->

<!--

Let's do a few more examples as an exercise.

```c++
int max(int a, int b) {
    return (a > b) * a + (a <= b) * b;
}
```

```c++
int max(int a, int b) {
    return (a > b ? a : b);
}
```


```c++
int abs(int a, int b) {
    return max(diff, -diff);
}
```

```c++
int abs(int a, int b) {
    int diff = a - b;
    return (diff < 0 ? -diff : diff);
}
```

```c++
int abs(int a) {
    return (a > 0 ? a : -a);
}
```

```c++
int abs(int a) {
    int mask = a >> 31;
    a ^= mask;
    a -= mask;
    return a;
}
```

-->

### 더 많은 예시

**문자열**을 지나치게 단순화하면, `std::string`은 힙 어딘가에 할당된 null로 끝나는 `char` 배열("C-string"이라고도 알려짐)에 대한 포인터와 문자열 크기를 담고 있는 정수로 구성됩니다.

문자열에서 흔히 사용되는 값은 빈 문자열이며, 이는 기본값입니다. 또한 이를 처리하는 관용적인 방법은 포인터에 `nullptr`을 할당하고, 문자열 크기에는 `0`을 할당한 뒤, 문자열과 관련된 모든 절차 시작 시 포인터가 null인지 또는 크기가 0인지 확인하는 것입니다.

그러나 이는 개별 분기를 요구하며, 이는 비용이 많이 듭니다(대부분의 문자열이 비어있거나 비어있지 않은 경우를 제외하고). 이 확인과 분기를 제거하기 위해 "제로 C-String"을 할당할 수 있습니다. 이는 어딘가에 할당된 0바이트로, 빈 문자열을 모두 이곳을 가리키도록 합니다. 이제 빈 문자열에 대한 모든 문자열 연산은 이 쓸모없는 0바이트를 읽어야 하지만, 이는 분기 예측 실패보다는 훨씬 저렴합니다.

**표준 이진 탐색**은 분기 없이 [구현](/hpc/data-structures/binary-search)할 수 있으며, 작은 배열(캐시에 적합한 크기)에서는 분기하는 `std::lower_bound`보다 약 4배 빠르게 작동합니다.

```c++
int lower_bound(int x) {
    int *base = t, len = n;
    while (len > 1) {
        int half = len / 2;
        base += (base[half - 1] < x) * half; // will be replaced with a "cmov"
        len -= half;
    }
    return *base;
}
```

더 복잡할 뿐만 아니라 잠재적으로 더 많은 비교를 수행할 수 있는 단점이 있으며(상수 $\lceil \log_2 n \rceil$ 대신 $\lfloor \log_2 n \rfloor$또는 $\lceil \log_2 n \rceil$가 되어야 할 경우) 미래의 메모리 읽기를 예측할 수 없기 때문에 이는 prefetching처럼 작용해, 매우 큰 배열에서는 성능 손실이 발생합니다.

일반적으로 자료구조는 암묵적 또는 명시적으로 `padding`을 추가하여 분기 없이 만들 수 있으며, 이렇게 하면 그 연산이 상수 횟수만큼 반복됩니다. 더 복잡한 예시들은 [이 글](/hpc/data-structures/binary-search)에서 확인할 수 있습니다.

<!--

The only downside of the branchless implementation is that it potentially does more memory reads: 

There are typically two ways to achieve this:

And in general, data structures can be "padded" to be made constant size or height.

That there are no substantial reasons why compilers can't do this on their own, but unfortunately this is just how it is right now.

-->

**데이터 병렬 프로그래밍**에서 분기가 없는 프로그래밍은 [SIMD](/hpc/simd) 애플리케이션에 매우 중요합니다. 왜냐하면 SIMD 애플리케이션은 애초에 분기가 없기 때문입니다.

배열 합 예제에서 `volatile` 타입 한정자를 제거하면 컴파일러가 루프를 [벡터화](/hpc/simd/auto-vectorization)할 수 있게 됩니다.

```c++
/* volatile */ int s = 0;

for (int i = 0; i < N; i++)
    if (a[i] < 50)
        s += a[i];
```

이제 이는 원소마다 약 0.3 사이클로 작동하며, 주로 메모리가 [병목](/hpc/cpu-cache/bandwidth)이 됩니다.

컴파일러는 보통 분기나 반복 간 의존성이 없는 루프를 벡터화할 수 있습니다. 그 외에도 일부 작은 편차(예: [축소](/hpc/simd/reduction) 또는 `if` 문이 단독으로 포함된 단순 루프 등)는 벡터화할 수 있습니다. 하지만 더 복잡한 것들을 벡터화하는 것은 쉽지 않은 문제로, [마스킹](/hpc/simd/masking)이나 [레지스터 내 순열](/hpc/simd/shuffling)과 같은 다양한 기법을 동원해야 할 수 있습니다.

<!--

**Binary exponentiation.** However, when it is constant

When we can iterate in small batches, [autovectorization](/hpc/simd/autovectorization) speeds it up 13x.

-->
