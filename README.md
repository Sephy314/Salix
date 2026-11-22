# Salix: Leaderless BFT Consensus with Majority Quorums and Constant Rounds

## What is Salix?

Salix is a consensus protocol designed to achieve consensus within a constant number of rounds in Byzantine Fault-Tolerant (BFT) systems using quorums of only half the nodes.

Its key characteristics include:

- **Leaderless consensus:** Unlike conventional view-change-based approaches, Salix adopts a leaderless architecture to address the centralised write-distribution bottleneck associated with a leader.
- **Reduced quorum requirements:** By employing digital signatures, Salix allows consensus to be reached with \(f+1\) signatures, rather than the conventional \(2f+1\) threshold.
- **No fallback protocol:** Similar existing approaches often rely on Dolev–Strong-based fallback mechanisms, which introduce high communication complexity. Salix does not require such a fallback and maintains a constant number of rounds even in the worst case, making it comparatively lightweight.

## Assumptions

- A synchronous network model is assumed. Consensus completion is not guaranteed before the Global Stabilisation Time (GST).
- The system assumes \(n > 2f+1\), where the quorum size is \(f+1\).
- Clients may also be Byzantine.
- Byzantine behaviours include crashes, collusion, silence, equivocation, malicious requests, and replay attacks.
- Digital signatures are used, with mutually verifiable public keys and an unforgeability assumption.
- Client request arrival order is not guaranteed.
- Fail-recovery is not guaranteed by the protocol, although it can be added at the implementation level.
- Hash conflicts are assumed not to occur.
- The log is assumed to be insertable rather than strictly append-only.

## Guaranteed Properties

Salix guarantees the following properties:

1. **Agreement:** All honest nodes eventually observe an honest proposal and apply it to their respective state machines.
2. **Liveness:** Byzantine behaviour does not prevent consensus from progressing, and consensus terminates within a finite amount of time.
3. **Validity:** Byzantine nodes cannot commit invalid values.

## Data Structure Definitions

The following data structures are required for consensus.

```text
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

Note that this is pseudocode provided solely to illustrate the data structures.

## Propagation Mechanisms

Salix employs two communication mechanisms.

### Type 1 Broadcast

```text
on Type1Broadcast(data)
    for i in all nodes
        i.send(data)
```

This mechanism broadcasts data to all nodes. If the round-trip communication time is \(t\), the response is received within \(t\).

More precisely, a node sends the data to every other node, excluding itself. When an honest node performs a Type 1 broadcast, all nodes become aware of the broadcast.

### Type 2 Broadcast

```text
on Type2Broadcast(data)
    if broadcasted(data)
        return
    Type1Broadcast(data)

on sent(data)
    Type2Broadcast(data)
```

This mechanism allows all nodes to propagate data to every other node. Each node that receives the data performs a Type 1 broadcast if it has not already broadcast the same data.

As long as at least one honest node exists, the Type 1 broadcast ensures that all nodes receive the data. If the round-trip communication time is \(t\), the entire propagation process takes at most \(2t\).

## Consensus Specification

### Fetch

> Fetch() -> (latestCC)

Fetch is a preliminary step used to synchronise Sequence values before consensus or read operations.

Before GST, Sequence values may temporarily diverge, as described in the subsequent protocol specification. Fetch is intended to mitigate this divergence.

When Fetch is invoked, each node returns its latest Commit Certificate (CC). This allows the requester to determine values such as `nextSequence`.

Fetch uses Type 1 broadcast and is not considered the beginning of a consensus instance.

### Propose

> Propose(request) -> ()

Propose marks the beginning of a consensus instance.

The client selects an arbitrary node as the Proposer and supplies the request data, excluding the fields that the Proposer must add.

The Proposer verifies the validity of the request, inserts a Sequence value, constructs a Request based on the client request, and performs a Type 1 broadcast to all nodes.

### Prepare

> Prepare(request) -> (request)

If the Proposer is Byzantine, honest nodes may not all receive the original Propose message.

This phase is intended to handle Byzantine equivocation and silence, which could otherwise prevent honest nodes from becoming aware of the proposal.

A node that receives a Propose message executes Prepare and performs a Type 2 broadcast to all nodes.

According to the definition of Type 2 broadcast, a request that has already been propagated is not broadcast again.

### Vote

> Vote(request, signature) -> (request, signature)

This phase is responsible for forming the signature collection.

During this phase, each node validates the request. Validation checks include whether the Sequence matches its local state machine, whether the ProposeId is valid, and whether the same request has already been received.

If the request passes validation, the node generates its signature to contribute to the signature collection.

A total of \(f+1\) node signatures is required.

Two Commit Certificates (CCs) are considered identical if their Requests are identical, regardless of which nodes contributed their signatures.

### Commit

> Commit(request, commitCert) -> (request, commitCert)

This phase ensures state convergence among honest nodes, even when Byzantine nodes selectively distribute signatures that contribute to their own Commit Certificates.

In this phase, \(f+1\) honest signatures obtained during the Vote phase are combined with the corresponding Request to form a CC.

The resulting CC is then propagated using Type 2 broadcast.

A CC corresponding to a particular Request is broadcast only once.

In other words, as described above, CCs are considered identical whenever their Requests match, regardless of whether their signature collections are identical. Even if different nodes contributed the signatures, a CC containing only honest signatures is shared only once.

Once a node obtains a valid CC containing honest signatures, it records the certificate, applies the corresponding request to its state machine, and increments its Sequence by one.

Upon receiving this CC, an honest node notifies the client and terminates the consensus instance.

### Phase Transition

Phase transitions are performed locally.

A node advances to the next phase when it receives \(f+1\) responses supporting the same result.

Requests concerning previous phases remain unchanged, and the node returns the value it previously responded with.

For requests concerning future phases beyond the node's current phase, it returns a `retry` response.

This response does not constitute a vote or any other form of support. An honest node receiving such a response handles the retry according to its implementation policy.

### Sequence Rules

Sequence values are generally compared numerically. However, the protocol permits multiple consensus instances to belong to the same Sequence, and this is intentional.

Under the protocol's assumptions, consensus does not necessarily preserve the arrival order of client requests. More specifically, this applies to requests submitted concurrently before an ongoing consensus instance has completed.

Nevertheless, since consensus must guarantee Agreement, the order is determined by comparing the hashes of ProposeIds.

A proposal's Sequence may be lower than the current local Sequence. In such cases, the node considers the proposal valid if its Sequence falls within the range of the current Sequence \(\pm 1\), according to the lemma presented below, using the Sequence observed before the corresponding consensus instance began.

According to the subsequent lemma, the Sequence of an honest proposal cannot be greater than the current Sequence by more than one.

When multiple consensus instances belong to the same Sequence, the Sequence is incremented only once.

### Validity Verification

When a node receives a proposal, it performs validity verification.

This includes checking the ProposeId hash, the current phase, the signature, and any other explicitly specified conditions.

Validation is a no-op action in the sense that honest nodes produce the same result when given identical inputs and state.

Hash collisions and identifier collisions are assumed not to occur.

### ProposeId and RequestId

A single ProposeId corresponds to exactly one Request.

According to the definition of ProposeId above, it is computed by hashing the client signature, client request, and RequestId. This value is verified during validity checking.

RequestId exists to allow the same logical request to be submitted more than once when required by the implementation.

Repeated submissions of the same request with different RequestIds, as well as exceptional cases such as DDoS attacks, are not considered Byzantine actions within the consensus protocol itself. Mitigating such cases requires rate limiting at the implementation level.

## Demonstration

### Assumptions

This section specifies the assumptions and guarantees required to prove the Liveness and Safety properties of the protocol.

Salix guarantees Liveness and Safety under the following mathematical assumptions.

| Category | Assumption |
|---|---|
| Network | Synchronous |
| Cryptography | Digital signatures |
| Nodes | \(2f+2\) (minimum configuration) |
| Quorum | \(f+1\) |
| Byzantine Node Actions | Collusion, equivocation, silence, crash |
| Byzantine Client Actions | Collusion, replay, malformed requests |

**Liveness**

\[
\forall i \in H, \forall r \in R_{valid} :
Propose(r) \Rightarrow \Diamond Decided_i(r)
\]

**Safety**

\[
\forall i,j \in H, \forall r \in R_{valid} :
Decision_i(r) = Decision_j(r)
\]

### Lemma A. Property of Type 2 Broadcast

\[
\forall i,j \in HonestNodes,\ \forall k \in AllNodes,\ \forall r \in Request :
T2Broadcast_k(r) \land Observed_i(r)
\Rightarrow Observed_j(r)
\]

If any node \(k\) performs a Type 2 broadcast and at least one honest node observes the broadcast, then all nodes can observe the corresponding value.

**Proof:**

When an honest node receives a value propagated through Type 2 broadcast, it performs a Type 1 broadcast according to the protocol rules.

By the definition of Type 1 broadcast, the value is propagated to and can be observed by every node. \(\blacksquare\)

Consequently, during Prepare, if a request has been propagated to at least one honest node, all honest nodes can become aware of its existence.

Similarly, during Commit, if at least one honest node discovers a CC, all nodes can become aware of that CC.

### Lemma B. Proof of the Lower Bound \(n > 2f+1\) Under the Current Protocol Rules

\[
|N| > |Q| + |F|
\]

The total number of nodes must always exceed the sum of the quorum size and the maximum number of faulty nodes.

**Proof:**

According to Lemma A, if a value is propagated through Type 2 broadcast and reaches at least one honest node, all nodes can observe it.

Furthermore, to accept a value, a node requires support from at least \(q=f+1\) nodes, excluding itself.

Suppose a Byzantine node forwards a request to only one honest node. According to the protocol rules, the other honest nodes must obtain support from at least \(f+1\) honest nodes, excluding themselves, before accepting the value.

Additionally, the number of nodes must be a natural number. According to the protocol rules, only one vote per node is counted.

Considering the worst-case scenario in which Byzantine nodes make claims that oppose the honest nodes, thereby maximally obstructing them, we obtain:

\[
q \leq n-f-1
\]

\[
q+f+1 \leq n
\]

\[
\therefore n > q+f
\iff n > 2f+1
\quad (\because n,q,f \in \mathbb{N})
\]

Therefore, the inequality holds.

Within the protocol, quorum size is used in Phase Transition, Prepare, Vote, Commit, and the CC threshold.

The first four cases can be justified using this inequality, while the final case follows from the fact that \(q>f\) always holds. \(\blacksquare\)

Furthermore, due to the properties of Type 2 broadcast, conventional quorum intersection does not need to be considered in this protocol.

### Lemma C. Upper Bound on the Time Required for Sequence Convergence

\[
\forall i,j \in H,\ \exists \tau \in [0,t+e] :
Seq_i(\tau)=Seq_j(\tau)
\]

All honest nodes converge to the same Sequence within the sum of the synchronous network bound \(t\) and the hardware execution error \(e\).

**Proof:**

Let the time at which the first honest node discovers and applies a Commit Certificate be defined as time \(0\).

According to the protocol rules, immediately after verifying and applying the value, this node broadcasts it to all other nodes.

Under the synchronous network assumption, delivery takes at most \(t\).

Considering the execution error \(e\) associated with applying the value to the state machine, the maximum time between the first honest node applying the value and the last honest node applying it is \(t+e\). \(\blacksquare\)

### Lemma D. Stability of the Optimistic Sequence Validation Range

\[
\forall i,j \in H,\ \forall r \in R_{valid} :
ValidSeq_i(r)=ValidSeq_j(r)
\]

All honest nodes determine the validity of a given honest request's Sequence consistently.

**Proof:**

According to the protocol, the Sequence-related component of the Valid function can be defined as follows.

Let \(r\) denote the request's Sequence and \(l\) denote the current local Sequence.

\[
ValidSeq(r,l)=
\begin{cases}
true & l-1 \leq r \leq l+1 \\
false & otherwise
\end{cases}
\]

This function determines whether a request is valid based on the Sequence at the time its proposal was initially submitted and the Sequence observed after the latest CC has been obtained.

According to Lemma C, the latter observation occurs within \(t+e\) of the relevant point in time.

To verify the Agreement property of this function, we first establish the following basic properties and then consider the possible cases.

**Basic Properties**

1. According to Lemma C, CC propagation and Sequence synchronisation complete within the synchronous network bound \(t\) plus the execution error \(e\).
2. Due to Fetch, the maximum discrepancy in observing the request's preceding Sequence is \(t\).
3. The maximum difference in observation time between the node that first receives request \(r\) and the node that receives it last is \(3t\), corresponding to Propose + Prepare.
4. Since a new consensus instance must be completed after the Sequence has increased before the Sequence can increase again, consensus instances responsible for Sequence increments cannot proceed in parallel. Consequently, at least \(t+2t+2t+2t=7t\) must elapse before the next Sequence increment can occur.

**Case I: \(R_{Seq} < local\)**

When the local Sequence is greater than the request's Sequence, this indicates that a Sequence increment has occurred.

In the worst case, the node receives the request at the latest possible time, which may differ by up to \(3t\) from the time at which the request was first discovered, according to Property 3.

To account for this discrepancy, the local state up to \(3t\) before the first discovery must be considered.

However, according to Property 4, consensus instances that increment the Sequence cannot proceed in parallel.

Therefore, in this case, the Sequence can differ by at most \(-1\).

**Case II: \(R_{Seq}=local\)**

This is the ideal case. The request is accepted.

**Case III: \(R_{Seq} > local\)**

When the request's Sequence is greater than the current local Sequence, Property 4 ensures that consensus instances responsible for Sequence increments cannot proceed in parallel.

According to Lemma C, CC propagation and Sequence synchronisation take at most \(t+e\).

Since at least \(7t\) must elapse before the next Sequence increment, according to Property 4, and generally \(e<6t\), the Sequence can differ by at most \(+1\).

Therefore, based on the cases above, the Sequence of an honest request can differ from the current local Sequence by at most \(\pm 1\).

Consequently, the function \(ValidSeq(r,l)\) produces the same result for every honest node when evaluating an honest request. \(\blacksquare\)

## References

- **Dolev–Strong:** Addresses the problem that the observed complexity of the conventional Dolev–Strong Protocol, frequently used as a fallback mechanism, increases approximately in the order of \(O(n^3)\).
  - DOI: [10.1137/0212045](https://doi.org/10.1137/0212045)

- **DLS Impossibility Theorem:** Addresses the impossibility of consensus under partial synchrony assumptions.
  - DOI: [10.1145/42282.42283](https://doi.org/10.1145/42282.42283)

- **EPaxos:** Provides inspiration for leaderless consensus and discusses the bottleneck introduced by centralised leaders.
  - DOI: [10.1145/2517349.2517350](https://doi.org/10.1145/2517349.2517350)

- **Leader Consensus Comparison:** Provides supporting evidence concerning leader availability bottlenecks.
  - DOI: [10.5281/zenodo.24422667](https://doi.org/10.5281/zenodo.24422667)