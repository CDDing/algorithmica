---
title: Programming Languages
aliases:
  - /hpc/analyzing-performance
weight: 2
published: true
---

만약 이 책을 읽고 있다면, 컴퓨터과학의 여정 어딘가에서 처음으로 코드의 효율성에 대해 신경쓰기 시작한 순간을 경험하였을 것입니다.

필자의 경우 고등학생 시절이었습니다. 웹사이트를 만들고 유용한 ㅡㅍ로그래밍을 하는 것만으로는 대학에 진학할 수 없다는 사실을 깨닫고, 알고리즘 프로그래밍 올림피아드라는 흥미로운 세계에 입문하였습니다. 저는 고등학생치고는 괜찮은 프로그래머였지만, 그전까지는 제 코드가 실행되는데 얼마나 시간이 걸리는지에 대해서는 한번도 생각해본 적이 없었습니다. 그러나 이제 상황이 달라졌습니다. 각 문제에는 엄격한 시간 제한이 걸려 있었기 때문입니다. 저는 연산 횟수를 세기 시작하였습니다. 1초에 얼마나 많은 연산을 수행할 수 있을까요?

이 질문에 답할만큼 컴퓨터 구조에 대해 많이 알고 있지는 않았습니다. 그러나 꼭 정답이 필요한 것은 아니었습니다. 일종의 경험이 있었습니다. 제 사고과정은 이랬습니다. "2 ~ 3GHz는 초당 20억에서 30억 개의 명령어를 실행한다는 의미이므로, 배열 원소에 대해 단순한 작업을 수행하는 반복문에서도 루프 카운터 증가, 루프 종료 조건 확인, 배열 인덱싱 등의 작업이 필요하니, 유용한 연산 하나당 3 ~ 5개의 추가 명령어가 필요하겠구나." 그래서 저는 대략적인 추정치로 $5\cdot 10^8$를 사용했습니다. 이 중에서 사실인 내용은 거의 없지만, 알고리즘이 수행하는 연산 수를 세고 이 숫자로 나누는 방식은 제 경우에 꽤 유용한 경험칙이었습니다.

물론 실제 정답은 훨씬 더 복잡하며, 어떤 종류의 "연산"을 상정하느냐에 따라 크게 달라집니다. [포인터 추적(pointer chasing)](/hpc/cpu-cache/latency)과 같은 경우에는 $10^7$까지 낮아질 수 있으며, [SIMD 가속](/hpc/simd)이 적용된 선형대수 연산의 경우 $10^{11}$까지 도달할 수 있습니다. 이러한 극적인 차이를 보여주기 위해, 우리는 다양한 언어로 구현된 행렬 곱셈 사례를 분석하고, 컴퓨터가 이를 실제로 어떻게 실행하는지를 더 깊이 살펴볼 것입니다. 

<!--

Because of this logic, and also because of the [computation model](../) postulated in CS 101, many programmers have a misconception that computers can execute a certain number of "operations" per second, and that using different programming languages has some sort of [multiplier effect](https://benchmarksgame-team.pages.debian.net/benchmarksgame/index.html) on that number:

- "you can execute about $5 \cdot 10^8$ operations per second on this machine,"
- "C is 2 times faster than Java,"
- "Python is 100x slower than C++."

-->

## 언어의 종류

<!--

Processors can be thought of as *state machines*. They keep their *state* in several fixed-length *registers*, one of which, the instruction pointer, indicates a memory location of the next instruction to be read and executed. This instruction somehow modifies the registers and moves the instruction pointer to the next instruction to be executed, and so on.

These instructions — called *machine code* — are binary encoded, quirky and very difficult to work with, so no sane person writes them directly nowadays. Instead, we use higher-level programming languages and employ alternative means to feed instructions to the processor.

-->

가장 낮은 수준에서 컴퓨터는 CPU를 제어하기 위해 이진수로 인코딩된 명령어로 구성된 기계어를 실행합니다. 이러한 명령어는 매우 구체적이고 까다로우며, 이를 다루기 위해서는 상당한 지적 노력이 필요합니다. 그래서 컴퓨터가 발명된 이후, 사람들이 가장 먼저 한 일 중 하나는 바로 프로그래밍 언어를 만드는 것이었습니다. 이는 컴퓨터의 동작 방식을 어느 정도 추상화 함으로써 프로그래밍 과정을 단순화하기 위함이었습니다.

프로그래밍 언어는 본질적으로 하나의 인터페이스입니다. 프로그래밍 언어로 작성된 프로그램은 좀 더 보기 좋고 추상화된 고수준 표현일 뿐이며, 결국 CPU에서 실행되기 위해서는 기계어로 변환되어야 합니다. 그리고 이를 수행하는 방식에는 여러 가지가 있습니다.

- 프로그래머의 관점에서는 언어를 두 가지로 나눌 수 있습니다. 실행 전에 미리 처리하는 컴파일 언어와, 실행 중에 인터프리터라는 별도의 프로그램을 통해 처리되는 인터프리터 언어입니다.
- 컴퓨터의 관점에서도 언어는 두 가지로 구분됩니다. 기계어를 직접 실행하는 네이티브 언어와, 실행을 위해 일종의 런타임 환경에 의존하는 매니지드 언어입니다.

기계어를 인터프리터로 실행하는 것은 논리적으로 맞지 않기 때문에, 실제로 존재하는 언어 유형은 다음의 세 가지로 정리할 수 있습니다.

- 인터프리터 언어(예: Python, JavaScript, Ruby)
- 런타임을 사용하는 컴파일 언어(예: Java, C#, Erlang 및 이들의 VM에서 동작하는 Scala, F#, Elixir 등)
- 네이티브 컴파일 언어(C, Go, Rust)

컴퓨터 프로그램을 실행하는 데 있어 정답은 없습니다. 각각의 방식은 장단점이 존재합니다. 인터프리터와 가상머신은 유연성을 제공하며, 동적 타이핑, 런타임 코드 변경, 자동 메모리 관리 등 다양한 고수준 프로그래밍 기능을 가능하게 해줍니다. 하지만 이러한 장점들은 필연적으로 일부 성능 손실이라는 대가를 수반합니다. 이제 이러한 트레이드에오프에 대해 이야기해보겠습니다.

### 인터프리터 언어

여기 순수 Python으로 작성한 $1024 \times 1024$ 크기의 행렬 곱 예제가 있습니다.

```python
import time
import random

n = 1024

a = [[random.random()
      for row in range(n)]
      for col in range(n)]

b = [[random.random()
      for row in range(n)]
      for col in range(n)]

c = [[0
      for row in range(n)]
      for col in range(n)]

start = time.time()

for i in range(n):
    for j in range(n):
        for k in range(n):
            c[i][j] += a[i][k] * b[k][j]

duration = time.time() - start
print(duration)
```

이 코드는 630초, 즉 10분 이상이 걸립니다!

이 숫자의 의미를 살펴봅시다. 이를 실행한 CPU의 클럭 주파수는 1.4GHz로, 이는 초당 $1.4\cdot 10^9$사이클을 수행한다는 뜻이며, 전체 연산에는 거의 $10^{15}$ 사이클이 소모됩니다. 반복문의 가장 안쪽에서 곱셈 하나를 수행하는 데 약 880 사이클이 소요됩니다.

Python이 어떤 작업을 수행해야 하는지를 생각해 보면, 이 결과는 그리 놀랍지 않습니다.

- 표현식 `c[i][j] += a[i][k] * b[k][j]`를 파싱합니다.
- `a`, `b`, `c`가 무엇인지 확인하고, 이들의 타입 정보를 알아내기 위해 특수한 해시 테이블에서 이름을 조회합니다.
- `a`가 리스트라는 것을 인식하고 `[]` 연산자를 불러오며, `a[i]`에 접근해 포인터를 가져옵니다. 이것 역시 리스트이므로, 다시 `[]` 연산자를 호출하여 `a[i][k]`에 접근하고 실제 원소를 가져옵니다.
- 해당 원소의 타입을 확인해 `float`임을 알아내고, 곱셈 연산을 수행하는 메서드를 가져옵니다.
- `b`와 `c`에도 동일한 과정을 거친 후, 결과를 `c[i][j]`에 더해 저장합니다.

물론 Python처럼 널리 사용되는 언어의 인터프리터는 매우 잘 최적화되어 있어서, 동일한 코드가 반복 실행될 경우 일부 단계를 생략할 수 있습니다. 하지만 언어 설계 상 불가피한 오버헤드가 여전히 존재합니다. 만약 타입 확인이나 포인터 추적 등의 과정을 제거한다면, 곱셈당 사이클 수가 1에 가까워질 수 있고, 혹은 네이티브 곱셈과 거의 같은 수준의 성능을 낼 수 있습니다.

### 매니지드 언어

동일한 행렬곱 연산이지만, Java로 구현된 버전입니다.

```java
import java.util.Random;

public class Matmul {
    static int n = 1024;
    static double[][] a = new double[n][n];
    static double[][] b = new double[n][n];
    static double[][] c = new double[n][n];

    public static void main(String[] args) {
        Random rand = new Random();

        for (int i = 0; i < n; i++) {
            for (int j = 0; j < n; j++) {
                a[i][j] = rand.nextDouble();
                b[i][j] = rand.nextDouble();
                c[i][j] = 0;
            }
        }

        long start = System.nanoTime();

        for (int i = 0; i < n; i++)
            for (int j = 0; j < n; j++)
                for (int k = 0; k < n; k++)
                    c[i][j] += a[i][k] * b[k][j];
                
        double diff = (System.nanoTime() - start) * 1e-9;
        System.out.println(diff);
    }
}
```

이제는 실행하는데 10초가 걸립니다. 곱셈당 약 13개의 CPU 사이클이 소모되어 Python보다 63배나 빠릅니다. `b`의 원소들을 메모리에서 비순차적으로 읽어야 한다는 점을 고려하면, 이 실행 시간은 대체로 예상 가능한 수준입니다.

Java는 컴파일 언어지만 네이티브 언어는 아닙니다. 프로그램은 먼저 바이트코드로 컴파일된 후, 가상 머신(JVM)에 의해 실행됩니다. 더 높은 성능을 위해, 가장 안쪽의 `for` 반복문처럼 자주 실행되는 코드들은 런타임 중에 기계어로 컴파일되어 거의 오버헤드 없이 실행됩니다. 이러한 기법을 Just In Time Compilation(JIT 컴파일)이라고 합니다.

JIT 컴파일은 언어 자체의 기능이 아니라, 구현체에 따른 기능입니다. 파이썬에도 JIT 컴파일이 적용된 버전인 [PyPy](https://www.pypy.org/)가 있으며, 위 코드를 수정 없이 약 12초 만에 실행할 수 있습니다.

### 컴파일 언어

이제 C로 바꿔봅시다.

```cpp
#include <stdlib.h>
#include <stdio.h>
#include <time.h>

#define n 1024
double a[n][n], b[n][n], c[n][n];

int main() {
    for (int i = 0; i < n; i++) {
        for (int j = 0; j < n; j++) {
            a[i][j] = (double) rand() / RAND_MAX;
            b[i][j] = (double) rand() / RAND_MAX;
        }
    }

    clock_t start = clock();

    for (int i = 0; i < n; i++)
        for (int j = 0; j < n; j++)
            for (int k = 0; k < n; k++)
                c[i][j] += a[i][k] * b[k][j];

    float seconds = (float) (clock() - start) / CLOCKS_PER_SEC;
    printf("%.4f\n", seconds);
    
    return 0;
}
```

`gcc -O3`으로 컴파일했을 때 9초가 걸립니다.

그다지 큰 성능 향상처럼 보이지 않을 수 있습니다. Java나 Pypy보다 1~3초 정도 빠른 것은 JIT 컴파일에 걸리는 시간 때문이라고 볼 수 있습니다. 하지만 우리는 아직 C 컴파일러 생태계의 강점을 제대로 활용하지 않았습니다. 만약 `-march=native`와 `-ffast-math` 플래그를 추가한다면, 실행 시간이 갑자기 0.6초로 줄어듭니다!

여기서 우리가 한 일은 컴파일러에게 현재 사용 중인 CPU의 정확한 모델(`-march=native`)을 [알려주고](/hpc/compilation/flags/), [부동 소수점 연산](/hpc/arithmetic/float)을 재배열 할 수 있는 자유(`-ffast-math`)를 준 것 뿐입니다. 그러자 컴파일러는 이를 활용해 [벡터화](/hpc/simd)를 수행했고, 그 결과로 속도가 크게 향상된 것입니다.

PyPy와 Java같은 JIT 컴파일러에서도 소스 코드를 크게 변경하지 않고 이와 비슷한 성능을 얻는 것이 불가능한 것은 아니지만, 네이티브 코드로 직접 컴파일되는 언어에서 이를 달성하는 것이 확실히 더 쉽습니다.

### BLAS

마지막으로는 전문가에 의해 최적화된 구현이 어떤 성능을 낼 수 있는지 살펴봅시다. 널리 사용되는 최적화된 선형대수 라이브러리인 [OpenBLAS](https://www.openblas.net/)를 테스트해보겠습니다. 이를 가장 쉽게 사용하는 방법은 Python으로 돌아가 `numpy`에서 호출하는 것입니다.

```python
import time
import numpy as np

n = 1024

a = np.random.rand(n, n)
b = np.random.rand(n, n)

start = time.time()

c = np.dot(a, b)

duration = time.time() - start
print(duration)
```

이제 약 0.12초밖에 걸리지 않습니다. 벡터화된 C 버전보다 5배 빠르고 초기 Python 버전보다 5250배 빠릅니다.

이런 극적인 성능 향상은 흔치 않습니다. 지금은 이것이 어떻게 가능한지는 자세히 설명하지 않겠습니다. OpenBLAS의 밀집 행렬 곱셈 구현은 일반적으로 각 아키텍처에 맞춰 개별적으로 작성된 [5000줄 짜리 수작업 어셈블리 코드](https://github.com/xianyi/OpenBLAS/blob/develop/kernel/x86_64/dgemm_kernel_16x2_haswell.S)로 구성되어 있습니다. 이후 챕터에서 관련 기술들을 하나씩 설명한 뒤, 이 예제로 [돌아와](/hpc/algorithms/matmul) 40줄 이하의 C 코드만으로 BLAS 수준의 구현을 만들어볼 것입니다.

### 요점

여기서 얻을 수 있는 핵심 교훈은, 네이티브 저수준 언어를 사용한다고 해서 반드시 성능이 보장되는 것은 아니지만, 성능을 제어할 수 있는 권한은 제공한다는 점입니다.

1초에 N개의 연산이라는 단순화된 개념 외에도, 많은 프로그래머들이 프로그래밍 언어가 성능에 일정한 배수를 곱하는 것처럼 오해합니다. 하지만 성능을 기준으로 언어를 [비교하는 것](https://benchmarksgame-team.pages.debian.net/benchmarksgame/index.html)은 큰 의미가 없습니다. 프로그래밍 언어란 본질적으로, 편리한 추상화를 제공하는 대신 성능 제어에 대한 일부 권한을 포기하게 만드는 도구에 불과합니다. 어떤 실행 환경에서든, 하드웨어가 제공하는 기회를 얼마나 잘 활용하는지는 결국 프로그래머의 역량에 달려 있습니다.
