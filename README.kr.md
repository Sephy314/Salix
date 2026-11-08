# Salix Consensus — Owner-Scoped Core

## 1. 개요와 연구 목표

Salix는 동기 통신과 디지털 서명을 이용하여, 과반 미만의 비잔틴 노드와 비잔틴 클라이언트가 존재하는 환경에서 클라이언트가 서명한 액션에 대한 모든 정직 노드의 합의 및 상태 수렴을 제공하는 Leaderless Byzantine Consensus 프로토콜이다.

시스템은 다음 조건을 목표로 한다.

- 노드 수: $n=2f+1$
- 허용 비잔틴 노드 수: 최대 $f$
- 커밋 쿼럼: $f+1$
- 통신 모델: 동기 통신
- 인증: 디지털 서명
- 합의 구조: Leaderless
- 합의 라운드: $f$에 의존하지 않는 상수 라운드
- 변수 모델: 모든 변수가 단일 Owner에게 귀속됨

### 1.1. 연구 목표

Salix의 주된 목표는 $n=2f+1$ 환경에서 동기 통신과 디지털 서명을 이용하여 Leaderless 및 상수 라운드 합의를 구성하는 것이다.

이 설계가 해결하고자 하는 문제는 다음과 같다.

1. **Leader availability:** 단일 리더에 대한 의존 및 리더 장애 이후 View Change에 의존하는 가용성 구조를 제거한다.
2. **Round-dependent communication overhead:** $f+1$ 라운드 BackOff 계열 구조가 초래하는 라운드 및 통신 비용을 줄인다.
3. **Centralized ordering dependency:** 하나의 중앙화된 리더가 전역 순서를 결정해야 하는 구조를 피한다.

Salix의 추가적인 기여는 합의에서 다루는 행위를 변수의 정의 및 수정으로 추상화하는 데 있다. 합의 대상은 임의의 트랜잭션 묶음이 아니라, Owner가 명시된 변수에 대한 버전 지정 상태 전이(action)다.

### 1.2. 합의 대상의 추상화

합의가 다루는 모든 액션은 하나의 변수에 대한 연산으로 환원한다.

- **OP 연산:** 변수의 정의를 다룬다.
  - `Declare`: 변수 선언
  - `Redefine`: 변수 메타데이터 재정의
  - `Retire`: 변수 정의 제거
- **WR 연산:** 변수의 값 또는 읽기를 다룬다.
  - `Write`: 변수 값 수정
  - `Read`: 다른 변수의 특정 실행 버전 참조

핵심 규칙은 다음과 같다.

1. 한 액션은 정확히 하나의 변수만 수정한다.
2. 모든 변수는 하나의 Owner에게 귀속된다.
3. 한 변수의 버전 순서는 해당 변수의 독립적인 버전 열로 표현한다.
4. 서로 다른 변수 사이에는 기본 전역 순서를 두지 않는다.
5. 변수 간 의존성이 필요한 경우 Read dependency로 명시한다.
6. 여러 변수로부터 계산되는 공유 상태는 직접 합의하는 변수가 아니라 결정적 함수로부터 도출되는 파생 상태로 취급한다.

### 1.3. Owner-scoped state와 파생 공유 상태

Salix는 공유 상태가 존재한다는 사실과 공유 변수가 존재해야 한다는 주장을 구분한다.

여러 주체가 함께 관찰하거나 사용하는 논리적 상태 $S$는 각 주체가 소유한 독립 변수들의 결정적 함수로 표현할 수 있다.

$$
S=F(X_1,X_2,\ldots,X_m)
$$

여기서 $X_i$는 Owner $i$에게 귀속된 변수 또는 변수 집합이며, $F$는 구현체가 정의하는 결정적 함수다.

예를 들어 공동 계좌의 잔액은 Alice와 Bob이 각각 소유한 거래 내역의 합으로 도출할 수 있다.

$$
Balance=B_0+\sum T_A+\sum T_B
$$

이 모델에서 `Balance`는 반드시 별도의 공유 변수일 필요가 없다. 합의 대상은 각 Owner의 거래 내역에 대한 수정 액션이며, 잔액은 해당 상태들의 함수로 계산된다.

이 접근은 공유 상태의 직접적인 다중 작성 권한을 요구하지 않고도 공유 상태를 표현할 가능성을 제공한다. 다만 파생 상태의 계산 가능성이 원자적 다중 변수 갱신, 전역 불변식 또는 동시성 제약까지 자동으로 보장한다는 뜻은 아니다. 해당 제약은 애플리케이션 설계에서 별도로 다뤄야 한다.

### 1.4. Core와 Blocks

- **Core**는 어떤 피소유 변수에 어떤 액션이 몇 번째 버전으로 합의되었는지를 보장한다. 핵심 산출물은 액션과 그 order를 $f+1$개 노드 서명으로 증명하는 자기검증형 Commit Certificate(CC)다.
- **Blocks**는 Failure Recovery, Checkpoint, Admission, Application, Result Query, Reaction, Gateway 등 Core 외부의 부가 기능이다.

Core는 단일 Fast Path만 제공한다. Epoch consensus, GLOBAL Owner 및 Slow Path는 존재하지 않는다.

### 1.5. Core 원칙

- 리더는 없다. 제안자(Proposer)는 클라이언트와 노드 사이의 브릿지이며 순서나 채택을 결정하지 않는다.
- 모든 변수는 하나의 Owner에게 귀속된다.
- 변수 Owner만 해당 변수의 액션을 서명하고 제출할 수 있다.
- 액션과 Fast CC는 검증 후 한 번 전파한다.
- 모든 CC는 액션과 order를 함께 증명한다.
- 액션은 읽을 값 자체가 아니라 이미 실행된 버전의 참조를 포함한다.
- 실행은 액션과 고정된 입력값만 사용하는 결정적 함수로 수행한다.
- 공유 상태는 피소유 변수들의 결과로부터 파생할 수 있으며, 그 파생 상태 자체는 Core의 합의 대상이 아니다.

---

## 2. 시스템 모델 및 가정

### 2.1. 노드

노드 수는 $n=2f+1$이다.

- 정직 노드는 최소 $f+1$개이며 장애가 없고 프로토콜을 따른다.
- 비잔틴 노드는 최대 $f$개다.
- 비잔틴 노드는 Equivocation, 침묵, 부분 침묵, 담합 및 임의 메시지 전송을 할 수 있다.

### 2.2. 클라이언트와 Owner

클라이언트는 비잔틴일 수 있다. 비잔틴 클라이언트는 거짓 액션, 리플레이, 동시 제출, 동일한 `(V, ver)`에 대한 서로 다른 액션 서명 등을 시도할 수 있다.

- 클라이언트는 자신의 키만 사용할 수 있고 다른 클라이언트를 사칭할 수 없다.
- 변수의 Owner는 해당 변수의 수정 권한을 가진 클라이언트 식별자다.
- 정직한 Owner는 동일한 `(V, ver)`에 대해 하나의 액션에만 서명한다.
- 비잔틴 Owner는 동일한 `(V, ver)`에 상충하는 액션을 서명할 수 있다. 이 경우 해당 변수의 진행이 정지할 수 있다.

### 2.3. 암호학적 가정

1. 노드와 클라이언트는 서로의 공개키를 알고 서명을 검증할 수 있다.
2. 서명은 위조할 수 없고 해시는 충돌 저항성을 가진다.
3. 서명은 메시지 종류와 대상을 포함하여 도메인이 분리된다. 예: `"ACT"`, `"COMMIT"`.
4. 정직 노드는 자신의 서명을 변조하지 않는다.

### 2.4. 동기 모델

- $\Delta$: 정직 노드 간 메시지 전달 지연 상한
- $\epsilon$: 검증 및 스케줄링 처리 시간 상한
- $\Delta'=\Delta+\epsilon$
- $W$: Fast 투표 전 대기 시간이며 $W\geq\Delta'$

Fast Path는 로컬 상대 시간만 사용한다. Slow Path의 라운드 정렬을 위한 클럭 편차 $\delta$ 및 에폭 시작 시각은 더 이상 필요하지 않다.

### 2.5. 제안자

제안자는 클라이언트가 액션을 보낸 노드다.

1. 액션을 수신하고 모든 노드에게 `Propose`로 전달한다.
2. 해당 액션의 CC를 보유하면 클라이언트에게 전달한다.

제안자는 액션을 변조할 수 없고(클라이언트 서명), CC를 위조할 수 없다(노드 서명). 비잔틴 제안자는 침묵, 일부 전달, 지연 또는 CC 보류만 가능하며 안전성에는 영향을 주지 않고 진행성에만 영향을 준다.

클라이언트는 타임아웃 후 같은 액션을 다른 제안자에게 재시도할 수 있다. 제안자를 균등하고 독립적으로 선택한다고 가정하면 $k$번 안에 정직 제안자를 만날 확률은 다음과 같다.

$$
1-\left(\frac{b}{2f+1}\right)^k
\geq
1-\left(\frac{f}{2f+1}\right)^k
$$

여기서 $b\leq f$는 실제 비잔틴 노드 수다.

### 2.6. Failure Recovery

Core는 Failure Recovery 자체를 제공하지 않는다. 복구는 Core 위에 쌓는 블록이다. 노드가 어떤 시점까지 CC를 받지 못했다는 사실은 실패가 아니라 결과 미정 상태를 뜻한다.

---

## 3. 변수 모델과 연산

### 3.1. 변수와 상태

변수는 다음과 같이 식별한다.

$$
V=(owner,name)
$$

`owner`는 클라이언트 키 또는 `clientId`다. `GLOBAL` Owner는 존재하지 않는다.

상태는 다음과 같이 표현한다.

```text
S : V ↦ (def = meta, val, ver)       // 미정의 변수는 S에 없음
lastVer[V]                          // 마지막으로 소모된 버전, Retire 후에도 유지
history[V, ver]                     // 구현체가 보관하는 실행 이력
```

- `ver`는 해당 변수에 합의된 액션이 소모한 버전 수다.
- 버전은 변수마다 단조 증가한다.
- 삭제와 재선언을 거쳐도 버전은 이어진다.
- `(V, ver)`는 해당 변수의 역사에서 유일한 위치다.
- `lastVer[V]`는 삭제 이후에도 묘비처럼 유지한다.

모든 변수의 갱신 권한은 동일한 규칙을 따른다.

| Owner | 갱신 권한 | 순서 |
|---|---|---|
| `clientId` | 해당 Owner가 서명한 액션만 | 변수 `V`의 버전 열 |

### 3.2. 연산 정의

| 종류 | 연산 | 선행 조건 | 효과 |
|---|---|---|---|
| OP | `Declare(V, meta, init)` | `V`가 미정의 | 변수를 정의하고 초기값 설정 |
| OP | `Redefine(V, meta')` | `V`가 정의됨 | 메타데이터 교체, 값과 Owner 유지 |
| OP | `Retire(V)` | `V`가 정의됨 | 정의 제거, `lastVer[V]` 유지 |
| WR | `Write(V, f, args)` | `V`가 정의됨 | `val(V)=f(pinned reads,args)` |
| WR | `Read(V', ver')` | 해당 버전이 이미 실행됨 | 상태를 바꾸지 않고 액션 입력으로 사용 |

OP 연산의 의미, 변수의 존재, 소유권, 버전 및 선행 조건은 Core가 정의한다.

WR 쓰기의 값 함수 `f`와 업무 규칙은 구현체가 정의한다. 단, `apply`는 선언된 읽기와 대상 변수만 다룰 수 있다. 이를 위반하면 액션 전체를 no-op으로 처리한다.

소유권 이전은 Core 연산에 포함하지 않는다. 필요하다면 기존 Owner의 `Retire`와 새 Owner의 `Declare`를 이용해 구현체 수준에서 표현할 수 있다. 이 과정은 서로 다른 Owner의 버전 열 사이에 별도의 조정이 필요할 수 있음을 의미한다.

### 3.3. 액션 형식

```text
Action {
    author            // 서명자(clientId), target.owner와 같아야 함
    target            // 영향을 주는 단 하나의 변수 V
    ver               // V에서 소모할 버전
    reads             // [(V', ver')] 이미 실행된 다른 변수 버전 참조
    effect            // OP.Declare | OP.Redefine | OP.Retire | WR.Write(f, args)
    authorSignature   // Sign(authorSK, "ACT" || author || target || ver || reads || effect)
}

proposeId = H(author || target || ver || reads || effect)
```

정적 검증은 서명, 해시, 형식 및 `target.owner == author`를 검사한다.

한 액션은 하나의 `target`만 가지므로 하나의 변수만 수정한다.

### 3.4. Read dependency 계약

액션은 읽을 변수를 `(V', ver')` 참조로 지목한다. 값 자체는 액션에 포함하지 않는다.

**핵심 제한: Read는 이미 실행이 완료된 버전만 참조할 수 있다.**

$$
Executed(V',ver') \prec Accept(a)
$$

여기서 `Accept(a)`는 해당 액션이 Fast 투표 대상이 되기 위한 로컬 수락 시점을 뜻한다. 단순히 CC가 존재하거나 커밋되었다는 사실만으로는 충분하지 않다. CC는 합의된 순서를 증명하지만, 해당 버전의 상태 적용이 완료되었음을 자동으로 뜻하지 않기 때문이다.

규칙은 다음과 같다.

1. 액션의 모든 Read reference는 유효한 변수 및 버전을 지목해야 한다.
2. 참조된 버전은 해당 노드에서 이미 실행되어야 한다.
3. 실행되지 않은 참조가 포함된 액션은 전파할 수 있지만, 참조가 실행될 때까지 투표 대상이 될 수 없다.
4. 정직한 클라이언트는 참조 버전의 실행을 확인한 후 액션을 제출해야 한다.
5. 이미 실행된 버전에 대한 참조는 실행 대기 그래프에 역방향 의존성을 만들지 않는다.
6. 과거 버전을 참조하려면 해당 버전의 이력이 보관되어 있어야 한다.

이 제한은 실행되지 않은 버전 사이의 순환 대기 및 의존성 교착을 방지하기 위한 것이다.

### 3.5. 결정적 실행 계약

실행 함수는 다음 형태를 가진다.

$$
apply(action,pinned)\rightarrow state\ change\ |\ no\text{-}op
$$

- `pinned`는 액션이 명시적으로 참조한 이미 실행된 버전들의 값이다.
- 함수는 액션과 `pinned` 이외의 상태를 읽을 수 없다.
- 상태 의존 검증(예: 잔액 확인)은 `apply` 안에서 결정적으로 수행한다.
- 검증 실패는 no-op으로 처리한다. CAS가 성공한 액션은 실패 여부와 관계없이 버전을 소모한다.

**구현체 결정성 요구사항:** 모든 정직 노드는 동일한 액션, 동일한 pinned input, 동일한 이전 상태에서 동일한 결과를 생성해야 한다.

구현체는 이를 위해 다음을 통제해야 한다.

- 비동기적 또는 노드별 Wall-clock 의존 금지
- 조정되지 않은 난수 사용 금지
- 직렬화 및 해시 입력의 정규화
- 부동소수점 연산 차이 또는 비결정적 자료구조 순회 통제
- 외부 데이터 접근의 합의된 입력화
- 동일한 초기 상태 및 동일한 애플리케이션 규칙 버전
- 런타임 및 플랫폼 차이로 결과가 달라질 수 있는 연산의 제한 또는 정규화

Core는 업무 연산의 구체적 구현을 강제하지 않지만, 구현체는 위 결정성 계약을 만족해야 한다.

### 3.6. 파생 공유 상태

여러 Owner의 피소유 변수로부터 도출되는 공유 상태는 Core 변수로 선언하지 않아도 된다.

$$
S=F(X_1^{v_1},X_2^{v_2},\ldots,X_m^{v_m})
$$

여기서 $X_i^{v_i}$는 Owner $i$가 소유한 변수의 특정 실행 버전이다.

예: 공동 계좌

```text
Alice.transactions = [+100, -30, +50]
Bob.transactions   = [-20, +200, -10]

Balance = initialBalance
        + sum(Alice.transactions)
        + sum(Bob.transactions)
```

`Balance`는 독립적인 공유 변수 대신 피소유 거래 내역의 결정적 함수로 계산된다.

이 모델의 의미:

- 각 Owner는 자기 변수만 수정한다.
- 공유 상태는 여러 피소유 변수의 결과로 구성한다.
- 합의 대상은 개별 변수의 상태 전이다.
- 파생 상태 자체는 별도의 합의나 Owner를 요구하지 않는다.

#### 3.6.1. 파생 상태의 일관성

파생 상태를 동일하게 계산하려면 모든 정직 노드가 같은 입력 버전을 사용해야 한다.

예를 들어 다음은 하나의 명시적 입력 스냅샷이다.

$$
Balance=F(Alice^{(5)},Bob^{(8)})
$$

각 노드가 서로 다른 버전 조합을 사용하면 서로 다른 결과를 계산할 수 있다. 따라서 파생 상태의 조회 및 계산은 입력 버전 벡터를 명시해야 한다.

이 모델이 자동으로 보장하지 않는 것:

- 여러 변수의 원자적 동시 수정
- 여러 변수에 걸친 전역 불변식
- 동시 작업에 대한 전역 선형화 가능성
- 파생 상태의 자동 최신화

이러한 요구사항은 구현체의 업무 규칙, 분할 전략, 예약 프로토콜 또는 별도의 조정 메커니즘으로 해결해야 한다.

### 3.7. 변수 선언 규칙

선언(`Declare`)은 일반 액션이다.

1. 정의되지 않은 변수의 읽기와 쓰기는 결정적 no-op이다.
2. 동일한 `V`를 여러 번 선언하면 버전 순서에서 먼저 실행된 선언만 유효하다.
3. 선언 이전 버전을 지목한 Read는 유효하지 않다.
4. 선언 자격과 이름 규칙은 구현체의 결정적 정책으로 정한다.
5. 삭제 후 같은 이름으로 재선언하면 기존 `lastVer[V]` 다음 버전부터 이어간다.

---

## 4. 식별과 Commit Certificate

### 4.1. 레인과 충돌

- **레인(lane)**은 변수 `V`다.
- 서로 다른 변수 사이에는 기본 순서가 없다.
- 변수 간 선후 관계는 Read dependency로 명시한다.
- 충돌은 동일한 `(V, ver)`에 서로 다른 `proposeId`가 존재하는 경우다.
- 멱등키는 `(V, ver)`다.

버전 열 규칙:

- `(V, ver)`는 `(V, ver-1)`이 실행된 뒤에만 실행한다.
- 정직한 Owner는 이전 버전의 합의 및 실행을 확인한 뒤 다음 버전을 제출하는 것을 기본 규칙으로 한다.
- 파이프라이닝은 구현체가 별도 안전 조건을 만족하는 경우에만 허용한다. 기본 명세에서는 앞 버전의 실행 전 다음 버전의 실행을 허용하지 않는다.

### 4.2. Commit Certificate

```text
CC {
    action
    order      // (V, ver)
    proof      // { (nodeId, sig) }
               // sig = Sign("COMMIT" || proposeId || order)
}
```

Fast CC 유효 조건:

1. 서로 다른 노드의 유효한 서명 $f+1$개 이상.
2. 모든 서명은 같은 `proposeId`와 `order`를 대상으로 한다.
3. `order == (action.target, action.ver)`.
4. `action.target.owner == action.author`.
5. 액션의 서명 및 정적 형식 검증이 유효하다.

CC 하나로 액션, 순서 및 서명자를 검증할 수 있으며 누구에게서 받든 독립적으로 검증할 수 있다.

CC는 해당 액션이 해당 변수 버전 위치로 합의되었음을 증명한다. 실행 결과의 조회는 Result Query 블록이 담당한다.

---

## 5. Consensus

### 5.1. 메시지 및 전파

메시지 종류:

- `Propose`: 제안자에서 노드로 전달
- `Prepare`: 노드 간 액션 중계
- `Vote`: 노드의 Commit 서명
- `CC`: 커밋 인증서

메시지는 수신 시점의 시간 구간으로 거부하지 않는다.

```text
on receive(x):                       // Action 또는 Fast CC
    id = proposeId(x)
    if forwarded[id, kind(x)]: return
    if !verifyStatic(x): return
    atomically:
        if forwarded[id, kind(x)]: return
        forwarded[id, kind(x)] = true
    broadcast(x, everyone)
    handle(x)
```

- `forwarded`는 상태 무관 검증 통과 후에만 기록한다.
- 전파 게이트에는 현재 버전, 실행 여부 등 상태 의존 조건을 두지 않는다.
- 유효한 액션은 Read dependency가 아직 실행되지 않았더라도 전파한다.
- 투표 자격은 별도로 검사한다.

### 5.2. 투표 규칙

액션 $r$을 처음 본 로컬 시각을 $t_r$, 대기 시간을 $W\geq\Delta'$라 한다.

투표 시점은 다음 두 조건을 모두 만족해야 한다.

1. 첫 목격 이후 $W$가 경과했다.
2. 모든 Read dependency가 해당 노드에서 이미 실행되었다.

```text
on first sight of action r:
    t_r = local_time()
    schedule vote eligibility check at t_r + W

on vote eligibility check(r):
    key = (r.target, r.ver)

    if |seen[key]| > 1:
        return

    if voted[key]:
        return

    if any read dependency is not executed:
        wait until all read dependencies are executed
        re-check conflict and voted conditions
        return

    voted[key] = true
    order = key
    broadcast(
        Vote(
            r.proposeId,
            order,
            Sign("COMMIT" || r.proposeId || order)
        ),
        everyone
    )
```

`seen[key]`에는 상태 무관 검증을 통과한 해당 키의 서로 다른 액션이 기록된다. Read dependency가 실행되기를 기다리는 동안에도 충돌 검사는 다시 수행한다.

### 5.3. CC 구성 및 Commit

```text
on receive Vote:
    verify signature and order
    votes[proposeId, order] += distinct signer

    if at least f+1 distinct valid signatures
       and action is known:
        cc = construct CC
        Commit(cc)

on Commit(cc):
    held[cc.proposeId] = cc
    scheduleExecution(cc)
```

CC 수신만으로 즉시 실행하지 않는다. 변수 버전 순서 및 Read dependency 실행 조건을 만족할 때 실행한다.

### 5.4. 클라이언트 응답

Reply는 CC다.

제안자는 액션의 CC를 보유하면 클라이언트에게 전달한다. 클라이언트는 CC를 직접 검증하여 액션이 해당 order로 커밋되었음을 확인한다.

이미 CC를 보유한 액션을 다시 제안받으면 새 합의 없이 CC를 반환할 수 있다.

실행 결과의 조회는 CC 범위 밖이며 Result Query 블록이 담당한다.

---

## 6. 실행

### 6.1. 실행 조건

`(V, ver)` 실행 조건:

1. `(V, ver-1)`이 실행되었다. 해당 변수의 첫 버전이면 생략한다.
2. 모든 Read dependency `(V', ver')`가 실행되었다.
3. 참조한 과거 버전의 값이 이력에서 조회 가능하다.
4. 액션에 대한 유효한 CC가 존재한다.

조건을 만족하면 `apply(action, pinned)`를 실행한다.

### 6.2. 변수별 실행

```text
on execution candidate cc:
    V = cc.action.target
    ver = cc.action.ver

    if ver <= lastVer[V]:
        skip as duplicate or stale action

    if ver > lastVer[V] + 1:
        buffer cc within deterministic window
        return

    if any read dependency is not executed:
        buffer cc
        return

    pinned = load exact referenced versions
    apply(cc.action, pinned)
    advance lastVer[V] according to CAS rule
    process next buffered version of V
```

기본 규칙에서 정직한 Owner는 버전 구멍을 만들지 않는다. 비잔틴 Owner가 버전을 건너뛰거나 상충 액션을 제출하면 해당 변수의 진행이 멈출 수 있다.

### 6.3. apply 의미

```text
apply(action, pinned):
    require action.target == one declared variable
    require all reads are explicitly declared and pinned

    if CAS does not match:
        no-op

    consume the version when CAS matches

    check deterministic OP/WR preconditions:
        if failed:
            preserve state
            keep consumed version
        else:
            apply the single-variable effect
```

모든 정직 노드는 동일한 실행 순서와 동일한 pinned input을 사용해야 한다. 구현체 결정성 계약을 위반하는 애플리케이션은 Salix의 상태 수렴 보장 범위에 포함되지 않는다.

---

## 7. Byzantine 공격 시나리오

### 7.1. 비잔틴 클라이언트 및 Owner

| 공격 | 결과 |
|---|---|
| 다른 클라이언트 사칭 | 서명 검증에서 거부 |
| 다른 Owner 변수 수정 | `target.owner == author` 검증에서 거부 |
| 액션 리플레이 | 이미 소모된 버전으로 판정하여 무시 |
| 동일 `(V, ver)`에 서로 다른 액션 서명 | 정직 투표가 발생하지 않아 해당 변수 레인만 정지 |
| 한 액션으로 여러 변수 수정 | 단일 `target` 형식으로 불가능 |
| 버전 구멍 생성 | 해당 변수의 후속 버전 실행이 대기 또는 폐기 |
| 실행되지 않은 버전 참조 | 투표 자격이 보류됨 |
| 미정의 변수 사용 | 결정적 no-op |
| 중복 선언 | 버전 순서상 첫 유효 선언만 적용 |
| 변수 대량 생성 | Admission 및 Declaration Policy 블록으로 제한 |
| 거짓 업무 의미(잔액 초과 등) | 구현체의 결정적 `apply` 검증에서 no-op |

### 7.2. 비잔틴 노드 및 제안자

| 공격 | 결과 |
|---|---|
| 일부 정직 노드에만 액션 전달 | 정직 노드 하나가 받으면 전파를 통해 모든 정직 노드에 전달 |
| 충돌 액션을 서로 다른 시점에 주입 | $W\geq\Delta'$ 대기와 전파로 양쪽 정직 투표를 방지 |
| 일부 노드와 담합한 부분 CC | 정직 노드 하나가 받으면 CC 전파로 전체에 전달 |
| 제안자 침묵 또는 CC 보류 | 클라이언트 재시도. 안전성에는 영향 없음 |
| 위조 CC | $f+1$개 유효 서명이 없으면 검증 실패 |
| 서로 다른 order에 서명 유도 | 정직 노드의 투표 규칙과 충돌 검사가 방지 |

---

## 8. 보장 속성

- **Validity:** 유효한 CC의 액션은 Owner 서명 및 상태 무관 검증을 통과한다. 비잔틴 노드만으로 CC를 만들 수 없다.
- **Uniqueness:** 하나의 `proposeId`에는 하나의 액션만 대응한다.
- **Totality:** 정직 노드가 본 유효한 액션과 Fast CC는 $\Delta'$ 안에 모든 정직 노드가 본다.
- **Fast Order Uniqueness:** 동일한 `(V, ver)`에 서로 다른 액션이 모두 정직 투표를 받을 수 없다.
- **Order Authenticity:** 유효한 CC는 하나의 액션과 하나의 변수 버전 order를 증명하며, 동일 order에 서로 다른 액션의 CC가 존재하지 않는다.
- **Single-Variable Effect:** 한 액션은 하나의 변수 정의 또는 값에만 영향을 준다.
- **OP Determinism:** 변수 정의 테이블은 모든 정직 노드에서 같다.
- **Read Safety:** 액션은 이미 실행된 버전만 참조하며 실행 의존성 순환을 만들지 않는다.
- **Deterministic Execution:** 동일한 액션, 이전 상태 및 pinned input은 모든 정직 노드에서 동일한 결과를 만든다.
- **Convergence:** 모든 정직 노드는 각 변수에 대해 동일한 버전 열과 상태에 수렴한다.
- **Liveness:** 정직 Owner의 유효한 액션은 충돌 및 실행 의존성 조건을 만족하면 모든 정직 노드에 CC가 전달된다.
- **Proposer Safety:** 제안자의 비잔틴 행위는 CC의 위조 또는 액션의 변조를 가능하게 하지 않는다.
- **Derived-State Agreement:** 동일한 입력 버전 벡터와 동일한 결정적 함수가 주어지면 모든 정직 노드는 같은 파생 공유 상태를 계산한다.

---

## 9. 정리와 증명 의무

다음 정리는 명세상 목표이며, 최종 논문에서는 각 가정과 상태 전이 규칙에 대한 엄밀한 증명이 필요하다.

### T1. Validity

유효한 CC의 액션은 유효한 Owner 서명과 형식 검증을 통과하며 비잔틴 노드만으로 CC를 만들 수 없다.

**증명 개요.** CC에는 서로 다른 노드의 서명 $f+1$개 이상이 필요하다. 비잔틴 노드는 최대 $f$개이므로 최소 하나의 정직 서명이 필요하다. 정직 노드는 검증을 통과한 액션에만 서명한다.

### T2. Uniqueness

충돌 저항성에 의해 하나의 `proposeId`는 하나의 액션에 대응한다.

### T3. Totality / Propagation

정직 노드 중 가장 먼저 유효한 액션 또는 Fast CC를 받은 시각을 $t$라 하면 모든 정직 노드는 $t+\Delta'$ 안에 이를 받는다.

**증명 개요.** 최초 수신 정직 노드는 검증 후 한 번 전체에 전파한다. 전파는 상태 의존 조건에 막히지 않는다.

### T4. Fast Order Uniqueness

같은 `(V, ver)`에서 서로 다른 두 액션은 모두 정직 투표를 받을 수 없다. 단 $W\geq\Delta'$이다.

**증명 개요.** 두 충돌 액션의 최초 정직 목격 시각을 각각 $a,b$라 한다. T3에 의해 각 액션은 최초 목격 후 $\Delta'$ 이내에 모든 정직 노드에 전파된다. 정직 노드가 한 액션에 투표하려면 최초 목격 후 최소 $W$를 기다려야 한다. 두 액션 모두 정직 투표를 받는다고 가정하면 기존 Fast uniqueness의 시간 부등식에서 $W<\Delta'$가 도출되어 모순이다. Read dependency에 따른 투표 지연은 투표를 더 늦출 뿐이며, 충돌 감지 후 투표를 허용하지 않는다.

### T5. Read Safety

실행되지 않은 버전을 참조하는 액션은 투표되지 않으며, 실행 완료 버전만 참조하는 의존성 그래프에는 실행 대기 순환이 생성되지 않는다.

**증명 의무.** 액션이 참조하는 버전의 실행 완료가 액션의 투표 자격보다 선행한다는 규칙을 모든 정직 노드가 동일하게 적용해야 한다. Read 참조는 이미 완료된 실행 사건을 향하므로 이후 액션의 실행을 기다리는 역방향 의존성이 형성되지 않는다.

### T6. Fast Liveness

정직 Owner가 제출한 유효 액션에 대해, 해당 액션이 처음 정직 노드에 도달하기 전에 모든 Read dependency가 모든 정직 노드에서 실행 완료되어 있고 동일 `(V, ver)` 충돌이 없다면, 정직 노드 하나가 액션을 처음 받은 시각 $a$로부터 $a+2\Delta'+W$ 이내에 모든 정직 노드가 CC를 보유한다.

**증명 개요.** 모든 정직 노드는 $a+\Delta'$ 이내에 액션을 받고, 이후 $W$ 이내에 투표한다. 투표는 $\Delta'$ 이내에 전파된다. 정직 노드가 최소 $f+1$개이므로 비잔틴 협조 없이 CC를 구성할 수 있다. Read dependency가 모든 정직 노드에서 미리 실행 완료되었다는 조건이 중요하다. 이 조건이 충족되지 않은 액션에는 위 시간 상한을 적용하지 않는다.

### T7. Convergence

모든 정직 노드는 각 변수에 대해 같은 상태에 수렴한다.

**증명 의무.**
1. 유효 액션의 전파(T3).
2. 동일 order의 액션 유일성(T4).
3. 버전별 실행 순서의 동일성.
4. Read dependency의 실행 완료 및 동일 버전 참조(T5).
5. 구현체 결정성 계약.
6. 동일한 초기 상태.

위 조건이 충족되면 각 변수의 버전 열은 모든 정직 노드에서 동일한 결정적 상태 전이를 만든다.

### T8. CC Soundness — Honest Owner

정직 Owner가 서명한 액션에 대해 유효한 CC가 존재하면, 해당 액션은 모든 정직 노드에서 지정된 order로 실행된다. 단, Read dependency 및 이전 버전의 실행 조건이 충족되어야 한다.

### T9. Order Authenticity

동일한 `(V, ver)` order에 서로 다른 액션의 유효 CC가 존재하지 않는다.

**증명 개요.** 각 CC에는 최소 하나의 정직 서명자가 존재한다(($f+1$개 서명 중 비잔틴 서명은 최대 $f$개)). 두 CC가 서로 다른 액션을 증명한다고 가정하면, 각 액션은 해당 order에서 정직 노드의 투표를 하나씩 이상 받아야 한다. 이는 T4의 Fast Order Uniqueness에 모순이다. 단순한 쿼럼 교집합만으로는 충분하지 않다. $n=2f+1$, quorum $f+1$에서는 두 쿼럼의 교집합이 비잔틴 노드만으로 구성될 수 있기 때문이다.

### T10. OP Determinism

모든 정직 노드의 변수 정의 테이블은 동일하다.

**증명 개요.** OP 연산은 일반 액션이며 변수별 단일 order로 실행된다. 선행 조건과 효과가 Core에서 결정적으로 정의되므로 동일한 초기 상태와 동일한 액션 열에서 같은 정의 테이블이 도출된다.

### T11. Derived-State Agreement

모든 정직 노드가 동일한 입력 변수 버전 벡터를 사용하고 동일한 결정적 함수 $F$를 실행하면 파생 공유 상태가 동일하다.

$$
X_i^{v_i}=X_{i,\ honest}^{v_i}
\quad\forall i
\implies
F(X_1^{v_1},\ldots,X_m^{v_m})
\text{ is identical}
$$

이 정리는 파생 함수의 결정성과 입력 버전 일치를 전제로 한다. 전역 스냅샷을 자동으로 제공하는 정리는 아니다.

---

## 10. Core와 Blocks

### 10.1. Core 계약

| # | 계약 | 근거 |
|---|---|---|
| C1 | 자기검증 CC: $f+1$개 서명이 액션과 order를 증명 | T1, T9 |
| C2 | 전파: 정직 노드가 본 액션과 Fast CC는 $\Delta'$ 안에 모든 정직 노드가 봄 | T3 |
| C3 | 순서: 변수마다 하나의 버전 열, 변수 간 기본 전역 순서 없음 | T4, T9 |
| C4 | 결정적 실행: `apply(action,pinned)`는 구현체 결정성 계약을 준수 | T7 |
| C5 | 진행: 충돌이 없고 Read dependency가 실행된 정직 Owner 액션은 상수 시간 내 CC 획득 | T6 |
| C6 | 진행 커서: `cursor = {V → lastVer[V]}` | 6장 |
| C7 | OP 유일성: 변수 정의와 Owner는 CC로 재구성 가능 | T10 |
| C8 | 단일 변수 액션: 한 액션은 하나의 변수만 수정 | 3.3 |

### 10.2. 블록 인터페이스

```text
Admission.onAction(act)             -> accept | delay
Persistence.onCC(cc), .onExecuted(order)
Application.apply(act, pinned)      -> delta | no-op
Recovery.fetch(cursor, peer)         -> [CC]
Checkpoint.propose(cursor, digest), .onStable(cp)
ResultQuery.get(V, ver)              -> value
DerivedQuery.get(F, versionVector)   -> derived value
Reaction.watch(V_src, rule)          -> 후속 액션 제출
Gateway.submit(act), .retry()        -> CC
Audit.record(cc, execution)
```

### 10.3. 블록 제약

| # | 제약 |
|---|---|
| B1 | 블록은 검증된 액션과 Fast CC의 전파를 막거나 늦출 수 없다. |
| B2 | 투표를 거부하는 정책은 모든 정직 노드에서 동일한 결정적 규칙이어야 한다. 노드별 로컬 요율 제한은 지연 큐로만 구현한다. |
| B3 | 블록은 CC의 order를 바꾸거나 Core order 규칙 밖에서 `apply`를 호출할 수 없다. |
| B4 | 피어 또는 저장소에서 가져온 상태 데이터는 CC 및 실행 이력 검증을 통과해야 한다. |
| B5 | 상태에 영향을 주는 모든 블록은 결정적이어야 한다. |
| B6 | Reaction은 후속 액션을 일반 액션으로 제출하고 Owner 서명 및 단일 변수 규칙을 우회할 수 없다. |
| B7 | 파생 상태 계산은 입력 버전 벡터를 명시해야 하며, 서로 다른 버전 벡터의 결과를 동일 스냅샷으로 취급해서는 안 된다. |

### 10.4. 블록 카탈로그

| 블록 | 역할 | 의존 Core 보장 |
|---|---|---|
| Failure Recovery | cursor 및 피어 상태를 비교하고 누락 CC를 받아 검증 후 재실행 | C1, C3, C6, C7 |
| Checkpoint & GC | cursor, 정의 테이블, 묘비 및 상태 digest를 체크포인트하고 과거 데이터 정리 | C1, C4, C6, C7 |
| Application | 업무 규칙과 결정적 `apply` 구현 | C4 |
| Admission / Rate Limit | 신원별 요율 제한 및 등록 정책 | C2, C5 |
| Declaration Policy | 변수 이름, 선언 자격, 쿼터 및 수수료 | C4, C7 |
| Identity / Key Rotation | 키 교체 및 Owner 식별 정책 | C3, C4 |
| Result Query | 실행된 변수 버전의 값 조회 | C1, C4 |
| Derived Query | 명시된 버전 벡터를 입력으로 파생 상태 계산 | C4, B7 |
| Reaction | 원인 변수의 실행을 관찰하고 후속 액션 제출 | C1, C3, C8, B6 |
| Gateway | 제안자 선택, 타임아웃 및 재시도 | T7 |
| Audit | CC 및 실행 이력 보관 | C1 |

### 10.5. 조립 규칙

- 각 블록은 요구하는 Core 보장, 제공 인터페이스 및 준수 제약을 명시한다.
- 블록 간 의존은 Core 계약을 통해서만 맺는다.
- 블록을 제거하거나 교체해도 Core 안전성에 영향을 주어서는 안 된다. 다만 활성성, 가용성 및 파생 상태의 최신성은 영향을 받을 수 있다.
- Recovery를 위해 필요한 CC와 이력의 보관 기한은 Checkpoint 블록이 정한다.

---

## 11. 한계와 열린 항목

### 11.1. 한계

- Fast Path의 유일성은 동기 가정 $W\geq\Delta'$에 의존한다.
- 변수 사이에는 기본 전역 순서가 없다. 서로 다른 변수의 상태를 읽는 스냅샷은 명시적인 버전 벡터 없이는 동일하다고 보장할 수 없다.
- 파생 공유 상태는 동일한 입력 버전 벡터와 결정적 함수가 주어질 때만 동일하다.
- 파생 상태의 자동 최신화는 Core가 보장하지 않는다.
- Owner가 동일 `(V, ver)`에 이퀴보케이션하거나 버전 구멍을 만들면 해당 변수 레인이 영구 정지할 수 있다.
- Owner가 오프라인이면 해당 변수의 새 액션 제출이 중단된다. Leaderless 구조는 Owner 자체의 가용성까지 제거하지 않는다.
- 소유권이 명시된 변수 모델은 모든 다중 주체 연산의 원자성이나 전역 불변식을 자동으로 제공하지 않는다.
- 결정적 `apply`는 구현체가 계약을 지켜야 한다. 구현체의 비결정성은 상태 수렴을 깨뜨릴 수 있다.
- 여러 변수에 걸친 원자적 갱신은 Core의 기본 기능이 아니다.
- 동적 target, 즉 실행 중 계산되는 대상 변수는 지원하지 않는다.

### 11.2. 열린 항목

1. 변수별 보류 창 크기 $w$와 버전 구멍의 폐기 규칙.
2. 이력 보관 기한과 묘비 유지 정책.
3. Checkpoint 규격: 간격, 상태 digest, 정리 이후 도착한 CC의 결정적 처리.
4. Read 참조가 오래된 이력에 접근할 때의 보관 및 증명 정책.
5. 블록 인터페이스의 정식 명세와 호환성 검증.
6. Admission 및 Declaration Policy에서 스팸과 변수 대량 생성을 막는 결정적 규칙.
7. Owner 장애 및 키 분실 시 소유권 이전 절차.
8. 파생 공유 상태의 버전 벡터 구성 및 조회 일관성.
9. 여러 변수에 걸친 원자적 불변식이 필요한 애플리케이션의 설계 패턴.
10. 선행 연구와의 비교 및 통신 복잡도 분석.

---

## 12. 안정성 검증 및 TLA+ 계획

후속 검증은 수학적 증명과 모델 검증을 병행한다. TLA+ 모델 검증은 유한 상태 범위에서 상태 전이 및 불변식 위반을 탐색하는 수단이며, 일반적인 무한 실행이나 모든 구현에 대한 수학적 증명을 대체하지 않는다.

### 12.1. 우선 검증할 안전성 불변식

- **TypeOK:** 모든 상태 변수와 메시지가 정의된 타입을 만족한다.
- **AtMostOneActionPerOrder:** 동일 `(V, ver)`에 대해 정직 노드가 상충 액션을 커밋하지 않는다.
- **ValidCommitCertificate:** CC에는 서로 다른 노드의 유효 서명 $f+1$개가 존재한다.
- **OwnerAuthorization:** 액션 작성자는 target 변수의 Owner와 일치한다.
- **MonotonicVersion:** `lastVer[V]`는 감소하지 않는다.
- **SingleVariableMutation:** 하나의 액션은 하나의 변수만 수정한다.
- **ReadExecutedBeforeVote:** Read dependency가 모두 실행되기 전에는 액션에 투표하지 않는다.
- **ReadVersionStable:** 실행된 버전 참조는 해당 버전의 고정된 값을 사용한다.
- **DeterministicApply:** 동일한 상태, 액션 및 pinned input은 동일한 결과를 만든다.
- **StateConvergence:** 모든 정직 노드는 동일한 변수별 실행 이력에 대해 동일한 상태를 가진다.
- **DerivedStateAgreement:** 동일한 버전 벡터와 함수로 계산한 파생 상태는 동일하다.

### 12.2. 검증할 활성성 속성

- 정직 Owner의 충돌 없는 유효 액션은 모든 Read dependency가 실행된 후 CC를 얻는다.
- 유효한 액션 및 Fast CC는 모든 정직 노드에 전파된다.
- 하나의 변수에서 발생한 충돌은 다른 변수의 진행을 막지 않는다.
- 버퍼링된 액션은 선행 버전 및 Read dependency가 실행되면 실행 가능 상태가 된다.

### 12.3. 모델링할 공격

1. 동일 `(V, ver)`에 대한 Owner equivocation.
2. 비잔틴 노드의 선택적 전파 및 침묵.
3. 서로 다른 시점에 도착하는 충돌 액션.
4. 잘못된 Read version 및 미실행 Read reference.
5. 버전 구멍과 중복 액션.
6. 재시작 및 Checkpoint 이후의 늦은 CC.
7. 비잔틴 Owner의 액션 제출 중단.
8. 여러 변수에 걸친 파생 상태 계산에서 서로 다른 버전 벡터 사용.

---

## 13. 프로토콜 요약

| 구성 | 역할 |
|---|---|
| 노드 모델 | $n=2f+1$, 최대 $f$ Byzantine |
| Owner | 각 변수의 유일한 수정 권한자 |
| Action | 하나의 피소유 변수에 대한 OP 또는 WR 연산 |
| Order | `(V, ver)` 변수별 버전 위치 |
| 전파 | 유효한 액션과 Fast CC를 수신하면 한 번 전체 전파 |
| Fast 투표 | 첫 목격 후 최소 $W\geq\Delta'$, 충돌이 없고 Read dependency가 실행된 경우 |
| CC | `{action, order, proof}` 자기검증형 커밋 증명 |
| 실행 | 이전 버전 및 모든 Read dependency 실행 후 결정적으로 적용 |
| 공유 상태 | 여러 피소유 변수와 명시된 버전 벡터의 결정적 함수 |
| 합의 경로 | Fast Path 단일 경로 |
| Blocks | Recovery, Checkpoint, Application, Admission, Declaration Policy, Identity, Result Query, Derived Query, Reaction, Gateway, Audit |

## 14. 핵심 정의

Salix는 피소유 변수의 버전 지정 상태 전이에 대해 Leaderless Byzantine Consensus를 제공한다. 공유 상태는 여러 피소유 변수의 결정적 함수로 표현할 수 있으며, Core는 공유 상태 자체를 별도의 합의 대상으로 요구하지 않는다.

$$
\boxed{
\text{Consensus on owned state transitions}
\quad\Longrightarrow\quad
\text{Deterministically derived shared state}
}
$$

이 명제는 동일한 입력 버전, 결정적 함수 및 애플리케이션 불변식이 충족되는 범위에서 적용된다.
