---
title: Functions and Recursion
weight: 3
published: true
---

어셈블리에서 함수를 호출하려면, 함수의 시작 지점으로 [점프](../loops)한 뒤 다시 되돌아와야 합니다. 하지만 이 과정에서 두 가지 중요한 문제가 발생합니다.

1. 호출자와 피호출자가 같은 레지스터를 사용하고 있다면, 피호출자의 데이터는 어떻게 되는가?
2. "되돌아올 위치"는 어디인가?

이 두 가지 문제는, 함수를 호출하기 전에 함수로부터 돌아오는 데 필요한 모든 정보를 저장할 수 있는 메모리상의 전용 공간을 마련함으로써 해결할 수 있습니다. 이 공간을 **스택(stack)**이라고 부릅니다.

### 스택

The hardware stack works the same way software stacks do and is similarly implemented as just two pointers:
하드웨어 스택은 소프트웨어 스택과 동일한 방식으로 작동하며, 두 개의 포인터만으로 구현된다는 점에서도 유사합니다.

- **베이스 포인터**는 스택의 시작 지점을 나타내며, 일반적으로 `rbp`에 저장됩니다.
- **스택 포인터**는 스택의 마지막 원소를 가리키며, 일반적으로 `rsp`에 저장됩니다.

함수를 호출해야 할 때는, 모든 지역 변수를 스택에 푸시하고(레지스터가 부족할 때 등 다른 상황에서도 사용될 수 있습니다), 현재 명령어 포인터를 스택에 저장한 뒤 함수의 시작 지점으로 점프합니다. 함수에서 반환할 때는 스택 맨 위에 저장된 포인터를 참조하여 해당 위치로 점프하고, 스택에 저장된 변수들을 다시 레지스터로 복원합니다.

<!--

Function parameters and local variables are accessed by adding and subtracting, respectively, a constant offset from `ebp`.

ebp itself actually points to the previous frame's base pointer, which enables stack walking in a debugger and viewing other frames local variables to work. Fun things, such as stopping the program and seeing which functions are called by which.

push ebp      ; Preserve current frame pointer
mov ebp, esp  ; Create new frame pointer pointing to current stack top
sub esp, 20   ; allocate 20 bytes worth of locals on stack.

frame pointer omission optimization which you can enable will actually eliminate this and use ebp as another register and access locals directly off of esp, but this makes debugging a bit more difficult since the debugger can no longer directly access the stack frames of earlier function calls.

When a function starts, it executed a *function prologue*: saves the previous base pointer on the stack and sets `rbp = rsp`.

-->

이러한 과정을 일반적인 메모리 명령어와 점프 명령어만으로 구현할 수는 있지만, 너무 자주 사용되기 때문에 이를 위한 네 가지 특수 명령어가 존재합니다.

- `push` : 현재 스택 포인터 위치에 데이터를 저장하고 포인터를 감소시킵니다.
- `pop` : 현재 스택 포인터 위치에서 데이터를 읽고 포인터를 증가시킵니다.
- `call` : 다음 명령어의 주소(복귀 주소)를 스택에 저장하고, 특정 레이블로 점프합니다.
- `ret` : 스택 맨 위에 있는 복귀 주소를 읽고, 해당 위치로 점프합니다.

이 명령어들은 두 개의 명령어로 이뤄진 동작을 하나로 묶은 것에 불과하므로, 이것들이 실제 하드웨어 명령어가 아니었다면 문법적 설탕(syntactic sugar)이라고 불렸을 것입니다.

```nasm
; "push rax"
sub rsp, 8
mov QWORD PTR[rsp], rax

; "pop rax"
mov rax, QWORD PTR[rsp]
add rsp, 8

; "call func"
push rip ; <- instruction pointer (although accessing it like that is probably illegal)
jmp func

; "ret"
pop  rcx ; <- choose any unused register
jmp rcx
```

`rbp`와 `rsp` 사이의 메모리 영역은 스택 프레임이라고 불리며, 함수의 지역 변수들이 일반적으로 저장되는 공간입니다. 이 영역은 프로그램 시작 시 사전 할당되며, 기본 크기(리눅스에서는 8MB)보다 더 많은 데이터를 푸시하면 스택 오버플로우가 발생할 수 있습니다. 현대 운영체제는 실제 메모리 페이지를 주소 공간에 읽거나 쓰기를 시도하기 전까지 할당하지 않기 때문에, 스택 크기를 매우 크게 지정해도 문제가 없습니다. 이 크기는 모든 프로그램이 반드시 사용하는 고정된 양이 아니라, 사용할 수 있는 스택 메모리의 한계에 더 가깝습니다.

<!--

It is convenient to save the frame pointer `rbp` at the beginning of a function and replace it with `rsp` — this way, when leaving a function, you could just restore `rbp` and forget about all its local variables. This sequence is called *function prologue* and usually looks somewhat like that (which is often optimized away by the compiler):

```nasm
push rbp     ; preserve the current frame pointer
mov rbp, rsp ; create a new frame pointer pointing to the current top of the stack
sub rsp, 20  ; allocate 20 bytes worth of locals on stack
```

-->

<!--
The memory region dedicated for stack memory (called *stack frame*) is not any different from any other memory region. It is allocated on the start of the program. You could also do tricky stuff, such as 

Functions execute a *prologue* which usually looks somewhat like that:

```nasm
push rbp     ; preserve the current frame pointer
mov rbp, rsp ; create a new frame pointer pointing to the current top of the stack
sub rsp, 20  ; allocate 20 bytes worth of locals on stack
```

Note that the data in the stack is written top-to-bottom. This is just a convention: it could be the other way around. When you need to "leave" a function or a visibility scope such as the body of an `if` or a `for`, you can just increase the stack pointer.

-->

### 호출 규약

컴파일러와 운영체제를 개발한 사람들은 결국 함수의 작성과 호출 방식에 대한 [규약](https://wiki.osdev.org/Calling_Conventions) 을 만들어냈습니다. 이러한 규약 덕분에 컴파일을 여러 단위로 나누거나, 이미 컴파일된 라이브러리를 재사용하거나, 심지어 서로 다른 프로그래밍 언어로 작성된 코드를 함께 사용하는 것과 같은 중요한 [소프트웨어 공학적 혁신](/hpc/compilation/stages/)들이 가능해졌습니다.

다음은 C 코드의 예시입니다.

```c
int square(int x) {
    return x * x;
}

int distance(int x, int y) {
    return square(x) + square(y);
}
```

<!--

When compiled without any optimization flags, it produces the following assembly:

```nasm
square:
    push    rbp
    mov     rbp, rsp
    mov     DWORD PTR [rbp-4], edi
    mov     eax, DWORD PTR [rbp-4]
    imul    eax, eax
    pop     rbp
    ret
length:
    push    rbp
    mov     rbp, rsp
    push    rbx
    sub     rsp, 8
    mov     DWORD PTR [rbp-12], edi
    mov     DWORD PTR [rbp-16], esi
    mov     eax, DWORD PTR [rbp-12]
    mov     edi, eax
    call    square
    mov     ebx, eax
    mov     eax, DWORD PTR [rbp-16]
    mov     edi, eax
    call    square
    add     eax, ebx
    mov     rbx, QWORD PTR [rbp-8]
    leave
    ret
```
-->

규약상, 함수는 인자를 `rdi`, `rsi`, `rdx`, `rcx`, `r8`, `r9`순으로 전달받으며, 인자가 이보다 많을 경우 나머지는 스택에 저장합니다. 반환 값은 `rax`에 저장되며, 이후 `ret` 명령어로 복귀합니다. 따라서 단일 인자를 받는 단순한 함수인 `square`는 다음과 같이 구현할 수 있습니다.

```nasm
square:             ; x = edi, ret = eax
    imul edi, edi
    mov  eax, edi
    ret
```

`distance` 함수에서 이를 호출할 때마다, 지역 변수들을 보존하는 데 약간의 처리를 해줘야 합니다.

```nasm
distance:           ; x = rdi/edi, y = rsi/esi, ret = rax/eax
    push rdi
    push rsi
    call square     ; eax = square(x)
    pop  rsi
    pop  rdi

    mov  ebx, eax   ; save x^2
    mov  rdi, rsi   ; move new x=y

    push rdi
    push rsi
    call square     ; eax = square(x=y)
    pop  rsi
    pop  rdi

    add  eax, ebx   ; x^2 + y^2
    ret
```

더 많은 사항들이 있지만, 여기서는 다루지 않겠습니다. 이 책은 성능 최적화에 관한 책이며, 함수 호출을 다루는 가장 좋은 방법은 애초에 호출하지 않는 것이기 때문입니다.

### 인라이닝

데이터를 스택에 저장하고 불러오는 작업은 작은 함수들에서는 눈에 띄는 오버헤드를 유발합니다. 이렇게 해야 하는 이유는 일반적으로 피호출 함수가 지역 변수를 저장해 둔 레지스터를 수정할지 아닐지 알 수 없기 때문입니다. 하지만 `square` 함수의 코드에 접근할 수 없다면, 변경되지 않을 것이라고 확신할 수 있는 레지스터에 데이터를 저장(stashing)하여 이 문제를 해결할 수 있습니다.

```nasm
distance:
    call square
    mov  ebx, eax
    mov  edi, esi
    call square
    add  eax, ebx
    ret
```

이 방법이 더 낫지만, 여전히 스택 메모리에 암묵적으로 접근하고 있습니다. 각 함수 호출마다 명령어 포인터를 push하고 pop해야 하기 때문입니다. 이런 단순한 경우에는, 피호출 함수의 코드를 호출자에 직접 붙이고 레지스터 충돌을 해결하는 방식으로 함수 호출을 인라인할 수 있습니다.

```nasm
distance:
    imul edi, edi       ; edi = x^2
    imul esi, esi       ; esi = y^2
    add  edi, esi
    mov  eax, edi       ; there is no "add eax, edi, esi", so we need a separate mov
    ret
```

이는 최적화된 컴파일러가 이 코드를 처리했을 때 생성하는 결과와 매우 유사합니다. 다만 결과 기계어 코드를 몇 바이트 줄이기 위해 [lea 트릭](../assembly)을 사용합니다.

```nasm
distance:
    imul edi, edi       ; edi = x^2
    imul esi, esi       ; esi = y^2
    lea  eax, [rdi+rsi] ; eax = x^2 + y^2
    ret
```

이런 상황에서는 함수 인라이닝이 명백한 이점을 가지며, 대부분의 경우 컴파일러가 이를 [자동으로](/hpc/compilation/situational)수행합니다. 하지만 항상 그런 것은 아니며, 이에 대해서는 [조금 뒤](../layout)에 더 자세히 다룰 것입니다.

### 꼬리 호출 제거

함수가 다른 함수를 호출하지 않거나, 최소한 이러한 호출이 재귀가 아닌 경우에는 인라이닝을 비교적 쉽게 수행할 수 있습니다. 이제 더 복잡한 예제로 넘어가봅시다. 아래는 팩토리얼을 재귀적으로 계산하는 예시입니다.

```cpp
int factorial(int n) {
    if (n == 0)
        return 1;
    return factorial(n - 1) * n;
}
```

이에 해당하는 어셈블리 코드는 다음과 같습니다.

```nasm
; n = edi, ret = eax
factorial:
    test edi, edi   ; test if a value is zero
    jne  nonzero    ; (the machine code of "cmp rax, 0" would be one byte longer)
    mov  eax, 1     ; return 1
    ret
nonzero:
    push edi        ; save n to use later in multiplication
    sub  edi, 1
    call factorial  ; call f(n - 1)
    pop  edi
    imul eax, edi
    ret
```

함수가 재귀적이라 하더라도 구조를 바꿔 호출을 제거할 수 있는 경우가 많습니다. 특히 함수가 꼬리 재귀일 경우, 즉 재귀 호출 직후 바로 반환하는 경우에 해당합니다. 호출 이후에 추가 동작이 필요 없기 때문에, 스택에 무언가를 저장할 필요가 없고 재귀 호출을 안전하게 함수 시작 지점으로의 점프로 바꿀 수 있습니다. 이는 사실상 해당 함수를 반복문으로 바꾸는 것입니다.

`factorial`함수를 꼬리 재귀형으로 만들기 위해, 현재까지의 곱셈 결과를 인자로 전달할 수 있습니다.

```cpp
int factorial(int n, int p = 1) {
    if (n == 0)
        return p;
    return factorial(n - 1, p * n);
}
```

이 함수는 다음과 같이 반복문으로 쉽게 변환할 수 있습니다.

```nasm
; assuming n > 0
factorial:
    mov  eax, 1
loop:
    imul eax, edi
    sub  edi, 1
    jne  loop
    ret
```

재귀가 느려지는 주된 이유는 스택에 데이터를 읽고 쓰는 작업이 필요하기 때문입니다. 반면 반복문이나 꼬리 재귀 알고리즘은 그런 작업이 없습니다. 이 개념은 반복문 없이 함수만 사용하는 함수형 프로그래밍에서 특히 중요합니다. 꼬리 호출 제거가 없다면, 함수형 프로그래밍은 실행하는 데 훨씬 더 많은 시간과 메모리를 소비하게 될 것입니다.
