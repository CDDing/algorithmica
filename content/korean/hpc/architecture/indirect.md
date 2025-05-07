---
title: Indirect Branching
weight: 4
---

어셈블리에서는 모든 레이블이 (절대 또는 상대) 주소로 변환된 후, 점프 명령어에 인코딩됩니다.

또한, 상수가 아닌 값이 저장된 레지스터를 이용해 점프할 수도 있는데, 이를 **계산된 점프(computed jump)**라고 한다.

```nasm
jmp rax
```

이 기법은 동적 언어의 구현이나 더 복잡한 제어 흐름 구조와 관련된 흥미로운 용도로 활용됩니다.

### 다중 분기

만약 `switch` 문이 정확히 어떤 일을 하는지 기억나지 않는다면, 아래는 미국식 성적 체계에 따른 GPA 계산 예시 서브루틴입니다.

```cpp
switch (grade) {
    case 'A':
        return 4.0;
        break;
    case 'B':
        return 3.0;
        break;
    case 'C':
        return 2.0;
        break;
    case 'D':
        return 1.0;
        break;
    case 'E':
    case 'F':
        return 0.0;
        break;
    default:
        return NAN;
}
```

개인적으로는 교육적인 목적이 아닌 경우에 `switch`문을 마지막으로 사용했던 시점을 기억하지 못합니다. 일반적으로 `switch`문은 일련의 "if, else if, else if..." 구조와 동일하며, 이러한 이유로 많은 언어에서는 이 문법을 지원하지 않기도 합니다. 그럼에도 불구하고 `switch`와 같은 제어 흐름 구조는 파서(parse), 인터프리터(interpreter), 기타 상태 기계(state machine)을 구현할 때 중요한 역할을 합니다. 이들 구조는 보통 `while(true)` 루프 안에 `switch(state)`문을 포함하는 형태로 구성됩니다.

변수가 가질 수 있는 값의 범위를 우리가 제어할 수 있는 경우, 계산된 점프를 활용한 다음과 같은 트릭을 사용할 수 있습니다. 조건 분기를 N개 만드는 대신, 점프 대상의 포인터 또는 오프셋을 담은 분기 테이블(branch table)을 만들고, `state` 변수를 $[0, n)$ 범위의 인덱스로 사용해 해당 위치로 점프하는 방식입니다.

컴파일러는 변수 값이 조밀하게 분포되어 있을 경우(꼭 연속적일 필요는 없지만, 테이블에 공백이 있어도 감수할 만한 경우) 이 기법을 사용합니다. 또한, 이 구조는 **계산된 goto**를 명시적으로 구현하여 사용할 수도 있습니다.

```cpp
void weather_in_russia(int season) {
    static const void* table[] = {&&winter, &&spring, &&summer, &&fall};
    goto *table[season];

    winter:
        printf("Freezing\n");
        return;
    spring:
        printf("Dirty\n");
        return;
    summer:
        printf("Dry\n");
        return;
    fall:
        printf("Windy\n");
        return;
}
```

`switch` 기반 코드는 항상 컴파일러가 최적화해주지는 않기 때문에, 상태 기계 구현 시에는 `goto` 문이 직접 사용되는 경우가 많습니다. 예를 들어, `glibc`의 I/O 관련 코드에는 이러한 예시가 다수 존재합니다.

### 동적 메서드 호출

간접 분기는 런타임 다형성을 구현하는 데에도 핵심적인 역할을 합니다.

가상 함수 `.speak()`를 가진 추상 클래스 `Animal`과, 이를 상속한 `Dog`(짖음)와 `Cat`(야옹거림)이라는 두 구체적인 구현이 있는 고전적인 예제를 살펴보겠습니다.

```cpp
struct Animal {
    virtual void speak() { printf("<abstract animal sound>\n");}
};

struct Dog {
    void speak() override { printf("Bark\n"); }
};

struct Cat {
    void speak() override { printf("Meow\n"); }
};
```

우리는 어떤 동물 객체를 생성한 뒤, 그 타입을 사전에 알지 못한 채 `.speak()`를 호출하되, 해당 타입에 맞는 구현이 호출되기를 원합니다.

```c++
Dog sparkles;
Cat mittens;

Animal *catdog = (rand() & 1) ? &sparkles : &mittens;
catdog->speak();
```

이 동작을 구현하는 방법은 여러 가지가 있지만, C++은 **가상 함수 테이블**(Virtual Method Table, VTable)을 사용합니다.

Animal을 상속하는 모든 클래스에 대해, 컴파일러는 각 메서드가 동일한 길이를 갖도록 `ret` 뒤에 [채움 명령어(filler instructions)](../layout)를 삽입하여 패딩을 적용한 뒤, 그것들을 명령어 메모리 어딘가에 순차적으로 배치합니다. 그리고 각 인스턴스의 구조체에는 런타임 타입 정보(RTTI) 필드가 추가되며, 이는 해당 클래스의 가상 함수 구현이 위치한 메모리 영역을 가리키는 오프셋입니다.

가상 함수 호출 시, 구조체 인스턴스로부터 이 오프셋 필드를 가져오고, 그것을 통해 일반적인 함수 호출을 수행합니다. 모든 파생 클래스의 메서드와 필드가 동일한 오프셋을 갖도록 설계되어 있기 때문에 가능한 방식입니다.

물론, 몇 가지 오버헤드가 있습니다.

- [분기 예측 실패(branch misprediction)](/hpc/pipelining)와 마찬가지로, 파이프라인 플러싱으로 인해 약 15사이클 정도의 추가 비용이 발생할 수 있습니다.
- 컴파일러는 이러한 함수 호출을 인라인 하지 못할 가능성이 큽니다.
- 클래스 크기가 몇 바이트 정도 증가하며, 이는 구현에 따라 다를 수 있습니다.
- 바이너리 크기 자체도 약간 늘어납니다.

이러한 이유로, 런타임 다형성은 대개 성능 중심 애플리케이션에서는 기피되는 경향이 있습니다.
