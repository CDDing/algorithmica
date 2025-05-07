---
title: Loops and Conditionals
weight: 2
---

약간 더 복잡한 예제를 살펴보겠습니다.

```nasm
loop:
    add  edx, DWORD PTR [rax]
    add  rax, 4
    cmp  rax, rcx
    jne  loop
```

이 코드는 단순한 `for` 반복문처럼 동작하여, 32비트 정수 배열의 합을 계산합니다.

루프의 본문인 `add edx, DWORD PTR [rax]` 명령어는, 반복자 역할을 하는 `rax`가 가리키는 메모리에서 값을 읽어와 누산기인 `edx`에 더합니다. 그 다음 `add rax, 4`를 통해 반복자를 4바이트(정수 하나)만큼 앞으로 이동시킵니다. 그리고 조금 더 복잡한 동작이 이어집니다.

### 점프

어셈블리에는 if, for, 함수 같은 고수준 언어의 흐름 제어 구조가 없습니다. 대신 저수준 프로그래밍 세계에서 흔히 점프(jump) 혹은 `goto`라고 알려진 명령어만 존재합니다.

점프는 명령어 포인터를 피연산자로 지정된 위치로 이동시킵니다. 이 위치는 메모리 상의 절대 주소일 수도 있고, 현재 위치 기준의 상대 주소거나, 심지어 [실행 중에 계산된 주소](../indirect)일 수도 있습니다. 이런 주소들을 직접 관리하는 번거로움을 피하기 위해, 어떤 명령어든 문자열 뒤에 `:`를 붙여 마킹할 수 있습니다. 그러면 해당 문자열은 라벨로 동작하여, 기계어로 변환될 때 이 명령어의 상대 주소로 치환됩니다.

라벨은 임의의 문자열로 지정할 수 있지만, 컴파일러는 이름을 지을 때 보통 창의성을 발휘하지 않고 [일반적으로](https://godbolt.org/z/T45x8GKa5) 일반적으로 소스 코드의 줄 번호나 함수 이름과 시그니처를 사용합니다.

**무조건**적인 점프인 `jmp`는 `while(true)`와 같은 무한 루프를 구현하거나 프로그램의 여러 부분을 이어붙일 때만 사용됩니다. 실제 제어 흐름을 구현할 때는 **조건부** 점프 계열 명령어들이 사용됩니다.

이러한 조건들이 어딘가에서 `bool` 값으로 계산되어 조건 점프 명령어의 피연산자로 전달된다고 생각하는 것은 그럴듯합니다. 실제로 고수준 언어에서는 그렇게 동작합니다. 하지만 하드웨어 수준에서는 그렇지 않습니다. 조건 연산은 `FLAGS`라는 특수한 레지스터를 사용하는데, 이 레지스터는 먼저 어떤 종류의 비교나 체크를 수행하는 명령어를 실행해서 값을 설정해야 합니다.

우리 예제에서는 `cmp rax, rcx`가 반복자 `rax`와 배열 끝을 가리키는 포인터 `rcx`를 비교합니다. 이 명령어는 `FLAGS` 레지스터를 갱신하고, 이후 `jne loop`가 이 레지스터의 특정 비트를 확인하여 두 값이 같은지 다른지를 판단합니다. 그 결과에 따라 루프의 처음으로 점프하거나 다음 명령어로 진행하여 루프를 종료합니다.

### 루프 풀기(Loop Unrolling)

위 루프에서 눈치챘을 수도 있는 한 가지는, 단일 원소를 처리하는 데 상당한 오버헤드가 있다는 점입니다. 각 사이클마다 실제로 유용한 명령어는 단 하나뿐이고, 나머지 세 개는 반복자를 증가시키거나 작업이 끝났는지를 확인하는 데 사용됩니다.

우리가 할 수 있는 최적화 방법 중 하나는 반복을 묶어 루프를 **펼치는(unroll)** 것입니다. 이는 C 코드로 보면 다음 형태와 같습니다.

```c++
for (int i = 0; i < n; i += 4) {
    s += a[i];
    s += a[i + 1];
    s += a[i + 2];
    s += a[i + 3];
}
```

어셈블리에서는 다음과 비슷하게 나타납니다.

```nasm
loop:
    add  edx, [rax]
    add  edx, [rax+4]
    add  edx, [rax+8]
    add  edx, [rax+12]
    add  rax, 16
    cmp  rax, rsi
    jne  loop
```

이제는 4개의 유용한 명령어를 실행하기 위해 루프 제어 명령어가 3개만 필요합니다(효율성 측면에서 $\frac{1}{4}$에서 $\frac{4}{7}$로 개선되었습니다). 그리고 이 과정을 계속 확장하면 오버헤드를 거의 0에 가깝게 줄일 수 있습니다. 

하지만 실전에서는 항상 루프 풀기가 필수적인 것은 아닙니다. 현대의 프로세서는 명령어를 하나씩 순차적으로 실행하지 않고, [대기중인 명령어들의 큐](/hpc/pipelining)를 유지하면서 서로 독립적인 연산을 동시에 실행할 수 있도록 합니다.

우리의 경우도 마찬가지입니다. 루프 풀기를 한다고 해서 실제 속도가 정확히 4배 빨리지지는 않습니다. 왜냐하면 반복자 증가와 종료 조건 체크는 루프 본체와 독립적이기 때문에 동시에 스케줄링되어 실행될 수 있기 때문입니다. 하지만 여전히 컴파일러에게 적절한 수준에서 루프 풀기를 [요청](/hpc/compilation/situational)하는 것이 도움이 될 수 있습니다.

### 대안

`cmp`나 조건 분기를 위한 유사한 명령어를 명시적으로 사용할 필요는 없습니다. `FLAGS` 레지스터를 읽거나 수정하는 많은 명령어들이 있으며, 이 중 일부는 선택적인 예외 검사를 가능하게 하기도 합니다.

예를 들어, `add` 명령어는 항상 여러 플래그를 설정합니다. 이 플래그들은 결과가 0인지, 음수인지, 오버플로우 또는 언더플로우가 발생했는지 등을 나타냅니다. 이러한 매커니즘을 활용하여, 컴파일러는 종종 다음과 같은 루프를 생성합니다.

```nasm
    mov  rax, -100  ; replace 100 with the array size
loop:
    add  edx, DWORD PTR [rax + 100 + rcx]
    add  rax, 4
    jnz  loop       ; checks if the result is zero
```

이 코드는 사람이 읽기에는 다소 어렵지만, 반복되는 부분에서 명령어가 하나 적습니다. 이는 성능에 유의미한 영향을 줄 수 있습니다.

<!--

### A More Complex Example

Let's do a more complicated example.

```c++
int collatz(int n) {
    int cnt = 0;
    while (n != 1) {
        cnt++;
        if (n & 2 == 1)
            n = 3 * n + 1;
        else
            n = n / 2;
    }
    return cnt;
}
```

It is a notoriously difficult math problem that seems ridiculously simple.

Make use of [lea instruction](../assembly).

E.g., if you want to make a computational experiment [Collatz conjecture](https://en.wikipedia.org/wiki/Collatz_conjecture), you may use `lea rax, [rax + rax * 2 + 1]`, and then try to `sar` it.

Another way is to check add.

Eliminating branching. Or at least making it easier for the compiler to predict which instructions are going to be executed next.

tzcnt

cmov

Need to somehow link it to branchless programming and layout article. We now have 3 places introducing the concept.

Many other operations set something in the `FLAGS` register. For example, add often. It is useful to, and then decrement or increment it to save on instruction. Like a while loop:

```
while (n--) {
    // ...
}
```

There is an important "conditional move" operation.

-->
