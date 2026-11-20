# Salix, BFT Leaderless Majority Quorum Constant Round Consensus

## Salix란?
기존 BFT 합의에서, 절반 이상 쿼럼만으로 상수 라운드 내에 합의에 도달할 수 있게 해주는 프로토콜.

특징으론, 기존의 View-Change 방식은 Leader의 중앙화된 Write 분산의 어려움을 해결하기 위한 leaderless 기반의 합의, 디지털 서명을 사용하여 정상 노드가 오랄 메시지의 하한인 2f+1개 미만인 f+1개 초과 조건만으로 합의 도달 가능, 기존 비슷한 연구들은 Fallback으로 Dolev-Strong 기반의 것을 사용하여 높은 통신 복잡도를 가졌지만 이는 항상 상수 라운드에 Fallback이 없는 형태라 최악시 상대적으로 가벼움.

## 가정
- 부분 동기식 가정을 사용 한다. GST 이전에는 합의 완료를 보장 할 순 없다.
- n > 2f+1를 가정한다. Quorum은 f+1이다.
- 클라이언트 또한 비잔틴일 수 있다.
- 비잔틴 행동은 크래시, 담합, 사일런트, 이퀴보케이션, 거짓 요청, 리플레이등이 있다.
- 공개키는 서로 확인이 가능하고, 위조 될 수 없는 서명인 디지털 서명을 사용한다.
- 클라이언트 요청 도착 순서는 보장되지 않는다.
- Fail Recovery는 보장되지 않으나 구현체 단에서 추가할 수 있다.
- Hash Conflict는 가정하지 않는다.

## 보장 성질
Salix는 다음 성질을 보장한다.
1. Agreement: 모든 정직 노드는 정직한 제안을 보고 최종적으로 이를 자신의 State Machine에 적용한다.
2. Liveness: Byzantine이 영향을 주더라도 합의는 영향을 안 받으며 합의는 유한한 시간내 종료된다.
3. Validity: Byzantine이 부정한 값을 commit 시킬 수 없다.

## 자료 구조 정의
다음은 합의에 필요한 자료 구조를 정의한다.
```
ProposeId: H(clientSignature, clientRequest, requestId)

ClientRequest {
    ProposeId
    RequestId
    ClientId
    ClientSignature
    RequestBody
}

Request {
    ClientRequest

    Sequence 
    ProposerSignature
}

CommitCert {
    Request
    Threshold [{
        NodeId
        NodeSignature
    }, ...]
}

```
참고, 이는 이해만을 위한 Psuedo Code이다.

## 전파 기법 정의
Salix는 2개의 통신 방법을 사용한다.
### 제 1종 전파
```
on Type1Broadcast(data)
    for i in all nodes
        i.send(data)
```
모든 노드에게 보내는 방법이다. 왕복 통신 시간이 t라면 응답까지 t의 시간이 걸린다
한 노드가 자신을 제외한 다른 노드에게 전파를 하는 전파 방법이다. 정직 노드가 제 1종 전파를 수행하면, 모든 노드가 이 전파의 존재를 알게 된다. 

### 제 2종 전파
```
on Type2Broadcast(data)
    if broadcasted(data)
        return
    
    Type1Broadcast(data)

on sent(data)
    Type2BroadCast(data)
```
모든 노드가 다른 모든 노드들에게 전파를 하는 전파 기법이다. 모든 노드는 전파를 받고 자신이 전파를 하지 않았다면 제 1종 전파를 수행한다. 정직 노드가 최소 1개 이상 있을시, 제 1종 전파에 의해 모든 노드는 이 전파를 듣게 된다. 왕복 통신 시간이 t라면 총 2t의 시간이 걸린다.


## 합의 명세
### Fetch
> Fetch() -> (latestCC)
이 단은 합의 준비 전, 자신의 State Machine을 최신화 하는 작업이다. 제 1종 통신을 수행하며, 응답 받은 CC중 가장 버전이 높은 서명이 옳은 인증서를 기준으로 Sequence를 생성한다. 

### Propose
> Propose(request) -> ()
Propose는 합의의 시작점이다. 클라이언트는 임의의 노드를 Proposer로 선정후, Proposer가 추가로 감쌀 부분외 데이터를 채워서 제공을 한다. Proposer은 이의 정당성을 검정후 Sequence 삽입후 client request기반으로 request 생성 후, 모든 노드들에게 제 1종 전파를 수행한다.

### Prepare
> Prepare(request) -> (request)
Proposer가 비잔틴이면, 모든 정직 노드가 Propose를 듣지 못 할 수도 있다. 비잔틴 노드가 Equivocation을 행하거나 Silent를 대비하기 위한 단이다. Propose를 받은 노드는 Prepare을 수행하며 제 2종 전파를 모든 노드들에게 수행한다. 제 2종 전파의 정의에 의해, 한번 전파한 요청은 다시 전파하지 않는다.

### Vote
> Vote(requet, signature) -> (request, signature)
이 값은 서명 집합을 형성하기 위한 단계이다. 이때, 노드는 요청에 대한 Validate 요청을 수행한다. 이 Validate 검정은, Seq가 자신의 SM과 일치 하는지, ProposeId가 올바른지, 이미 왔던 동일한 요청인지등을 검정한 후, 옳을 시 서명 집합에 기여할 자신의 서명을 만든다. 서명 집합에는 f+1개의 노드의 서명이 필요하며, 일치하는 CC란, 서명에 기여한 노드에는 상관 
### Phase 전환
이는 로컬로 실행되며, 같은 응답을 지지하는 f+1개의 응답이 오면, 다음 페이즈로 넘어간다. 이전 페이즈에 대한 요청은 바뀌지 않으며 자신이 응답했던 값을 보내주며, 자신이 현재 위치한 페이즈의 미래의 페이즈에 대한 요청은 retry응답을 준다. 이는 어느 주장도 아니며 이 응답을 받은 정상 노드는 추후 구현체 정책에 따라 재요청 처리를 한다. 

### Sequence 규칙
Sequence는 기본적으로, 대소 비교를 하나, 프로토콜 상에서는 같은 시퀀스에 여러 합의가 해당될 수 있으며, 이는 의도된 사항이다. 프로토콜 가정에 따라, 합의가 클라이언트의 요청 순서는 항상 반영하지 않으며, 여기서의 자세한 조건은, 어떤 합의가 끝나기 전에 동시에 요청된것에 한한것이다. 그럼에도 불구하고, 합의는 Agreement를 보장해야하므로, proposeId를 해시하여 대소비교를 통하여 순서를 정한다. 

### ProposeId와 RequestId
한 ProposeId는 오직 하나의 Request에만 해당한다. ProposeId의 앞서 정의된 정의에 따르면, Request와 SignatureId와 RequestId를 해시한 값이므로, Valid 계산 시점에서, 이가 맞는지 확인한다. RequestId는 같은 요청을 구현체상 두번 해야하는경우를 대비해, 이를 허용하기 위한 수단이다. 같은 요청을 requestId만 바꿔 반복 요청하는 경우 및 DDOS등의 예외는, 합의 프로토콜에서 Byzantine Action으로 보지 않고, 이에 대한 대책은 Impl단에서의 제한을 요구한다.


## Demonstration
### Assumptions
이 단락은 프로토콜의 Liveness와 Safety를 증명하기 위해 필요한 가정과 보장할 것을 명시하는 단이다. 이 프로토콜은 Liveness와 Safety를 보장하며, 이의 수학적 성질은 다음과 같다. 

| | |
|---|---|
| Network | Partial Synchronous |
| Crytography | Digital Signature |
| Nodes | 2f+2 (minimal) |
| Quorum | f+1 |
| Byzantine Node Actions | Collusion, Equivocation, Silent, Crash |
| Byzantine Client Actions | Collusion, Replay, Malformed Request |

- Liveness
    $$
    \forall i \in H, \forall r \in R_{valid} : Propose(r) \Rightarrow \Diamond Decided_i(r)
    $$
    이는 모든 정직 노드들은 어느 요청이 올바르다면 Propose시 언젠가는 그 요청이 결정된다는 뜻이다.

- Safety
    $$
    \forall i. j \in H, \forall r \in R_{valid} : Decision_i(r) = Decision_j(r)
    $$
    이는 모든 정직 노드들은 어떤 요청에 대한 결정값은 항상 동일하다는 뜻이다.

### Lemma A. 제 2종 전파의 성질
> $\forall i, j \in HonestNodes, \forall k \in All Nodes, \forall r \in Request : T2Broadcast_k(r) \land Observed_i(r) \Rightarrow Observed_j(r) $

k가 제 2종 전파를 수행했고, 그로 인해 정직 노드가 적어도 하나라도 관측되었으면, 모든 노드가 그 값을 관측 할 수 있다. 

증명:  
정직 노드가 제 2종 관측에 해당되는 값을 확인하면, 규칙에 의해 제 1종 전파를 수행하고, 제 1종 전파의 정의에 의해 모든 노드한테 전파되고 알게 된다. $\blacksquare$

이를 통해, Prepare을 통하면, 어느 노드라도 Prepare을 정직 노드 적어도 하나한테 보냈다면, 모든 정직 노드가 Prepare단에서 요청의 존재를 알 수 있고, Commit단에서, 이를 발견한 정직 노드가 최소 하나라도 있다면, 모든 노드가 이 CC를 알 수 있다.

### Lemma B. 현 프로토콜 규칙 내에서 $n>2f+1$ 하한
> $|N| > |Q| + |F|$

전체 노드의 수는 쿼럼과 폴트 가능한 노드의 수보다 항상 많아야만 한다.

증명:  
Lemma A에 따르면, 어느 노드더라도, 제 2종 전파를 통해 적어도 하나의 정직 노드에게 전파가 되었으면, 이는 모든 노드가 알 수 있다. 그리고, 어떤 값을 수용하려면, 자신을 제외한 $q=f+1$ 이상의 노드의 지지표가 필요하다. 비잔틴 노드가 어떤 정직 노드 하나에게만 요청을 넘기면, 프로토콜 규칙에 따라, 다른 정직 노드들은 자신 제외 최대 f+1개의 정직 노드의 주장이 있어야 수용이 된다. 또한, 노드수는 자연수여야 한다. 추가로, 프로토콜 규칙에 따라 노드당 하나의 표만 인정된다. 비잔틴은 정직 노드에 반한 주장을 한다 가정하면, 즉 최대한 정직 노드를 방해한다 가정하면, 

$$
q \leq n - f - 1 \\
q + f + 1 \leq n \\
\therefore n > q + f 
\iff n > 2f + 1 
(\because n, q, f \in \N)
$$
다음이 성립한다. 프로토콜 내에서, Quorum이 유의미하게 쓰이는 부분은, Phase 전환, Prepare, Vote, CC Threshold 갯수 부분이다. 앞 3개는 이를 이용해 증명 가능하고, 마지막 경우는 $q > f$ 가 항상 성립하므로 된다. $\blacksquare$
 
