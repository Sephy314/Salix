# Salix, BFT Leaderless Majority Quorum Constant Round Consensus

## What is Salix?

A protocol that enables reaching consensus in constant rounds using only a majority quorum in BFT consensus.

Key features include: a leaderless consensus design that resolves the difficulty of distributing centralized writes in traditional View-Change approaches; the use of digital signatures so that consensus can be reached with more than $f+1$ honest nodes, below the $2f+1$ lower bound for oral messages; and a constant-round structure with no fallback—unlike similar prior works that rely on Dolev-Strong-based fallbacks with high communication complexity—making it relatively lightweight even in the worst case.

## Assumptions

* Uses a partial synchrony assumption; consensus completion cannot be guaranteed before GST.

* Assumes $n > 2f + 1$, with a Quorum size of $f + 1$.

* Clients may also be Byzantine.

* Byzantine behaviors include crash, collusion, silence, equivocation, false requests, replay, and similar faults.

* Public keys are mutually verifiable, and unforgeable digital signatures are used.

* The arrival order of client requests is not guaranteed.

* Fail Recovery is not guaranteed, though it can be added at the implementation layer.

* Hash conflicts (collisions) are assumed not to occur.

## Guaranteed Properties

Salix guarantees the following properties:

1. Safety: Every honest node observes every valid proposal and eventually applies it to its State Machine.

2. Liveness: Even under Byzantine interference, consensus is unaffected and terminates within finite time.

3. Validity: Byzantine actors cannot cause an invalid value to be committed.

## Data Structure Definitions

The following defines the data structures required for consensus.

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

Note: This is pseudocode for illustration only.

## Broadcast Mechanisms

Salix uses two communication methods.

### Type-1 Broadcast

```
on Type1Broadcast(data)
    for i in all nodes
        i.send(data)

```

A method for sending data to all nodes. If the round-trip communication time is $t$, it takes time $t$ until a response is received.
A single node broadcasts to all other nodes except itself. When an honest node performs a Type-1 broadcast, all nodes become aware of the broadcast.

### Type-2 Broadcast

```
on Type2Broadcast(data)
    if broadcasted(data)
        return
    
    Type1Broadcast(data)

on sent(data)
    Type2Broadcast(data)

```

A broadcast mechanism in which every node broadcasts to all other nodes. Each node receives a broadcast and, if it has not already broadcast the data, performs a Type-1 broadcast. As long as there is at least one honest node, all nodes will hear the broadcast via Type-1 broadcast. If the round-trip communication time is $t$, it takes a total of $2t$.

## Consensus Specification

### Fetch

> Fetch() -> (latestCC)

This stage updates a node's State Machine to the latest state before preparing for consensus. It performs Type-1 communication and generates the `Sequence` based on the highest-version valid certificate (with valid signatures) among the received CCs.

### Propose

> Propose(request) -> ()

Propose is the starting point of consensus. A client selects an arbitrary node as the Proposer and provides the `ClientRequest` filled with all fields except those wrapped by the Proposer. The Proposer validates it, inserts the `Sequence`, creates a `Request` based on the `ClientRequest`, and performs a Type-1 broadcast to all nodes.

### Prepare

> Prepare(request) -> (request)

If the Proposer is Byzantine, not all honest nodes may receive the Propose message. This stage guards against a Byzantine node performing equivocation or remaining silent. A node that receives a Propose executes Prepare by performing a Type-2 broadcast to all nodes. By the definition of Type-2 broadcast, a request that has already been broadcast is not broadcast again.

### Vote

> Vote(request, signature) -> (request, signature)

This stage forms the signature set. Here, each node performs validation on the request—checking whether `Sequence` matches its State Machine, whether `ProposeId` is valid, whether the same request has already been received, and so on. If valid, the node creates its own signature to contribute to the signature set. The signature set requires signatures from $f+1$ nodes. Matching CCs are defined as two valid CCs that share the same `Request` (including `ProposeId`), regardless of which nodes contributed the signatures.

### Commit

> Commit(request, commitCert) -> (request, commitCert)

This stage ensures state consistency among honest nodes even if Byzantine nodes only partially transmit signatures that contribute to a CC. In this stage, a node combines $f+1$ valid signatures acquired during the Vote stage with the `Request` to form a CC, then propagates this CC via Type-2 broadcast. A CC corresponding to a given `Request` is sent only once. That is, as stated above, even if the signatures differ, two CCs with the same `Request` are considered matching, and even if the contributing nodes differ, as long as all signatures are valid, a valid CC is shared only once. Upon obtaining a valid CC in this stage, every node records it, applies it to its State Machine, and increments `Sequence` by one. Honest nodes that receive it report the result to the client and terminate consensus.

### Phase Transitions

Phase transitions are executed locally: once $f+1$ responses supporting the same response are received, the node advances to the next phase. For requests from a previous phase, the node does not change its state and returns the value it previously responded with. For requests from a phase ahead of the node's current phase, the node returns a `retry` response. This does not constitute a claim; an honest node receiving this response handles retries according to its implementation-level policy.

### Sequence Rules

`Sequence` numbers are fundamentally ordered by magnitude comparison, though the protocol intentionally allows multiple consensus instances to correspond to the same `Sequence`. Under the protocol's assumptions, consensus does not always reflect the order of client requests—specifically, this applies only to requests submitted concurrently before a consensus instance completes. Nevertheless, because consensus must guarantee Agreement, ordering among them is determined by comparing the hashes of their `ProposeId` values.

### ProposeId and RequestId

One `ProposeId` corresponds to exactly one `Request`. According to the definition of `ProposeId` above, it is the hash of `clientSignature`, `clientRequest`, and `requestId`, which is verified during validation. `RequestId` is a mechanism to allow the same request to be submitted twice when needed at the implementation level. Repeatedly sending the same request with only a changed `RequestId`, as well as exceptions such as DDoS, are not treated as Byzantine actions within the consensus protocol; countermeasures for these are required at the implementation layer.

## Demonstration

### Assumptions

This section specifies the assumptions and guarantees required to prove the Liveness and Safety of the protocol. This protocol guarantees Liveness and Safety, whose mathematical properties are as follows:

|  |  | 
| ----- | ----- | 
| Network | Partial Synchronous | 
| Cryptography | Digital Signature | 
| Nodes | 2f+2 (minimal) | 
| Quorum | f+1 | 
| Byzantine Node Actions | Collusion, Equivocation, Silent, Crash | 
| Byzantine Client Actions | Collusion, Replay, Malformed Request | 

* Liveness
  

  $$
  \forall i \in H, \forall r \in R_{valid} : Propose(r) \Rightarrow \Diamond Decided_i(r)
  $$

  
  This means that for any valid request, once it is proposed, all honest nodes will eventually decide on that request.

* Safety
  

  $$
  \forall i, j \in H, \forall r \in R_{valid} : Decision_i(r) = Decision_j(r)
  $$

  
  This means that all honest nodes always reach the same decision value for any given request.

### Lemma A. Property of Type-2 Broadcast

> $\forall i, j \in HonestNodes, \forall k \in AllNodes, \forall r \in Request : T2Broadcast_k(r) \land Observed_i(r) \Rightarrow Observed_j(r)$

If node $k$ performs a Type-2 broadcast and at least one honest node observes it, then all nodes can observe that value.

Proof:

When an honest node observes a value subject to Type-2 broadcast, by the protocol rules it performs a Type-1 broadcast, and by the definition of Type-1 broadcast it is propagated to and learned by all nodes. $\blacksquare$

Through this property, in the Prepare stage, if any node sends Prepare to at least one honest node, all honest nodes learn of the request's existence; and in the Commit stage, if at least one honest node observes a CC, all nodes learn of that CC.

### Lemma B. Lower Bound $n>2f+1$ Under the Current Protocol Rules

> $\vert{}N\vert{} > \vert{}Q\vert{} + \vert{}F\vert{}$

The total number of nodes must always be strictly greater than the sum of the quorum size and the number of potentially faulty nodes.

Proof:

By Lemma A, if any node propagates a value via Type-2 broadcast to at least one honest node, all nodes become aware of it. Moreover, to accept a value, a node requires supporting votes from at least $q=f+1$ nodes excluding itself. If a Byzantine node forwards a request to only one honest node, then by the protocol rules, the remaining honest nodes must provide up to $f+1$ supporting honest claims (excluding themselves) for acceptance. Additionally, the number of nodes must be a natural number, and by the protocol rules only one vote per node is counted. Assuming Byzantine nodes make claims opposing the honest nodes—that is, obstructing honest nodes as much as possible:

$$
q \leq n - f - 1 \\ q + f + 1 \leq n \\ \therefore n > q + f \iff n > 2f + 1 (\because n, q, f \in \mathbb{N})
$$

Thus the inequality holds. Within the protocol, the places where Quorum is used critically are Phase transitions, Prepare, Vote, and the CC Threshold count. The first three can be proven using the above result, and the last holds because $q > f$ is always satisfied. $\blacksquare$