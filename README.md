# Salix Consensus - Leaderless Byzantine Consensus with Majority Quorums and Worst-Case Constant Rounds

## 1. Overview

Salix is a Byzantine Consensus Protocol that achieves consensus in environments with a minority of Byzantine nodes and Byzantine clients, using synchronous communication and cryptographic digital signatures.

The total number of nodes is defined as $n=2f+1$, tolerating up to $f$ Byzantine nodes.

The protocol consists of the following four phases:

$$
\boxed{
\text{Propose}
\rightarrow
\text{Prepare}
\rightarrow
\text{Merge}
\rightarrow
\text{Commit}
}
$$

The key features of New MBQ (Salix) are as follows:

- Uses signatures from $f+1$ nodes in an $n=2f+1$ environment.
- Separates request propagation and commit signature collection into distinct phases.
- Collects Commit signatures for valid requests during the Merge phase.
- Verifies and applies a valid Commit Certificate during the Commit phase.
- Uses a fixed sequence of 4 phases.
- Does not support built-in Fail Recovery.

## 2. System Model and Assumptions

### 2.1. Node Configuration

The total number of nodes is defined as:

$$
n=2f+1
$$

- Honest nodes: at least $f+1$
- Byzantine nodes: at most $f$

Byzantine nodes may exhibit the following behaviors:

- Equivocation
- Silent and Partial Silent behavior
- Collusion
- Crash failures
- Transmission of arbitrary malformed or invalid messages

### 2.2. Clients

Clients may also be Byzantine.

Byzantine clients may exhibit the following behaviors:

- Impersonation attempts
- False (invalid) requests
- Replay attacks
- Concurrent requests
- Generation of arbitrary request identifiers

Client identity and request authentication are verified through cryptographic signatures.

### 2.3. Cryptographic Assumptions

1. Nodes and clients know each other's public keys and can verify signatures.
2. Digital signatures are unforgeable.
3. The hash function is collision-resistant.
4. Honest nodes do not arbitrarily forge or tamper with signatures they generate.
5. All nodes already possess knowledge of the client's request signature and can verify it.

Based on the assumption that all nodes already know the client signature, it is not separately included in the Propose message.

### 2.4. Network

The network is synchronous.

Let $\Delta$ denote the upper bound on message transmission delay between honest nodes.

Any message sent by an honest node is delivered to other honest nodes within a bounded time.

### 2.5. Fail Recovery

New MBQ does not support protocol-level Fail Recovery.

Recovery and resumption from node failures occurring during consensus are outside the protocol's guarantees.

This can be implemented at the implementation layer.

[//]: # (### Log Ordering)

## 3. Request and Consensus Identification

### 3.1. Request Structure

When generating a consensus request, a client provides the following information:

```text
Request {
    proposeId
    requestId
    clientRequest
    clientSignature
}
```

It is assumed that `clientSignature` is already shared so that all nodes can verify it. Therefore, it is omitted from the actual Propose message.

### 3.2. Propose ID

The identifier for a consensus instance, `proposeId`, is defined as follows:

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

Identical request information produces the exact same `proposeId`.

The possibility of different request contents mapping to the same `proposeId` is ruled out by hash collision resistance.

Consequently, different request payloads cannot be bound to a single consensus instance ID.

### 3.3. Request ID

`requestId` is an identifier used to manage request uniqueness and idempotency.

The main rules are as follows:

- When retransmitting the same logical request, the same `requestId` is used.
- Even if the request content is identical, a new `requestId` may be used if it represents a new operation.
- If the request contents differ, they are treated as distinct requests even if they share the same `requestId`.
- `requestId` does not need to be sequential or globally unique.

Example:

| Request | Request ID | Handling |
|---|---:|---|
| PUT x = 10 | 1 | New operation |
| PUT x = 20 | 2 | New operation |
| PUT x = 10 | 3 | Distinct new operation |
| PUT x = 10 | 1 | Retransmission of existing operation |

Even if the same logical request is committed multiple times, it must be applied to the State Machine (SM) only once.

Duplicate application is determined based on the client's identity, `requestId`, and request content.

## 4. Commit Certificate

A Commit Certificate (CC) is a set of node signatures for a specific consensus instance.

A valid CC must satisfy the following conditions:

1. It contains at least $f+1$ signatures from distinct nodes.
2. All signatures must be valid.
3. All signatures must correspond to the same `proposeId`.
4. All signatures must correspond to the same detailed request data.

$$
|CC|\geq f+1
$$

Even if the signature sets themselves differ, any set of $f+1$ or more valid signatures targeting the same `proposeId` and the same detailed data is treated as a CC for the same consensus outcome.

A CC generated in one consensus instance is valid only for that instance.

## 5. Protocol Phases

### 5.1. Propose

**Objective: Initiation of consensus.**

The client sends its generated request to the nodes.

```text
Propose(
    proposeId,
    requestId,
    clientRequest
) -> ()
```

A node that receives a request from the client initiates propagation of that request.

Propose is the stage that signals the start of consensus; Commit signatures are not collected during this phase itself.

### 5.2. Prepare

**Objective: Request propagation and state exchange.**

Upon receiving a Propose message, a node forwards the request to other nodes.

```text
Prepare(
    proposeId,
    requestId,
    clientRequest
) -> ()
```

The primary purpose of Prepare is to mitigate scenarios where a Byzantine node sends Propose only to a subset of nodes or remains silent toward specific nodes.

Honest nodes propagate received Propose requests to other nodes. Information received via Prepare requests is not re-propagated.

Under the synchronous network assumption, if at least one honest node receives the request and propagates it, all honest nodes are guaranteed to receive the request.

Prepare is not a voting or commit decision phase.

### 5.3. Merge

**Objective: Request validation and Commit signature collection.**

Regardless of its prior local state, each node queries other nodes for the validity and signature of the corresponding consensus instance.

```text
Merge(
    proposeId
) -> (
    isValid,
    commitSignature
)
```

A node receiving a Merge request verifies the following:

- Hash verification of `proposeId`
- Client signature verification
- Request format and implementation-specific validation rules
- Implementation-required conditions, such as whether the request has already been processed

If the request passes validation:

```text
isValid = 1
```

If the request is valid, the node generates and returns a Commit signature bound to both `proposeId` and the request data.

No Commit signature is provided for invalid requests.

Implementation-specific validation rules must be deterministic—producing the exact same result across all honest nodes for identical validation inputs.

When Merge concludes, each node constructs a CC based on the signatures it has collected.

### 5.4. Commit

**Objective: Certificate verification and final application.**

```text
Commit(
    proposeId,
    isCommitted,
    commitCert
) -> ()
```

A node treats consensus as successful if the following conditions are met:

1. `isCommitted = 1`
2. The Commit Certificate is valid.
3. The CC contains at least $f+1$ signatures from distinct nodes for the same `proposeId` and detailed request data.

An honest node that receives a valid CC accepts the CC and applies the request regardless of its prior claims or previous state.

Only a valid CC received during the Commit phase is recognized as a final commit certificate.

If a valid CC is not obtained, the consensus instance is considered failed.

During the Commit phase, nodes propagate the CC to other nodes so that all honest nodes can observe the same certificate.

## 6. Phase Stability

In the current phase, once a node obtains valid responses from $f+1$ distinct nodes (including itself), it starts a wait timer $s_k$.

When the timer expires, the node terminates the current phase and transitions to the next phase.

### 6.1. Phase Transition Voting

- Once a node completes its own computation for the current phase, it includes itself in the phase transition vote count.
- Only valid responses from distinct nodes are counted as votes.
- Responses containing invalid signatures are excluded from the vote count.
- The timer starts after securing valid responses from $f+1$ nodes (including the node itself).

### 6.2. Timer Condition

$s_k$ must satisfy the following condition:

$$
s_k\geq \Delta+B_k
$$

- $\Delta$: Upper bound on message transmission delay
- $B_k$: Upper bound on entry time skew among honest nodes at the start of phase $k$
- $s_k$: Waiting duration for phase $k$

### 6.3. Requests Arriving After Phase Termination

Messages arriving for a phase that has already ended are rejected.

This prevents delayed injection attacks using messages from earlier phases.

### 6.4. Future Phase Requests

If a node receives a request for a future phase that it has not yet started, it returns the following response:

```text
Try again
```

This response does not represent any consensus claim or vote.

The sender treats the request as eligible for retry.

Retries are subject to backoff and a maximum retry limit.

## 7. Log Application and Consensus Termination

A consensus instance that secures a valid Commit Certificate terminates successfully.

The same logical request must be applied to the State Machine (SM) only once.

Even if multiple CCs are generated for the same logical request, duplicate application of the same operation is not permitted.

If consensus succeeds, the final CC is returned to the client.

If consensus fails, a failure result is returned.

## 8. Byzantine Attack Scenarios

### 8.1. Byzantine Clients

#### Replay

Retransmitting the same logical request with the same `requestId` allows nodes to identify it as the identical request.

Even if multiple CCs are generated, the SM applies the operation only once.

#### False Requests

A client can sign a false request using its own private key.

However, during validation in the Merge phase, nodes can inspect implementation-specific business rules and request validity conditions.

A signature guarantees client authentication and request integrity, not the truthfulness of the request payload itself.

#### Equivocation

Binding different request contents to the same `proposeId` is prevented by hash collision resistance.

Requests using different `proposeId` values are treated as separate consensus instances.

#### Concurrent Requests

Simultaneous submissions of the exact same request can be treated as the same logical operation.

Identical request contents with different `requestId` values are treated as distinct operations.

Different request contents constitute separate consensus instances.

### 8.2. Byzantine Nodes

#### Silent Behavior

**Total Silence in Propose**

Cases where a client fails to deliver the request to any honest node prior to the start of consensus fall outside the protocol's guarantees.

**Partial Silence in Propose**

If at least one honest node receives the request, it can propagate the request to all other honest nodes during the Prepare phase.

**Silence and Partial Silence in Prepare**

Because honest nodes propagate requests, silence by Byzantine nodes alone cannot block request delivery among honest nodes.

**Silence and Partial Silence in Merge**

Even if up to $f$ nodes remain silent, at least $f+1$ honest nodes exist.

Therefore, as long as honest nodes respond to a valid request, $f+1$ signatures can be secured.

**Silence and Partial Silence in Commit**

When an honest node obtains a valid CC, it propagates the CC to other nodes.

This ensures that selective silence by Byzantine nodes cannot block the propagation of a valid CC.

#### Equivocation

**Propose Equivocation**

Attempts to bind different request contents to the same `proposeId` are rejected during hash verification.

**Prepare Equivocation**

Attempts to forward conflicting request contents under the same consensus ID are rejected during hash verification.

**Merge Equivocation**

Even if a Byzantine node sends conflicting claims or responses, honest nodes only accept responses containing valid signatures.

A Byzantine node's signature cannot substitute for an honest node's signature.

**Commit Equivocation**

A Commit Certificate requires $f+1$ valid signatures targeting the same `proposeId` and detailed request data.

A Byzantine node cannot forge a valid CC without signatures from other nodes.

## 9. Target Safety and Liveness Properties

New MBQ aims to guarantee the following properties:

### Agreement

Honest nodes do not commit conflicting requests for the same consensus instance.

### Validity

Honest nodes do not provide Commit signatures for invalid requests.

### Termination

When consensus initiation conditions and synchronous communication assumptions hold and a valid request is delivered, honest nodes terminate consensus within a fixed 4 phases.

### Certificate Uniqueness

For a single consensus instance, there must be only one final consensus outcome stored in the log and applied.

### Idempotent Application

Even if duplicate Commit Certificates are generated for the same logical request, the operation is applied to the SM only once.

## 10. Protocol Summary

$$
\boxed{
\text{Propose}
\rightarrow
\text{Prepare}
\rightarrow
\text{Merge}
\rightarrow
\text{Commit}
}
$$

| Phase | Core Role |
|---|---|
| Propose | Client request submission and consensus initiation |
| Prepare | Request propagation among honest nodes |
| Merge | Validity verification and Commit signature collection |
| Commit | CC verification, propagation, and log application |

**Core Design Principle:** Separate request propagation from signature collection, construct a Commit Certificate via $f+1$ valid node signatures, and perform final state application exclusively in the Commit phase.
