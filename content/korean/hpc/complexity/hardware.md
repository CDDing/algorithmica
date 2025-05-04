---
title: Modern Hardware
weight: 1
ignoreIndexing: true
---

1960년대 슈퍼컴퓨터의 주요 단점은 느린 속도가 아니라, 크기가 거대하고 사용이 복잡하며, 세계 강대국 정부만이 감당할 수 있을 정도로 비쌌다는 점이었습니다. 이러한 거대한 크기 때문에 비쌀 수 밖에 없었습니다. 대량 생산이 불가능한 방식으로, 전기전자공학 고급 학위를 가진 전문가들이 정밀하게 조립해야 하는 맞춤형 부품이 필요했기 때문입니다.

전환점은 하나의 작고 완전한 회로인 **마이크로칩**의 개발이었습니다.이 발명은 산업에 혁명을 일으켰으며, 20세기의 가장 중요한 발명품으로 평가받습니다. 1965년에는 수백만 달러에 달하던 컴퓨팅 장비가, 1975년에는 25달러에 구매 가능한 [4mm × 4mm 실리콘 조각](https://en.wikipedia.org/wiki/MOS_Technology_6502)[^size]안에 들어갈 수 있게 되었습니다. 이처럼 극적으로 향상된 가격 접근성은 Apple II, Atari 2600, Commondore 64, IBM PC 등의 컴퓨터가 대중에게 보급되면서, 가정용 컴퓨터 혁명을 촉발시켰습니다.


[^size]: CPU의 크기가 실제로는 센티미터 단위인 이유는, 전력 관리와 발열 해소가 필요하고, 메인보드에 쉽게 연결하기 위해서입니다.

### 마이크로칩 제조 과정

마이크로칩은 [포토리소그래피(photolithography)](https://en.wikipedia.org/wiki/Photolithography)라는 과정을 통해 결정 구조의 실리콘 조각 위에 프린트됩니다. 이 과정은 다음과 같은 단계를 포함합니다.


1. [매우 순수한 실리콘 결정](https://en.wikipedia.org/wiki/Wafer_(electronics))을 성장시킨 뒤 얇게 절단한다.
2. 그 위에 [광자에 노출되면 녹는 물질의 층](https://en.wikipedia.org/wiki/Photoresist)을 입힌다.
3. 정해진 패턴의 광좌를 쬐어 특정 부분만 노출시킨다.
4. 노출된 부분을 화학적으로 [식각(etching)](https://en.wikipedia.org/wiki/Etching_(microfabrication))한다.
5. 남은 부분을 제거한다.

그리고 이후 몇 달에 걸쳐 40~50개의 추가 공정을 거쳐 나머지 CPU를 완성합니다.

![](../img/lithography.png)

이제 "광자를 쬐는" 부분을 좀 더 살펴보겠습니다. 이를 위해 패턴을 훨씬 더 작은 영역에 투사하는 렌즈 시스템을 활용합니다. 이렇게 하면 원하는 특성을 모두 갖춘 아주 작은 회로를 효과적으로 만들 수 있습니다. 1970년대의 광학 기술은 이 방식으로 손톱 크기 안에 수천 개의 트랜지스터를 집어넣을 수 있었으며, 이로 인해 마이크로칩은 거시 세계 컴퓨터가 갖지 못했던 다음과 같은 몇 가지 핵심 장점을 얻었습니다.

- 빛의 속도에 의해 제한되던 더 높은 클럭 주파수
- 생산 공정의 확장성
- 훨씬 적은 재료와 전력 소비로 인한 단위당 비용 절감

이러한 직접적인 이점 외에도, 포토리소그래피는 성능 향상을 위한 명확한 발전 방향을 제공했습니다. 즉, 렌즈 성능을 향상시키는 것만으로 비교적 적은 노력으로 더 작지만 기능적으로 동일한 장치를 만들 수 있게 된 것입니다.

### Dennard Scaling

마이크로칩이 축소되면 어떤 일이 일어날까요? 회로가 작아질수록 그에 비례해 적은 재료가 필요하고, 트랜지스터도 작아지면 스위칭 시간(및 칩 내 다른 모든 물리적 과정)도 짧아집니다. 이는 전압을 낮추고 클럭 속도를 높일 수 있게 해줍니다.

이와 관련된 좀 더 구체적인 관찰로 Dennard Scaling이라는 개념이 있습니다. 이는 트랜지스터 크기를 30% 줄이면

- 트랜지스터 밀도가 두 배로 증가하고($0.7^2 \approx 0.5$),
- 클럭 속도가 40% 증가하며 ($\frac{1}{0.7} \approx 1.4$),
- 전력 밀도는 동일하게 유지된다는 것입니다.

단위당 제조 비용은 면적에 따라 결정되며, 시스템 운용 비용 대부분[^power]은 전력 소비에서 발생합니다. 따라서 각 세대의 칩은 대체로 비슷한 총 비용으로 더 높은 클럭 속도(40% 향상)와 두 배의 트랜지스터 수를 제공하며, 이는 예컨대 새로운 명령어를 추가하거나 워드 크기를 늘리는데 사용할 수 있어, 메모리 칩에서도 같은 수준의 소형화를 따라갈 수 있게 합니다.

[^power]: The cost of electricity for running a busy server for 2-3 years roughly equals the cost of making the chip itself.
바쁜 서버를 2-3년간 운용하는 것과 칩 하나를 생산하는 전기적 비용은 거의 동일합니다.

설계시 성능과 전력 간의 트레이드오프가 존재하기 때문에, 제조 공정의 세부 지표(예: "180nm", "65nm")는 트랜지스터 밀도를 직접 반영하게 되었고, 이는 CPU 효율성을 나타내는 상징이 되었습니다.[^fidelity].

[^fidelity]: At some point, when Moore's law started to slow down, chip makers stopped delineating their chips by the size of their components — and it is now more like a marketing term. [A special committee](https://en.wikipedia.org/wiki/International_Technology_Roadmap_for_Semiconductors) has a meeting every two years where they take the previous node name, divide it by the square root of two, round to the nearest integer, declare the result to be the new node name, and then drink lots of wine. The "nm" doesn't mean nanometer anymore.

컴퓨터 역사 대부분, 성능 향상의 주요 동력은 광학적 축소였습니다. 인텔의 CEO였던 고든 무어는 1975년의 "마이크로프로세서의 트랜지스터 수가 2년마다 두 배가 될 것"이라고 예측했는데, 이 예측은 실제로 오랜 기간 동안 유효했고, 이는 **무어의 법칙(Moore's Law)**으로 불리게 되었습니다.

![](../img/dennard.ppm)

Dennard Scaling과 무어의 법칙은 물리학의 법칙이 아니라, 뛰어난 엔지니어들이 발견한 경험적 관찰입니다. 하지만 이 두 법칙 모두 물리적 한계로 인해 언젠가는 멈출 수 밖에 없습니다. 가장 근본적인 한계는 실리콘 원자의 크기입니다. 실제로, Dennard Scaling은 이미 전력 문제로 유효하지 않게 되었습니다.

열역학적으로 보면, 컴퓨터는 전력을 열로 변환하는 매우 효율적인 장치일 뿐입니다. 이 열은 결국 외부로 방출되어야 하며, 밀리미터 크기의 실리콘 칩에서 방출할 수 있는 전력에는 물리적인 한계가 있습니다. 성능을 극대화하려는 컴퓨터 엔지니어들은 전력 소비를 일정하게 유지하면서 가능한 한 높은 클럭 속도를 선택하게 됩니다. 트랜지스터가 작아지면 정전용량도 작아지고, 그만큼 더 낮은 전압으로 동작시킬 수 있으며, 이로 인해 더 높은 클럭 속도가 가능해집니다.

그러나 2005~2007년 경, 이 전략은 누설 전류 현상으로 인해 더 이상 유효하지 않게 되었습니다. 회로의 크기가 너무 작아지면서 자기장이 인접 회로의 전자를 의도치 않게 움직이게 했고, 이로 인해 불필요한 열 발생과 간헐적인 비트 플립 현상이 생겨났습니다.

이를 완화하기 위해 전압을 높일 수 밖에 없었고, 전력 소비를 다시 억제하기 위해서는 클럭 속도를 줄여야 했습니다. 이런 방식은 트랜지스터 밀도가 증가할수록 점점 더 수익성이 낮아지게 만들어, 결국 클럭 속도는 더 이상 소형화로 증가시킬 수 없게 되었고, 소형화 추세 역시 점차 둔화되기 시작했습니다.

<!--

### Power Efficiency

It may come as a surprise, but the primary metric for modern CPUs is not the clock frequency, but rather "useful operations per joule," or, more practically put, "useful operations per dollar."

Thermodynamically, a computer is just a very efficient device for converting electrical power into heat. This heat eventually needs to be removed, and it's not straightforward to do when you are working with a millimeter-scale crystal. There are physical limits to how much power you can consume and then dissipate.

Historically, the three main variables guiding microchip designs are power, performance, and area (PPA), commonly defined in watts, hertz, and nanometers. Until ~2005, cost, which was mainly a function of area, and performance, used to be the most important criteria. But as battery-driven mobile devices started replacing PCs, power quickly and firmly moved up on top of the list, followed by cost and performance.

Leakage: interfering magnetic fields make electrons move in the directions they are not supposed to and cause unnecessary heating. It isn't bad by itself: to mitigate it you need to increase the voltage, and it won't flick any bits. But the problem is that the smaller a circuit is, the harder it is to cope with this by isolating the wires. So modern chips keep the clock frequency at a level that won't cause overheat, although physically there aren't other reasons why they shouldn't.

-->

### 현대 컴퓨팅

Dennard Scaling은 끝났지만 무어의 법칙은 아직 유효합니다.

클럭 주파수는 정체되었지만, 트랜지스터 수는 계속 증가하고 있으며, 이는 새로운 병렬 하드웨어의 등장을 가능하게 합니다. 더 빠른 사이클을 추구하기보다는, CPU 설계는 한 사이클 내에서 더 많은 유용한 작업을 수행하는 데 집중하기 시작했습니다. 트랜지스터는 더 작아지는 대신, 형태를 바꾸기 시작했습니다.

그 결과, 매 사이클마다 수십, 수백, 심지어 수천 가지 작업을 수행할 수 있는 점점 더 복잡한 아키텍처가 등장했습니다.

![Die shot of a Zen CPU core by AMD (~1,400,000,000 transistors)](../img/die-shot.jpg)

다음은 최근 컴퓨터 설계를 주도하고 있는, 늘어난 트랜지스터 수를 활용한 핵심 접근 방식입니다.

- 명령어 실행을 겹쳐 CPU의 여러 부분이 동시에 동작(파이프라이닝)
- 이전 명령의 완료를 기다리지 않고 작업 실행(추측 실행 및 비순차 실행).
- 독립적인 작업을 동시에 처리하기 위한 여러 실행 유닛을 추가(슈퍼 스칼라 프로세서)
- 기계어 워드 크기를 늘려, 128/256/512비트 블록에서 동일한 연산을 수행할 수 있는 명령 추가([SIMD](/hpc/simd/));
-   [RAM및 외부 메모리](/hpc/external-memory/)접근 속도를 높이기 위한 [여러 계층의 캐시](/hpc/cpu-cache/)
- 병렬 컴퓨팅을 위한 동일한 코어를 칩에 여러개 추가(GPU 등);
- 하나의 메인보드에 여러 칩을 사용하거나, 저렴한 여러 대의 컴퓨터를 함께 사용하는 분산 컴퓨팅을 활용
- 특정 문제를 더 효율적으로 해결하기 위한 맞춤형 하드웨어(ASIC, FPGA).

현대 컴퓨터에서는 알고리즘 성능을 예측하기 위해 [모든 연산을 세는 방식](../)은 조금 틀린 수준이 아니라, 수십 배 이상의 오차를 발생시킵니다. 이로 인해 새로운 계산 모델과 알고리즘 성능 분석 방법이 요구되고 있습니다.

<!--

Pointer jumping and processing in most scripting languages: $10^7$
Branchy operations in native languages: $10^8$
Branchless scalar processing in native languages: $10^9$
Bandwidth-bound or complex SIMD applications: $10^{10}$
Linear algebra, single core: $10^{11}$
Typical desktop CPU: $10^{12}$
Typical mobile phone GPU: $10^{12}$
Typical integrated graphics card: $2 \cdot 10^{12}$
High-end gaming setups: $10^{13}$
Deep learning hardware: $10^{14}$
Deep learning full rigs: $10^{15}$
Being considered a supercomputer: $10^{16}$
Setups used to train LM neural networks: $5 \cdot 10^{17}$
Fugaku (#1): $2 \cdot 10^{18}$
Folding@home: $3 \cdot 10^{18}$

-->
