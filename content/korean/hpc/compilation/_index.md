---
title: Compilation
aliases: [/hpc/analyzing-performance/compilation]
weight: 4
---

[어셈블리어](../architecture/assembly)를 배우는 주된 이점은 그것으로 프로그램을 작성하는 능력을 기르는 것이 아니라, 컴파일된 코드가 실행되는 동안 내부에서 어떤 일이 일어나는지와 그에 따른 성능상의 영향을 이해하는 데 있습니다.

최대 성능을 위해 손으로 어셈블리어를 작성해야할 일은 드물며, 대부분의 경우 컴파일러는 자체적으로 거의 최적에 가까운 코드를 생성할 수 있습니다. 만약 그렇게 하지 못한다면, 대개 그 이유는 프로그래머가 문제에 대해 소스 코드만으로는 알 수 없는 추가 정보를 알고 있으면서도, 그 정보를 컴파일러에게 제대로 전달하지 못했기 때문입니다.

이번 장에서는 컴파일러가 우리가 의도한 대로 동작하게 만드는 방법과, 추가적인 최적화를 이끌어낼 수 있는 유용한 정보를 수집하는 과정의 복잡함을 다룰 것입니다.
