# Salix, BFT Leaderless Majority Quorum Constant Round Consensus

## What is Salix?
A protocol that enables reaching consensus within a constant number of rounds using only a majority quorum in BFT consensus.

Its key features include: a leaderless consensus architecture designed to solve the centralized write-distribution bottleneck of traditional View-Change mechanisms; the use of digital signatures to reach consensus with a condition of more than $f+1$ honest nodes, which is below the oral-message lower bound of $2f+1$; and a design that always operates in a constant number of rounds without a fallback, making it relatively lightweight in the worst case compared to similar existing studies that rely on Dolev-Strong-based fallbacks with high communication complexity.

## Assumptions
- A synchronous network model is assumed. Consensus completion cannot be guaranteed before GST.
- $n > 2f + 1$ is assumed. The Quorum size is $f + 1$.
- Clients can also be Byzantine.
- Byzantine behaviors include crash, collusion, silence, equivocation, forged requests, and replay attacks.
- Public keys are mutually verifiable, and unforgeable digital signatures are used.
- The arrival order of client requests is not guaranteed.
- Fail recovery is not guaranteed, though it can be added at the implementation layer.
- Hash collisions are assumed not to occur.
- The log is assumed to be insertable rather than strictly append-only.

## Guaranteed Properties
Salix guarantees the following properties:
1. **Agreement:** All honest nodes observe an honest proposal and eventually apply it to their own State Machine.
2. **Liveness:** Even under Byzantine influence, consensus is unaffected and terminates within a finite amount of time.
3. **Validity:** Byzantine actors cannot cause an invalid value to be committed.

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
Note: This is pseudo-code provided for illustrative purposes only.

## Broadcast Mechanisms
Salix uses two communication methods.

### Type 1 Broadcast
```
on Type1Broadcast(data)
    for i in all nodes
        i.send(data)
```
This is a method of sending to all nodes. If the round-trip communication time is $t$, it takes time $t$ to receive a response.
It is a broadcast method in which a single node propagates data to all other nodes except itself. When an honest node performs a Type 1 Broadcast, all nodes become aware of the existence of this broadcast.

### Type 2 Broadcast
```
on Type2Broadcast(data)
    if broadcasted(data)
        return
    
    Type1Broadcast(data)

on sent(data)
    Type2Broadcast(data)
```
This is a broadcast method in which all nodes propagate data to all other nodes. Upon receiving a broadcast, every node performs a Type 1 Broadcast if it has not already broadcast the data. As long as there is at least one honest node, all nodes will hear this broadcast via Type 1 Broadcast. If the round-trip communication time is $t$, this takes a total time of $2t$.

## Consensus Specification

### Fetch
> `Fetch() -> (latestCC)`

This is a stage for synchronizing the Sequence before consensus or a Read operation. Before GST, the Sequence may diverge for a certain period of time due to the mechanisms described below. This stage is intended to mitigate that divergence. When Fetch is invoked, every node returns its most recent CC. Through this, `nextSequence` and similar information can be determined. It uses Type 1 Broadcast and is not considered the start of consensus.

### Propose
> `Propose(request) -> ()`

Propose is the starting point of consensus. A client selects an arbitrary node as the Proposer and provides the data excluding the fields to be wrapped by the Proposer. After verifying the validity of the data, the Proposer inserts the Sequence, constructs a `Request` based on the `ClientRequest`, and performs a Type 1 Broadcast to all nodes.

### Prepare
> `Prepare(request) -> (request)`

If the Proposer is Byzantine, not all honest nodes may hear the Propose message. This stage guards against a Byzantine node performing equivocation or remaining silent. A node that receives a Propose executes Prepare and performs a Type 2 Broadcast to all nodes. By the definition of Type 2 Broadcast, a request that has already been broadcast is not broadcast again.

### Vote
> `Vote(request, signature) -> (request, signature)`

This stage is for forming a signature set. Here, the node performs validation on the request. This validation checks whether the Sequence matches its own State Machine (SM), whether the `ProposeId` is valid, whether the same request has already been received, etc. If valid, the node creates its own signature to contribute to the signature set. The signature set requires signatures from $f+1$ nodes, and matching CCs are treated as the same CC as long as the `Request` matches, regardless of which specific nodes contributed the signatures.

### Commit
> `Commit(request, commitCert) -> (request, commitCert)`

This stage ensures state consistency among honest nodes even if a Byzantine node selectively sends signatures that contribute to a CC. In this stage, a node combines $f+1$ valid signatures acquired in the Vote stage with the `Request` to form a CC, and then propagates this CC via Type 2 Broadcast. A CC corresponding to a given `Request` is sent only once. That is, as stated earlier, even if the constituent signatures differ, CCs with the same `Request` are considered identical; as long as all contributing signatures are valid, a CC with valid signatures is shared only once. Upon securing a valid CC in this stage, every node records it, applies it to its SM, and increments its Sequence by one. Honest nodes that receive this report the result to the Client and terminate consensus.

### Phase Transition
This is executed locally: upon receiving $f+1$ responses supporting the same value, a node transitions to the next phase. For requests corresponding to a previous phase, the node's response does not change and it resends the value it previously responded with. For requests corresponding to a future phase relative to the node's current phase, the node returns a `retry` response. This does not constitute any claim/vote, and an honest node receiving this response handles retransmission according to the implementation's policy.

### Sequence Rules
Sequences are primarily compared by numerical order; however, multiple consensus instances may correspond to the same Sequence in this protocol, which is intentional. According to the protocol assumptions, consensus does not always reflect the order of client requests—specifically, this applies only to requests submitted concurrently before a consensus instance completes. Nevertheless, since consensus must guarantee Agreement, ordering is determined by hashing `ProposeId` and comparing the hash values. A Proposal's Sequence may be lower than the current Sequence. In this case, based on the node's Sequence prior to the start of consensus, if the request's Sequence falls within the current Sequence $\pm 1$ according to the lemma below, it is treated as valid. By the lemma below, an honest proposal's Sequence cannot exceed the current Sequence (beyond this bound). When incrementing the Sequence, if two or more consensus instances share the same Sequence, the Sequence is incremented only once.

### Validation
When a node receives a Proposal, it performs Validation on it. Validation includes verifying the `ProposeId` hash, verifying the Phase, verifying signatures, and executing any additionally specified checks. Because Validation is a no-op action, all honest nodes always obtain the same result. Hash collisions and ID collisions are assumed not to occur.

### ProposeId and RequestId
One `ProposeId` corresponds to only one `Request`. According to the earlier definition of `ProposeId`, it is the hash of the Request, SignatureId, and RequestId, so its correctness is verified during Validation. `RequestId` is a mechanism to allow cases where the same request must be issued twice at the implementation level. Exceptions such as repeatedly sending the same request while changing only the `RequestId`, or DDoS attacks, are not considered Byzantine actions within the consensus protocol; countermeasures against them are left to rate-limiting at the implementation layer.

## Demonstration

### Assumptions
This section specifies the assumptions required to prove the Liveness and Safety of the protocol, along with the properties to be guaranteed. This protocol guarantees Liveness and Safety, whose mathematical properties are as follows:

| | |
|---|---|
| Network | Synchronous |
| Cryptography | Digital Signature |
| Nodes | $2f+2$ (minimal) |
| Quorum | $f+1$ |
| Byzantine Node Actions | Collusion, Equivocation, Silent, Crash |
| Byzantine Client Actions | Collusion, Replay, Malformed Request |

- **Liveness**

  $\forall i \in H, \forall r \in R_{valid} : Propose(r) \Rightarrow \Diamond Decided_i(r) $

- **Safety**

  $\forall i, j \in H, \forall r \in R_{valid} : Decision_i(r) = Decision_j(r)$

### Lemma A. Property of Type 2 Broadcast
> $\forall i, j \in HonestNodes, \forall k \in AllNodes, \forall r \in Request : T2Broadcast_k(r) \land Observed_i(r) \Rightarrow Observed_j(r)$

If any node $k$ performs a Type 2 Broadcast and at least one honest node observes it, then all honest nodes can observe that value.

**Proof:**  
When an honest node verifies a value received via Type 2 Broadcast, by the protocol rules it performs a Type 1 Broadcast. By the definition of Type 1 Broadcast, the value is propagated to and becomes observable by all nodes. $\blacksquare$

Through this, in the Prepare stage, if any node sends Prepare to at least one honest node, all honest nodes become aware of the request's existence; and in the Commit stage, if at least one honest node discovers the CC, all nodes will learn of this CC.

### Lemma B. Proof of the $n > 2f + 1$ Lower Bound Under Current Protocol Rules
> $|N| > |Q| + |F|$

The total number of nodes must always be strictly greater than the sum of the quorum size and the number of faulty nodes.

**Proof:**  
According to Lemma A, if any node propagates a value to at least one honest node via Type 2 Broadcast, it becomes observable by all nodes. Furthermore, to accept a value, a node requires supporting votes from at least $q = f + 1$ nodes excluding itself. If a Byzantine node forwards a request to only a single honest node, by the protocol rules, other honest nodes require assertions from at least $f + 1$ honest nodes (excluding themselves) to accept it. In addition, the number of nodes must be a natural number, and by the protocol rules, only one vote per node is recognized. Assuming the worst case in which Byzantine nodes make assertions opposing the honest nodes—that is, hindering honest nodes as much as possible—we have:

$$
q \leq n - f - 1 \\
q + f + 1 \leq n \\
\therefore n > q + f \iff n > 2f + 1 \quad (\because n, q, f \in \mathbb{N})
$$

Thus, the bound holds. Within the protocol, the Quorum is meaningfully used in Phase Transition, Prepare, Vote, Commit, and the CC Threshold count. The first four can be proven using the inequality above, and the last case holds since $q > f$ is always satisfied. $\blacksquare$

Additionally, due to the property of Type 2 Broadcast, this protocol does not need to consider conventional quorum intersection.

### Lemma C. Upper Bound on Time to Sequence Convergence
> $\forall i, j \in H, \exists \tau \in [0, t + e] : Seq_i(\tau) = Seq_j(\tau)$

All honest nodes converge to the same Sequence within the sum of the synchrony bound $t$ and the hardware error $e$.

**Proof:**  
Let time $0$ be the moment the first honest node discovers and applies a Commit. According to the protocol rules, immediately after verifying and applying the value, this node transmits it to all other nodes, propagating it across the network. Under the synchrony assumption, delivery takes at most $t$. Given an error margin $e$ accounting for SM application, the elapsed time from when the earliest honest node applies the commit to its SM until the latest honest node applies it to its SM is at most $t + e$. $\blacksquare$

### Lemma D. Stability Proof for the Optimistic Sequence Range
> $\forall i, j \in H, \forall r \in R_{valid} : ValidSeq_i(r) = ValidSeq_j(r)$

All honest nodes make the same determination regarding the sequence validity of any honest request.

**Proof:**  
According to the protocol, the sequence component of the `Valid` function and its evaluation can be defined as follows, where $r$ denotes the request's Sequence and $l$ denotes the local node's current Sequence:

$$
ValidSeq(r, l) =
\begin{cases}
\text{true} & l - 1 \leq r \leq l + 1 \\
\text{false} & \text{otherwise}
\end{cases}
$$

This function determines a request $r$ to be valid if its Sequence lies between the Sequence at the time the Proposal was initially proposed and the Sequence observed after $t + e$ from the current time (the time by which the latest CC is received per Lemma C). To verify the Agreement property of this check, we summarize the basic properties and divide the analysis into cases.

- **Basic Properties**
  1. By Lemma C, a CC is always delivered within the synchrony bound $t$ plus error $e$, causing Sequences to converge.
  2. Due to the Fetch stage, the past-bound error of a Request's Sequence is at most $t$.
  3. The observation time difference between the first node to receive request $r$ and the last node to receive it is at most $3t$ ($Propose + Prepare$).
  4. Because a Sequence can only increment again after a newly created request following the previous increment reaches consensus, Sequence-incrementing consensus instances cannot be processed in parallel. Consequently, a Sequence can increment only after at least one full consensus duration of $t + 2t + 2t + 2t = 7t$.

1. **Case I: $R_{Seq} < local$**

   When the local Sequence is greater than the request's Sequence, the node is already aware of the Sequence increment at the current point in time. In the worst case for its reception time, the node may lag behind the first discovering node by at most $3t$ according to Property 3. To account for this, the local state up to $3t$ prior to initial discovery must be considered; however, by Property 4, Sequence-incrementing consensus instances cannot proceed in parallel. Therefore, in this case, the Sequence can differ by at most $-1$.

2. **Case II: $R_{Seq} = local$**

   This is the ideal case, and the request is accepted.

3. **Case III: $R_{Seq} > local$**

   When the request's Sequence is greater than the local Sequence, by Property 4, Sequence-incrementing consensus instances cannot proceed in parallel. By Lemma C, CC propagation and Sequence synchronization take at most $t + e$. Since incrementing the Sequence takes at least $7t$ by Property 4, and generally $e < 6t$, the Sequence can differ by at most $+1$.

Therefore, by the cases above, the Sequence of any honest request lies within the current Sequence $\pm 1$. Accordingly, $ValidSeq(r, l)$ yields the same conclusion across all honest nodes for all honest requests. $\blacksquare$

## References
- **Dolev-Strong:** Highlights the issue that the Dolev-Strong protocol, frequently used as a fallback, scales to nearly $O(n^3)$ in communication complexity.
  - DOI: `10.1137/0212045`
- **DLS Impossibility Theorem:** Establishes the impossibility results under the partial synchrony model.
  - DOI: `10.1145/42282.42283`
- **EPaxos:** Introduces the concept of leaderless consensus and critiques the bottleneck of centralized leaders.
  - DOI: `10.1145/2517349.2517350`
- **Leader Consensus Comparison:** Provides supporting evidence regarding leader availability bottlenecks.
  - DOI: `10.5281/zenodo.24422667`