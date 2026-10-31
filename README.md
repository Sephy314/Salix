# Salix Consensus - Leaderless Byzantine Consensus with Majority Quorums and Constant Phases

[//]: # (Specification v3 (Excluding Order Preservation))

## 1. Overview

Salix is a Byzantine Consensus Protocol that performs **per-request consensus** (agreement based on a Commit Certificate for a signed request) using synchronous communication and cryptographic digital signatures in an environment where fewer than half of all nodes are Byzantine and Byzantine clients exist.

The total number of nodes is defined as $n=2f+1$, tolerating at most $f$ Byzantine nodes.

The protocol consists of three fixed phases and an always-on Commit procedure that is not bound to any phase.

$$
\boxed{
\text{Propose}
\rightarrow
\text{Prepare}
\rightarrow
\text{Merge}
}
\quad+\quad
\boxed{\text{Commit (Timing-independent, always-on propagation procedure)}}
$$

The main characteristics of Salix are as follows:

- There is no leader (proposer).
- In an $n=2f+1$ environment, a Commit Certificate (CC) is formed with signatures from $f+1$ nodes.
- It separates request propagation (Prepare) from signature collection (Merge).
- Commit is not a phase, but an always-on procedure independent of timing; any node that receives a valid CC unconditionally propagates it once.
- If an honest node receives a request in time, all honest nodes will hold the CC at the end of Merge without cooperation from Byzantine nodes.
- This version **does not guarantee ordering across requests** (Section 12).
- It does not support separate Fail Recovery.

## 2. System Model and Assumptions

### 2.1. Node Configuration

The total number of nodes is as follows:

$$
n=2f+1
$$

- Honest nodes: at least $f+1$. Honest nodes are fault-free and follow the protocol.
- Byzantine nodes: at most $f$.

Byzantine nodes may exhibit the following behaviors:

- Equivocation
- Silent and Partial Silent behavior
- Collusion
- Crash
- Transmission of arbitrary malformed messages

### 2.2. Clients

Clients may also be Byzantine.

Byzantine clients may exhibit the following behaviors:

- Impersonation attempts
- Fraudulent requests
- Replay attacks
- Concurrent Requests
- Generation of arbitrary request identifiers

Client identity and request authentication are verified via cryptographic signatures.

An honest client sends its request to **at least $f+1$ nodes** (to ensure at least one honest node is included).

### 2.3. Cryptographic Assumptions

1. Nodes and clients know each other's public keys and can verify signatures.
2. Digital signatures cannot be forged.
3. The hash function satisfies collision resistance.
4. An honest node does not arbitrarily forge or tamper with signatures it generates.
5. Signatures are domain-separated by including the message type and target (a request signature and a Commit signature cannot be substituted for one another).

### 2.4. Network and Timing

The network is synchronous.

- $\Delta$: Upper bound on message delivery delay between honest nodes
- $B_k$: Upper bound on entry skew among honest nodes at the start of phase $k$
- $\epsilon$: Upper bound on processing time required for signature verification and scheduling

A message sent by an honest node is delivered to other honest nodes within $\Delta$.

### 2.5. Fail Recovery and Indeterminate Outcomes

Salix does not support protocol-level Fail Recovery. Recovery and resumption from node failures occurring during consensus are handled by the implementation.

**"Failure" is not a finalized outcome, but an "indeterminate outcome."** The fact that a node has not received a CC by a certain point in time does not mean that a CC will not exist thereafter (a Byzantine node may withhold a CC and reveal it later).

## 3. Request and Consensus Identification

### 3.1. Request Structure

When generating a consensus request, a client provides the following information:

```text
Request {
    requestId
    clientRequest
    clientSignature
}

clientSignature = Sign(clientSK, "REQ" ‖ requestId ‖ clientRequest)
```

`proposeId` is not a value provided by the client, but a **derived value computed by nodes** (since `proposeId` takes `clientSignature` as input, the signature cannot include `proposeId`).

### 3.2. Propose ID

`proposeId`, the identifier of a consensus instance, is defined as follows:

$$
proposeId =
H(
clientSignature
\parallel
clientRequest
\parallel
requestId
)
$$

The possibility of different request contents having the same `proposeId` is ruled out by hash collision resistance. Therefore, different request contents cannot be bound to a single consensus instance ID.

If the signature scheme is non-deterministic, the same logical request may have different `proposeId` values. In this case, they are treated as separate instances, and duplicates are eliminated in the State Machine (SM) based on the criteria in Section 3.3.

### 3.3. Request ID

`requestId` is an identifier that manages request uniqueness and idempotency.

The main rules are as follows:

- When retransmitting the same logical request, the same `requestId` is used.
- Even if the request content is identical, a new `requestId` may be used if it represents a new operation.
- If the request content differs, it is treated as a distinct request even if the same `requestId` is used.
- `requestId` does not need to be sequential or globally unique.

Examples:

| Request    | Request ID | Handling                             |
|------------|-----------:|--------------------------------------|
| PUT x = 10 |          1 | New operation                        |
| PUT x = 20 |          2 | New operation                        |
| PUT x = 10 |          3 | Distinct new operation               |
| PUT x = 10 |          1 | Retransmission of existing operation |

Even if the same logical request is committed multiple times, it must be applied to the SM only once.

Whether an application is a duplicate is determined based on the client's identity, `requestId`, and request content.

## 4. Commit Certificate

A Commit Certificate (CC) consists of the entire request for a specific consensus instance and the set of Commit signatures from nodes.

```text
CC {
    request: { requestId, clientRequest, clientSignature }
    sigs:    { (nodeId, commitSignature) }
    order:   ⊥        // Reserved field. Currently unused (Section 12)
}

commitSignature = Sign(nodeSK, "COMMIT" ‖ proposeId ‖ H(request) ‖ order)
```

From a single CC, the `proposeId`, request, and signers can all be extracted.

A valid CC must satisfy the following conditions:

1. It includes at least $f+1$ signatures from distinct nodes.
2. All signatures must be valid.
3. All signatures must correspond to the same `proposeId` and the same request data (`H(request)`).
4. `proposeId` matches the value recomputed from `request`.
5. The client signature is valid and the request format is valid.
6. It passes the implementation's **stateless** validity rules.

$$
|CC|\geq f+1
$$

Even if the sets of signatures themselves differ, any set of $f+1$ or more valid signatures targeting the same `proposeId` and the same request data is treated as a CC for the same consensus result.

A CC generated in one consensus instance is valid only for that instance.

## 5. Protocol

### 5.1. Propose

**Purpose: Initiation of consensus.**

The client delivers the request it generated to the nodes.

```text
Propose(
    requestId,
    clientRequest,
    clientSignature
) -> ()
```

When a node receives a request during the Propose interval, it computes `proposeId` and verifies the client signature, format, and stateless rules. Only requests that pass verification are propagated in Prepare.

Propose is the stage that signals the start of consensus; Commit signatures are not collected at this stage.

### 5.2. Prepare

**Purpose: Propagation of requests.**

```text
Prepare(
    requestId,
    clientRequest,
    clientSignature
) -> ()
```

A node propagates **requests that were directly received and passed verification during the Propose interval** to all other nodes.

**Requests received via Prepare are not re-propagated.**

The primary purpose of Prepare is to compensate for situations where a Byzantine node or client delivers a request to only a subset of nodes. Under the synchronous network assumption, if at least one honest node receives a request during the Propose interval and propagates it, all honest nodes will receive that request within the Prepare interval.

Prepare is neither a voting stage nor a commit decision stage.

### 5.3. Merge

**Purpose: Request validity verification and Commit signature collection.**

A node holding the request body requests signatures for that consensus instance from other nodes.

```text
MergeRequest(proposeId) -> (isValid, commitSignature)
```

A node receiving this request verifies the following:

- Hash verification of `proposeId`
- Client signature verification
- Request format verification
- The implementation's **stateless** validity rules

**Stateful verification (such as account balance, whether the request has already been processed, etc.) is not performed in Merge.** This is because SM states may differ across nodes, causing honest nodes to produce different `isValid` results. Stateful verification is performed deterministically during the application stage (Section 7).

A node that does not hold the request body responds with `isValid = 0` and does not provide a signature. If verification passes, it responds with `isValid = 1` along with a Commit signature. A node includes its own signature in its own collection.

Once a node collects $f+1$ valid Commit signatures from distinct nodes, it constructs a CC and invokes `Commit(cc)` (Section 5.4).

Additional implementation-specific rules must be **stateless rules** that produce identical results across all honest nodes for the same input.

### 5.4. Commit

**Purpose: Verification, propagation, and application of the CC. This is an always-on procedure, not a phase.**

```text
Commit(cc) -> ()
```

The input is a single CC, and all necessary information is extracted from the CC. A node that constructs a CC during Merge also invokes `Commit(cc)` on itself to follow the same execution path. There are no timing constraints, and CCs arriving after phase termination are also processed.

```text
on Commit(cc):
    id = cc.proposeId
    if broadcasted[id]: return              // Exit without verification for already-propagated instances
    if !verify(cc): return                  // All validity conditions from Section 4
    atomically:                             // Prevent concurrent reception race conditions
        if broadcasted[id]: return
        broadcasted[id] = true
    broadcast(cc, everyone)                 // Propagation precedes SM execution
    if !applied[id]:
        applied[id] = true
        result = apply(cc)                  // Section 7
    reply(client, id, result)
```

The rules are as follows:

- `broadcasted[id]` is recorded **only after passing verification** (preventing a fake CC from preempting `proposeId`).
- Duplicate detection is based on `proposeId` rather than raw CC bytes (preventing amplification where valid CCs with different signer combinations are repeatedly propagated).
- Propagation is independent of SM application. Even if application is delayed or fails, propagation is still performed.
- Deduplication when different `proposeId` values represent the same logical request is handled by the SM.

## 6. Phase Stability

In the current phase, once each node secures valid responses from $f+1$ distinct nodes (including itself), it starts a wait timer $s_k$.

When the timer expires, the node terminates the current phase and transitions to the next phase.

### 6.1. Phase Transition Voting

- Once a node completes its current phase operations, it includes itself in the phase transition vote count.
- Only valid responses from distinct nodes are counted toward the vote.
- Responses containing invalid signatures are not included in the vote count.
- The timer starts after securing valid responses from $f+1$ nodes (including itself).

What counts as a "valid response" in the Propose and Prepare phases is an open item in Section 11.

### 6.2. Timer Conditions

Depending on how future-phase messages are handled, the following conditions must be satisfied:

| Phase                                     | Condition                    | Notes                                       |
|-------------------------------------------|------------------------------|---------------------------------------------|
| Prepare (buffering future-phase messages) | $s_k\ge\Delta+B_k$           | One-way propagation                         |
| Prepare (retrying after `Try again`)      | $s_k\ge2\Delta+B_k+\epsilon$ | Includes round-trip                         |
| Merge                                     | $s_k\ge2\Delta+B_k+\epsilon$ | Round-trip required due to request-response |

### 6.3. Requests After Phase Termination

Messages arriving for a phase that has already terminated are rejected. This prevents delayed insertion attacks using messages from previous phases.

**Exception: CC (`Commit`) messages are processed regardless of phase.**

### 6.4. Future-Phase Requests

When receiving a request for a future phase that has not yet started, the node either returns the following response or buffers the message:

```text
Try again
```

This response does not imply any consensus claim or vote. The sender treats the request as a retry target and applies backoff and a retry limit.

## 7. Request Application and Client Response

When a node receives a valid CC for the first time, it executes `apply(cc)`.

- **Stateful verification** (balance, whether the request was already processed, business rules) is performed deterministically at this stage. A request that fails verification is treated as a **no-op that does not modify SM state**, and that result is returned to the client.
- Even if multiple CCs are generated for the same logical request, it is applied to the SM only once (based on client identity, `requestId`, and request content).

**Client Success Determination.** The client determines success not by the CC itself, but upon receiving the same `reply` from **$f+1$ distinct nodes**. Since at least one of those nodes is honest, success cannot be established solely from a CC kept hidden by Byzantine nodes.

A client that fails to receive a result retries using the same `requestId`.

## 8. Byzantine Attack Scenarios

### 8.1. Byzantine Clients

#### Replay

Retransmitting the same logical request with the same `requestId` allows identical request identification. Even if multiple CCs are generated, the SM applies the operation only once.

#### Fraudulent Requests

A client can sign a fraudulent request using its own private key. A signature guarantees client authentication and request integrity, not the semantic truth of the request content. Violations of business rules are handled as no-ops during the application stage (Section 7).

#### Equivocation

Binding different request contents to the same `proposeId` is ruled out by hash collision resistance. Requests using different `proposeId` values are treated as separate consensus instances.

#### Concurrent Requests

Simultaneous submissions of the same request are treated as the same logical operation. Identical request contents using different `requestId` values are distinct operations. Different request contents form separate consensus instances, and **ordering across instances is not guaranteed in this version** (Section 12).

#### Spam

If a Byzantine client generates a large volume of validly signed requests, all honest nodes will sign and propagate them. Rate limiting and quotas are policies outside the consensus layer.

### 8.2. Byzantine Nodes

#### Silent Behavior

**Complete Silence in Propose.** If the client fails to deliver the request to any honest node, it falls outside the guarantee scope. An honest client avoids this by transmitting to at least $f+1$ nodes.

**Partial Silence in Propose.** If at least one honest node receives the request during the Propose interval, it is propagated to all honest nodes during Prepare.

**Silence and Partial Silence in Prepare.** Because honest nodes propagate the request, Byzantine silence alone cannot block request delivery among honest nodes.

**Silence and Partial Silence in Merge.** Even if up to $f$ nodes remain silent, there are at least $f+1$ honest nodes; if all honest nodes holding the request sign it, $f+1$ signatures are secured without Byzantine cooperation.

**Silence and Partial Silence in Commit.** Because an honest node immediately broadcasts any valid CC it receives to everyone, selective silence by Byzantine nodes cannot block the propagation of a valid CC.

#### Partial CC via Collusion with a Subset of Honest Nodes

A Byzantine node can send a request to only a subset of honest nodes during the Prepare stage (since items received in Prepare are not re-propagated) and combine signatures from those honest nodes with Byzantine signatures to form a valid CC. Because this CC is a valid CC for a valid request, the protocol does not prevent its creation, but **renders it harmless**:

- If a Byzantine node gives the CC to at least one honest node, that node broadcasts it immediately, so all honest nodes obtain the same CC within $\Delta$ (Theorem T3).
- If it is not given to any honest node, honest nodes do not commit it, and since the client does not receive responses from $f+1$ nodes, the client does not determine it as successful (Section 7).

#### Equivocation

**Propose / Prepare Equivocation.** Attempts to bind different request contents to the same `proposeId` are rejected during hash verification.

**Merge Equivocation.** Even if a Byzantine node delivers conflicting responses, honest nodes only accept responses containing valid signatures, and a Byzantine node's signature cannot substitute for an honest node's signature.

**Commit Equivocation.** A CC requires $f+1$ valid signatures targeting the same `proposeId` and request data. Since there are at most $f$ Byzantine nodes, they cannot forge a valid CC without at least one honest node's signature.

## 9. Target Properties

**Validity.** A valid CC exists only for a request that has passed client signature, format, and stateless rules verification. Byzantine nodes alone cannot create a CC for any request.

**Uniqueness.** Only a single request corresponds to a single `proposeId`.

**Totality.** If any honest node propagates a valid CC, all honest nodes propagate the same CC within $\Delta+\epsilon$.

**Liveness.** If at least one honest node receives a valid request during its Propose interval, all honest nodes hold a CC by the end of Merge.

**Client Soundness.** If a client determines a request as successful (receiving identical responses from $f+1$ distinct nodes), all honest nodes will hold the same CC.

**Idempotent Application.** Even if duplicate CCs are generated for the same logical request, the request is applied to the SM only once.

## 10. Theorems and Proofs

**T1 (Validity).** The request in a valid CC passes client signature, format, and stateless rules verification, and Byzantine nodes cannot construct a CC on their own.

- *Proof.* Conditions 5 and 6 of Section 4 are enforced by `verify`. A CC contains signatures from at least $f+1$ distinct nodes, and since there are at most $f$ Byzantine nodes, it contains at least one honest signature. An honest node signs only requests that pass verification. $\square$

**T2 (Uniqueness).** Only a single request corresponds to a single `proposeId`.

- *Proof.* `proposeId` is the hash of the request content and client signature; by collision resistance, different contents cannot produce the same value. $\square$

**T3 (Totality).** If an honest node $i$ performs `broadcast` at time $t$, all honest nodes perform `broadcast` within $t+\Delta+\epsilon$.

- *Proof.* The CC propagated by $i$ is valid. Each honest node $j$ receives it within $\Delta$. If $j$ has already propagated it, it has already done so; otherwise, it verifies the CC ($\le\epsilon$) and propagates it. Because the procedure in Section 5.4 is phase-independent, phase-boundary attacks do not exist. $\square$

**T4 (Liveness).** If an honest node $h$ receives a valid request $R$ during its Propose interval, all honest nodes hold a CC for $R$ when Merge ends.

- *Proof.* $h$ propagates $R$ at the start of its Prepare phase, $a_h$. The Prepare entry time $a_j$ of any honest node $j$ satisfies $|a_j-a_h|\le B_k$, and the propagation arrives by $a_h+\Delta\le a_j+B_k+\Delta\le a_j+s_k$, which falls within $j$'s Prepare interval (by the condition in Section 6.2). Thus, at the start of Merge, all honest nodes hold $R$. Since the stateless verification result for $R$ is identical across all honest nodes, all honest nodes sign it. Each honest node $i$ sends a Merge request and receives responses from all honest nodes within $2\Delta+B_k$ (Section 6.2), collecting at least $f+1$ signatures (including its own) from honest nodes alone to construct a CC. Cooperation from Byzantine nodes is not required. $\square$

**T5 (Client Soundness).** If a client receives the same `reply` from $f+1$ distinct nodes, all honest nodes propagate the same CC within $\Delta+\epsilon$.

- *Proof.* Among the $f+1$ responders, at least one is honest, and an honest node sends `reply` only after executing `broadcast`. The claim follows from T3. $\square$

Because T1 and T2 do not rely on synchrony, they **hold even in asynchronous settings**. Only T3, T4, and T5 rely on the synchrony assumptions ($\Delta$, $B_k$).

## 11. Limitations and Open Items

**Limitations**

- **No Ordering.** Because the exact times at which honest nodes receive a CC may differ within $\Delta$, the application order of non-commutative operations is not guaranteed. The guarantee of this version extends only to "for each request, all honest nodes hold the same CC."
- **State Retention.** `broadcasted` and `applied` cannot be retained indefinitely. Cleanup criteria (such as checkpointing) and rules for handling late-arriving CCs after cleanup are needed.
- **Bandwidth.** Because a CC embeds the entire request, it incurs a transmission cost of $(\text{request size}) \times O(n^2)$.
- **Synchrony Assumption Violations.** If $\Delta$ or $B_k$ is violated, the time bounds of T3, T4, and T5 are not guaranteed. T1 and T2 remain intact.

**Open Items**

1. Definition of a "valid response" in the Propose and Prepare phases (Section 6.1).
2. Proof of how the protocol guarantees the entry skew bound $B_k$ (currently an assumption).
3. Specification placement for rate-limiting policies.

## 12. Order Preservation Extension (Future Work)

The protocol is planned to be extended by including the CC's `order` field in the signed payload. However, this field alone does not guarantee ordering. Valid CCs for different requests could still be formed for the same position (since $Q=f+1$ does not guarantee an honest node intersection across quorums), and deciding who assigns the position is itself a consensus problem; therefore, extending the protocol requires rules to close this contention.

## 13. Protocol Summary

$$
\boxed{
\text{Propose}
\rightarrow
\text{Prepare}
\rightarrow
\text{Merge}
}
\;+\;
\text{Commit (Always-on)}
$$

| Stage              | Core Role                                                                |
|--------------------|--------------------------------------------------------------------------|
| Propose            | Client request submission                                                |
| Prepare            | Propagation of requests received in Propose (no re-propagation)          |
| Merge              | Stateless verification, Commit signature collection, and CC construction |
| Commit (Always-on) | CC verification, single propagation, SM application, and client response |

**Core Design Principle:** Separate request propagation from signature collection, form a CC with $f+1$ valid node signatures, and propagate any received valid CC once regardless of timing to ensure agreement across honest nodes.
