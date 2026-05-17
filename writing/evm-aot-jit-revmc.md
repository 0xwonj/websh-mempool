---
title: "EVM AOT/JIT 이란? (feat. revmc)"
status: draft
priority: low
modified: "2026-05-15"
tags: [evm, ethereum, compiler, revmc]
---

# EVM AOT/JIT 이란? (feat. `revmc`)

## 1. Reth 2.0 이후, EVM execution이 다시 중요해졌다.

2026년 4월 8일, Reth 2.0이 릴리스되었습니다. Reth는 이미 Ethereum execution client 중에서도 성능에 매우 공격적인 구현으로 알려져 있었는데, 2.0에서는 state root 계산, storage layout, persistence 쪽 병목을 크게 줄였다고 설명합니다.

이 변화가 중요한 이유는 단순히 "Reth가 더 빨라졌다"에 있지 않습니다. 더 흥미로운 지점은 **병목의 위치가 바뀌고 있다는 점**입니다.

Ethereum client가 block을 처리할 때는 transaction을 실행하는 것만으로 끝나지 않습니다. 실행 결과를 state에 반영하고, 그 state의 root를 계산하고, 필요한 데이터를 disk에 저장해야 합니다. 과거에는 이 state root / storage / persistence 쪽 비용이 전체 성능을 강하게 제한했습니다.

그런데 Reth 2.0에서 이 부분이 크게 개선되면서, 이제 다음 최적화 대상으로 EVM execution 자체가 다시 중요해지고 있습니다. Paradigm도 Reth 2.0 발표에서 이 흐름을 직접 언급했습니다. state root 계산이 더 이상 주요 병목이 아니게 되었기 때문에, 이제 `revmc`를 통한 AOT/JIT execution 작업을 다시 본격화하고 있다는 것입니다.

여기서 `revmc`는 EVM bytecode를 더 빠르게 실행하기 위한 compiler stack입니다. 핵심 아이디어는 EVM bytecode를 매번 interpreter로만 실행하지 않고, AOT 또는 JIT compilation을 통해 native machine code로 실행할 수 있게 만드는 것입니다.

그렇다면 AOT와 JIT은 무엇이고, 왜 이것이 EVM에서 의미가 있을까요?

## 2. Interpreter, AOT, JIT의 개념

프로그램을 실행하는 방식은 여러 가지가 있습니다. 여기서는 interpreter, AOT, JIT 세 가지를 간단히 보면 됩니다.

Interpreter는 실행 중에 code를 하나씩 읽고 처리하는 방식입니다. 어떤 bytecode나 opcode가 있으면, interpreter가 그것을 읽고 "이 instruction은 어떤 동작을 해야 하는가?"를 판단한 뒤 실행합니다. 구조가 유연하고 구현하기 쉽지만, 실행 중 계속 해석과 dispatch 비용을 지불해야 합니다.

AOT는 Ahead-Of-Time compilation의 약자입니다. 말 그대로 실행 전에 미리 compile하는 방식입니다. source code나 bytecode를 실행 전에 native machine code로 바꿔 둡니다. compile time을 먼저 지불하는 대신, 실제 실행 시점에는 CPU가 바로 실행할 수 있는 code를 사용하므로 빠르게 실행할 수 있습니다.

JIT은 Just-In-Time compilation의 약자입니다. 실행 전에 모든 것을 미리 compile하지 않고, 프로그램이 실행되는 중에 자주 실행되는 부분을 찾아 compile합니다. runtime이 "이 code path는 계속 반복해서 실행되는구나"라고 판단하면, 그 부분을 native code로 바꾸고 이후 실행에서 사용합니다.

그래서 JIT에는 warmup cost가 있습니다. 처음부터 빠른 것이 아니라, 처음에는 관찰하고 compile하는 시간이 필요합니다. 하지만 같은 code가 충분히 많이 반복 실행된다면, 이후 실행에서 이 비용을 회수할 수 있습니다.

간단히 말하면 다음과 같습니다.

- Interpreter는 실행하면서 계속 해석합니다.
- AOT는 실행 전에 미리 compile합니다.
- JIT은 실행 중에 hot code를 찾아 compile합니다.

셋의 차이는 결국 **compile을 언제 하느냐**와 **실행 중 interpreter overhead를 얼마나 줄일 수 있느냐**에 있습니다.

## 3. C와 Python으로 보는 직관

C는 AOT의 직관을 이해하기 좋은 예시입니다. 일반적으로 C source code는 실행 전에 compiler를 통해 native machine code로 compile됩니다. 이 과정에서 compile time이 들지만, 그 결과물은 CPU가 직접 실행할 수 있는 binary입니다. 그래서 실행 시점에는 매우 빠릅니다.

반대로 Python, 특히 일반적인 CPython 실행 모델은 interpreter의 직관을 주는 예시입니다. Python source code는 bytecode로 바뀌고, interpreter가 이 bytecode를 실행합니다. 이 방식은 개발 경험이 좋고 유연하지만, 실행 중에 bytecode를 해석하고 dispatch하는 비용이 존재합니다.

여기서 JIT의 직관을 보여주는 좋은 예시가 PyPy입니다. PyPy는 Python 구현체 중 하나인데, 반복적으로 실행되는 hot path를 찾아 native code로 compile할 수 있습니다.

예를 들어 어떤 Python loop가 한두 번 실행되고 끝난다면, 굳이 compile하는 비용을 들일 이유가 크지 않습니다. 하지만 같은 loop가 수천 번, 수만 번 반복된다면 이야기가 달라집니다. 처음에는 interpreter로 실행하면서 관찰하고, 충분히 hot하다고 판단되면 그 부분을 native code로 compile합니다. 이후부터는 같은 logic을 매번 bytecode interpreter로 해석하지 않아도 됩니다.

이 감각이 중요합니다. compile은 공짜가 아닙니다. 하지만 같은 code가 반복될수록 compile cost를 회수할 기회가 커집니다. AOT는 이 비용을 실행 전에 지불하고, JIT은 실행 중 hot code에 대해서만 선택적으로 지불합니다.

이제 이 관점을 EVM에 적용해볼 수 있습니다.

## 4. EVM 실행의 경우

Ethereum smart contract는 EVM bytecode로 배포됩니다. 그리고 execution client는 이 bytecode를 opcode 단위로 읽고 실행합니다. 즉, EVM 실행은 기본적으로 interpreter 구조입니다.

Interpreter 구조에서는 opcode를 읽고 dispatch하는 과정, stack check, gas accounting, memory expansion check 같은 작업이 반복됩니다. `SLOAD`, `SSTORE`, `CALL`처럼 state나 다른 account와 상호작용하는 opcode라면 host/state 쪽 로직도 호출해야 합니다.

이 구조는 Ethereum의 semantics를 정확하게 구현하기 위해 필요합니다. 하지만 동시에 opcode마다 반복되는 overhead가 있습니다. 앞에서 본 AOT/JIT 관점으로 보면, 같은 bytecode가 계속 반복 실행될수록 이런 overhead를 compiled native code로 줄이는 접근이 의미를 가질 수 있습니다.

예를 들어 아주 단순한 bytecode를 생각해볼 수 있습니다.

```evm
PUSH1 2
PUSH1 3
ADD
PUSH1 4
MUL
```

이 bytecode는 개념적으로 `(2 + 3) * 4`를 계산합니다. Interpreter는 이 다섯 개 opcode를 하나씩 읽고 실행합니다. 매번 opcode를 fetch하고, 어떤 동작인지 dispatch하고, stack에서 값을 꺼내고 다시 넣고, gas와 program counter를 처리합니다.

반대로 compiled native code에서는 이 opcode 묶음을 하나의 계산 흐름으로 다룰 수 있습니다. 이미 "2와 3을 더하고, 그 결과에 4를 곱한다"는 구조를 알고 있기 때문에, 매 opcode마다 다시 해석하고 dispatch할 필요가 없습니다. 경우에 따라 중간 값을 EVM stack에 계속 넣었다 빼는 대신 native register나 local value로 유지할 수도 있습니다.

핵심은 `ADD`나 `MUL` 하나가 마법처럼 빨라지는 것이 아닙니다. **opcode마다 반복되던 fetch, dispatch, stack bookkeeping 같은 비용이 줄어든다**는 점입니다. 같은 bytecode가 많이 반복될수록 이 차이가 누적됩니다.

여기서 Ethereum은 꽤 흥미로운 환경입니다. Ethereum에서는 **반복 실행되는 bytecode가 매우 큰 비중을 차지합니다.** WETH, USDC, 주요 ERC-20, Uniswap, vault, bridge, rollup system contract, oracle contract 같은 code는 계속 호출됩니다. 그리고 이런 hot contract의 상당수는 사전에 알 수 있습니다.

배포된 contract bytecode는 일반적인 application code처럼 계속 바뀌지 않습니다. runtime bytecode는 안정적이고, code hash로 식별할 수 있습니다. 따라서 자주 실행되는 contract code를 미리 compile해두고, code hash를 기준으로 compiled artifact를 cache하는 접근이 자연스럽습니다.

개인적으로는 이 지점에서 AOT 접근이 특히 흥미롭다고 봅니다. 이미 hot하다는 것을 알고 있는 contract라면 runtime에서 매번 새로 판단할 필요가 없습니다. 미리 compile해두고, 실행 시점에는 준비된 native code path를 사용하는 것이 가능합니다. 반대로 새로 배포된 contract나 특정 workload에서만 hot해지는 bytecode는 JIT 방식으로 runtime에서 관찰하고 background compile할 수 있습니다.

즉, EVM에서는 AOT와 JIT을 각각 미리 준비하는 경로와 실행 중 준비하는 경로로 이해하면 됩니다. 핵심은 **반복 실행되는 EVM bytecode를 native execution path에 올리는 것**입니다.

## 5. revmc의 동작 방식

`revmc`는 EVM bytecode를 native machine code로 compile하기 위한 experimental compiler stack입니다. 목표는 Ethereum의 semantics를 바꾸는 것이 아닙니다. 같은 EVM bytecode가 같은 결과를 내도록 유지하면서, 실행 경로를 interpreter에서 compiled native code 쪽으로 옮기는 것입니다.

대략적인 흐름은 다음과 같습니다.

먼저 `revmc`는 EVM bytecode를 분석합니다. bytecode 안의 control flow, jump target, stack 상태, gas, memory 사용 등을 파악해야 합니다. EVM은 stack machine이고, dynamic jump도 존재하기 때문에 단순히 opcode를 1:1로 native instruction으로 바꾸는 것만으로는 충분하지 않습니다. 실행 흐름과 stack 상태를 정확히 추적해야 합니다.

그 다음 bytecode를 intermediate representation, 즉 IR로 변환합니다. IR은 compiler가 최적화를 적용하기 좋은 중간 표현입니다. 이 단계에서 불필요한 stack operation을 줄이거나, control flow를 정리하거나, 같은 block을 deduplicate하는 식의 최적화가 가능해집니다.

이후 최적화된 IR은 LLVM으로 전달되고, LLVM이 실제 native machine code를 생성합니다. 이렇게 만들어진 code는 실행 시점에 interpreter 대신 사용할 수 있습니다.

운영 방식은 AOT와 JIT 두 가지가 모두 가능합니다. 자주 쓰이는 contract는 미리 compile해서 artifact로 저장해둘 수 있습니다. 이때 compiled artifact는 단순히 bytecode만이 아니라 hardfork/spec, compiler version, compile option 같은 조건과도 함께 관리되어야 합니다. 같은 bytecode라도 어떤 EVM rule에서 실행되는지에 따라 의미가 달라질 수 있기 때문입니다.

반대로 runtime에서 특정 bytecode가 자주 실행된다는 것을 관찰한 뒤, background에서 compile하고 준비가 되면 compiled path로 실행할 수도 있습니다. 이 경우 처음에는 interpreter로 실행하다가, compiled code가 준비된 뒤부터 native code path를 사용할 수 있습니다.

그리고 `revmc`에 의해 컴파일되지 않은 bytecode는 기존처럼 interpreter로 실행됩니다. 아직 compile 대상이 아니거나, compiled artifact가 없거나, compile에 실패한 경우에도 Ethereum client는 올바른 결과를 내야 하기 때문입니다.

## 6. 효과와 한계

`revmc` 같은 접근은 compute-heavy bytecode에서 가장 큰 효과를 기대할 수 있습니다. 반복 계산, arithmetic, loop처럼 interpreter가 많은 opcode를 계속 처리해야 하는 경우에는 compiled native code의 장점이 잘 드러납니다.

Paradigm은 `revmc` 발표에서 selected benchmark 기준 1.85x에서 19x까지의 개선을 언급했습니다. 특히 Fibonacci 같은 compute-heavy benchmark에서는 큰 차이가 날 수 있습니다. 이런 예시는 "같은 logic을 opcode interpreter로 계속 실행하는 것"과 "native code로 실행하는 것"의 차이를 잘 보여줍니다.

또 하나 의미 있는 영역은 simulation-heavy workload입니다. fuzzing, invariant test, fork test, transaction simulation, liquidation bot, risk engine처럼 같은 contract code를 반복해서 실행하는 환경에서는 compiled execution의 효과를 더 크게 기대할 수 있습니다. 이 경우에는 block 하나를 처리하는 것보다 같은 code path를 많이 반복하는 일이 더 중요해질 수 있습니다.

하지만 모든 EVM workload에서 큰 속도 향상이 나는 것은 아닙니다. EVM 실행은 opcode 계산만으로 이루어지지 않습니다. `SLOAD`, `SSTORE`, `CALL`, account access, DB I/O, state commitment 같은 비용은 native compilation만으로 사라지지 않습니다.

예를 들어 ERC-20 transfer나 AMM swap은 매우 자주 실행되는 workload이지만, storage read/write, log, state update 같은 비용이 큽니다. 이런 경우 interpreter overhead를 줄여도 전체 wall-clock time에서 차지하는 비중이 제한적일 수 있습니다. 결국 중요한 질문은 "JIT이 빠른가?"가 아니라, **해당 workload에서 opcode interpretation이 실제 병목인가?**입니다.

현재 `revmc`는 아직 experimental 성격이 강합니다. Reth integration도 진행 중이고, 성능은 workload-dependent합니다. 하지만 Reth 2.0 이후 state root / persistence 쪽 병목이 줄어든 상황에서는, EVM execution 자체를 어떻게 더 빠르게 만들 것인지가 더 중요한 질문이 되었습니다.

다만 execution client에서 이런 최적화가 실제로 쓰이려면, 빠른 것만으로는 부족합니다. **Correctness는 절대 양보할 수 없습니다.** compiled code가 기존 interpreter와 정확히 같은 결과를 내야 하고, gas, revert, exceptional halt, memory expansion, fork별 opcode semantics까지 모두 일치해야 합니다. 이를 위해 differential fuzzing이나 formal verification과 같은 검증이 필요할 것입니다.

제 생각에는 `revmc`의 가장 큰 의미가 여기에 있습니다. 당장 모든 EVM workload를 크게 빠르게 만드는 완성된 기능이라기보다, 반복적으로 실행되는 EVM bytecode를 안전하게 compiled execution path에 올리기 위한 중요한 기반으로 보는 것이 좋습니다.
