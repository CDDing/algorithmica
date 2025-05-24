---
title: Statistical Profiling
weight: 2
---

[Instrumentation](../instrumentation)은 프로파일링을 수행하는 데 다소 번거로운 방법이며, 특히 프로그램의 여러 작은 섹션에 관심이 있을 경우 더욱 그렇습니다. 도구를 이용해 일부 자동화할 수는 있지만, 고유의 오버헤드로 인해 고도로 세밀한 통계를 수집하는 데에는 여전히 한계가 있습니다.

보다 덜 침습적인 프로파일링 방식은 실행 중 무작위 간격으로 인터럽트를 걸어 명령어 포인터가 어디를 가리키는 지 확인하는 것입니다. 명령어 포인터가 각 함수 블록에서 멈춘 횟수는 해당 함수가 실행에 소요한 전체 시간에 대략 비례합니다. 또한 [콜 스택](/hpc/architecture/functions)을 검사함으로써 어떤 함수가 어떤 함수에 의해 호출되었는지 등의 유용한 정보도 얻을 수 있습니다.

이론적으로는 `gdb`로 프로그램을 실행한 후 무작위로 `ctrl+c`를 눌러 중단시키는 방식으로도 가능하지만, 현대의 CPU와 운영체제는 이러한 유형의 프로파일링을 위한 특수한 유틸리티를 제공합니다.

### 하드웨어 이벤트

하드웨어 성능 카운터는 특정 하드웨어 관련 활동의 횟수를 저장할 수 있는 마이크로프로세서 내 특수 레지스터입니다. 이는 단순히 활성화 신호가 연결된 이진 카운터이기 때문에 마이크로칩에 추가하는 비용이 저렴합니다.

각 성능 카운터는 회로의 큰 부분집합에 연결되어 있으며, 분기 예측 실패, 캐시 미스 등 특정 하드웨어 이벤트 발생 시 값을 증가하도록 구성할 수 있습니다. 프로그램 시작 시 카운터를 초기화하고, 실행 후 종료 시 그 값을 출력하면 해당 이벤트가 실행 중 몇 번 발생했는지를 정확히 알 수 있습니다.

여러 이벤트를 동시에 추적하려면 이벤트 간 다중화 기법을 사용할 수 있습니다. 이는 프로그램을 일정 간격으로 중단시키고 카운터 설정을 변경하는 방식입니다. 이 경우 결과는 정확하지 않고 통계적 근사치가 됩니다. 한 가지 주의할 점은 단순히 샘플링 빈도를 높인다고 해서 정확도가 증가하지는 않는다는 것입니다. 빈도 증가로 인한 성능 저하가 측정 분포를 왜곡하기 때문입니다. 따라서 다양한 통계를 수집하려면 프로그램을 더 오래 실행시켜야 합니다.

전반적으로, 이벤트 기반 통계 프로파일링은 성능 문제를 진단하는 데 가장 효과적이고 간단한 방법 중 하나입니다.

### perf 프로파일링

이러한 이벤트 샘플링 기법을 활용하는 성능 분석 도구를 통계 프로파일러(statistical profiler) 라고 합니다. 여러 도구가 있지만, 이 책에서는 주로 리눅스 커널에 포함된 통계 프로파일러인 [perf](https://perf.wiki.kernel.org/)를 사용할 것입니다. 비리눅스 시스템에서는 인텔의 [VTune](https://software.intel.com/content/www/us/en/develop/tools/oneapi/components/vtune-profiler.html#gs.cuc0ks)을 사용할 수 있습니다. Vtune은 우리의 목적에 맞는 기능을 거의 동일하게 제공하며, 무료이지만 독점 소프트웨어이며 90일마다 커뮤니티 라이선스를 갱신해야 합니다. 반면, perf는 자유 소프트웨어 입니다.

Perf는 프로그램의 실행 중 동작을 바탕으로 보고서를 생성하는 커맨드라인 애플리케이션입니다. 소스 코드 없이도 작동하며, 여러 프로세스나 운영체제와의 상호작용이 포함된 복잡한 애플리케이션도 프로파일링할 수 있습니다.

예시로는, 백만 개의 무작위 정수로 이루어진 배열을 생성하고 이를 정렬한 후, 백만 번의 이진 탐색을 수행하는 간단한 프로그램을 작성하였습니다.

```c++
void setup() {
    for (int i = 0; i < n; i++)
        a[i] = rand();
    std::sort(a, a + n);
}

int query() {
    int checksum = 0;
    for (int i = 0; i < n; i++) {
        int idx = std::lower_bound(a, a + n, rand()) - a;
        checksum += idx;
    }
    return checksum;
}
```

`g++ -O3 -march=native example.cc -o run`으로 컴파일한 후, `perf stat ./run` 명령어로 실행하면 프로그램의 실행 중에 발생한 기본적인 성능 이벤트의 횟수를 출력할 수 있습니다.

```yaml
 Performance counter stats for './run':

        646.07 msec task-clock:u               # 0.997 CPUs utilized          
             0      context-switches:u         # 0.000 K/sec                  
             0      cpu-migrations:u           # 0.000 K/sec                  
         1,096      page-faults:u              # 0.002 M/sec                  
   852,125,255      cycles:u                   # 1.319 GHz (83.35%)
    28,475,954      stalled-cycles-frontend:u  # 3.34% frontend cycles idle (83.30%)
    10,460,937      stalled-cycles-backend:u   # 1.23% backend cycles idle (83.28%)
   479,175,388      instructions:u             # 0.56  insn per cycle         
                                               # 0.06  stalled cycles per insn (83.28%)
   122,705,572      branches:u                 # 189.925 M/sec (83.32%)
    19,229,451      branch-misses:u            # 15.67% of all branches (83.47%)

   0.647801770 seconds time elapsed
   0.647278000 seconds user
   0.000000000 seconds sys
```

위 결과에서, 프로그램은 약 0.65초 동안 실행되었고, 이 동안 약 8억 5천만 사이클이 소모되었으며 평균 1.32GHz의 클럭에서 실행되었습니다. 약 4억 8천만 개의 명령어가 실행되었고, 1억 2천 2백만 개 이상의 분기가 발생했으며, 그 중 약 15.7%는 분기 예측이 실패한 것으로 나타났습니다.

`perf list` 명령어를 통해 시스템에서 지원하는 모든 이벤트를 확인할 수 있으며, `-e` 옵션을 사용하여 특정 이벤트만 선택적으로 추적할 수도 있습니다. 예를 들어 이진 탐색을 분석할 때는 주로 캐시 미스를 살펴봅니다.

```yaml
> perf stat -e cache-references,cache-misses ./run

91,002,054      cache-references:u                                          
44,991,746      cache-misses:u      # 49.440 % of all cache refs
```

`perf stat`은 전체 프로그램 실행에 대한 성능 카운터를 설정하고 총 이벤트 횟수를 알려주지만, 이벤트가 어디서 발생했는지는 알려주지 않습니다. 심지어 왜 발생했는지도 알 수 없습니다.

앞서 언급한 세계 정지(stop the world) 방식의 접근을 시도하려면 `perf record <cmd>` 명령어를 사용하여 실행 중 성능 데이터를 `perf.data`파일로 기록한 후, `perf report`로 분석하면 됩니다. 마지막 명령어는 대화형 인터페이스와 컬러 출력을 제공하기 때문에 직접 실행해보는 것을 강력히 권장합니다. 직접 실행이 어려운 분들을 위해 아래에 최대한 설명해보겠습니다.

`perf report`를 실행하면 우선 `top` 명령과 유사한 인터페이스로 각 함수별 사용된 비중을 보여줍니다.

```
Overhead  Command  Shared Object        Symbol
  63.08%  run      run                  [.] query
  24.98%  run      run                  [.] std::__introsort_loop<...>
   5.02%  run      libc-2.33.so         [.] __random
   3.43%  run      run                  [.] setup
   1.95%  run      libc-2.33.so         [.] __random_r
   0.80%  run      libc-2.33.so         [.] rand
```

여기서 보여지는 것은 각 함수의 총 실행 시간이 아니라 오버헤드 비중입니다. 예를 들어, `setup`은 `std::__introsort_loop`를 호출하더라도 자신의 오버헤드인 3.43%만을 표시합니다. 더 직관적으로 분석하려면 [flame graphs](https://www.brendangregg.com/flamegraphs.html)를 생성할 수 있는 도구들도 있으며, 인라인 함수로 인한 영향도 고려해야 합니다.(`std::lower_bound`는 인라이닝된 것으로 보입니다.) `perf`는 `libc`같은 공유 라이브러리나 생성된 외부 프로세스까지도 추적하므로, 원한다면 웹 브라우저를 실행한 상태로 프로파일링해 내부 동작을 살펴볼 수도 있습니다.

각 함수에 대해 더 자세히 들어가면 어셈블리 코드와 함께 히트맵(heatmap)을 제공해줍니다. 예를 들어, `query` 함수의 어셈블리 분석은 다음과 같습니다.

```asm
       │20: → call   rand@plt
       │      mov    %r12,%rsi
       │      mov    %eax,%edi
       │      mov    $0xf4240,%eax
       │      nop    
       │30:   test   %rax,%rax
  4.57 │    ↓ jle    52
       │35:   mov    %rax,%rdx
  0.52 │      sar    %rdx
  0.33 │      lea    (%rsi,%rdx,4),%rcx
  4.30 │      cmp    (%rcx),%edi
 65.39 │    ↓ jle    b0
  0.07 │      sub    %rdx,%rax
  9.32 │      lea    0x4(%rcx),%rsi
  0.06 │      dec    %rax
  1.37 │      test   %rax,%rax
  1.11 │    ↑ jg     35
       │52:   sub    %r12,%rsi
  2.22 │      sar    $0x2,%rsi
  0.33 │      add    %esi,%ebp
  0.20 │      dec    %ebx
       │    ↑ jne    20
```

왼쪽 열은 각 명령어 줄에서 명령어 포인터가 멈춘 비율을 나타냅니다. 예를 들어, 전체 실행 시간 중 약 65%가 `jle b0` 명령어에서 소비되었는데, 이는 그 앞의 비교 명령어(`cmp`)가 제어 흐름을 지연시키고 있다는 것을 의미합니다.

단, [파이프라이닝](/hpc/pipelining)과 out-of-order 실행 등의 구조로 인해 현대 CPU에서는 "지금"이라는 개념이 명확하지 않기 때문에, 이 데이터는 약간의 오차가 있을 수 있습니다. 그럼에도 불구하고 명령어 수준에서의 분석은 여전히 매우 유용하며, 개별 사이클 수준의 정밀 분석이 필요하다면[더 정교한 시뮬레이션 기반 도구](../simulation)를 사용할 수도 있습니다.

<!-- flame graphs -->
