# Salix Consensus — Owner-Scoped Core

## 1. Overview and Research Objectives

Salix is a Leaderless Byzantine Consensus protocol that uses synchronous communication and digital signatures to provide agreement among all honest nodes and state convergence for client-signed actions, in an environment containing fewer than one-third Byzantine nodes and Byzantine clients.

The system aims to satisfy the following conditions:

- Number of nodes: $n=2f+1$
- Maximum number of Byzantine nodes: $f$
- Commit quorum: $f+1$
- Communication model: Synchronous communication
- Authentication: Digital signatures
- Consensus architecture: Leaderless
- Consensus rounds: A constant number of rounds independent of $f$
- Variable model: Every variable is assigned to a single Owner

### 1.1. Research Objectives

The primary objective of Salix is to construct Leaderless, constant-round consensus using synchronous communication and digital signatures in an $n=2f+1$ environment.

The design aims to address the following problems:

1. **Leader availability:** Eliminate dependence on a single leader and on availability structures that rely on View Change after leader failure.
2. **Round-dependent communication overhead:** Reduce the round and communication costs incurred by BackOff-style protocols requiring $f+1$ rounds.
3. **Centralised ordering dependency:** Avoid architectures in which a single centralised leader must determine the global order.

An additional contribution of Salix is the abstraction of operations handled by consensus into variable definition and modification. The consensus target is not an arbitrary batch of transactions, but a versioned state transition (action) on a variable with an explicitly designated Owner.

### 1.2. Abstraction of Consensus Targets

Every action handled by consensus is reduced to an operation on a single variable.

- **OP operations:** Handle variable definitions.
    - `Declare`: Declare a variable.
    - `Redefine`: Redefine variable metadata.
    - `Retire`: Remove a variable definition.
- **WR operations:** Handle variable values or reads.
    - `Write`: Modify a variable value.
    - `Read`: Reference a specific execution version of another variable.

The core rules are as follows:

1. An action modifies exactly one variable.
2. Every variable is assigned to a single Owner.
3. The version order of a variable is represented by its own independent version sequence.
4. No default global order is imposed across different variables.
5. Dependencies between variables are explicitly specified through Read dependencies.
6. Shared state calculated from multiple variables is treated as derived state produced by a deterministic function, rather than as a variable that must be directly agreed upon.

### 1.3. Owner-Scoped State and Derived Shared State

Salix distinguishes between the existence of shared state and the claim that a shared variable must exist.

A logical state $S$, which is jointly observed or used by multiple parties, can be represented as a deterministic function of independent variables owned by those parties.

$$
S=F(X_1,X_2,\ldots,X_m)
$$

Here, $X_i$ is a variable or set of variables assigned to Owner $i$, and $F$ is a deterministic function defined by the implementation.

For example, the balance of a joint account can be derived from the transaction histories independently owned by Alice and Bob.

$$
Balance=B_0+\sum T_A+\sum T_B
$$

Under this model, `Balance` does not necessarily need to be a separate shared variable. The consensus targets are modification actions on each Owner's transaction history, while the balance is calculated as a function of those states.

This approach provides a way to represent shared state without requiring direct multi-writer permissions for that state. However, the ability to calculate derived state does not automatically guarantee atomic multi-variable updates, global invariants, or concurrency constraints. Such constraints must be addressed separately in the application design.

### 1.4. Core and Blocks

- **Core** guarantees which actions have been agreed upon, at which versions, for which owned variables. Its principal output is a self-verifying Commit Certificate (CC), which proves an action and its order through signatures from $f+1$ nodes.
- **Blocks** provide additional functionality outside Core, including Failure Recovery, Checkpoint, Admission, Application, Result Query, Reaction, and Gateway.

Core provides only a single Fast Path. Epoch consensus, the GLOBAL Owner, and the Slow Path do not exist.

### 1.5. Core Principles

- There is no leader. The Proposer acts as a bridge between clients and nodes and does not determine ordering or acceptance.
- Every variable is assigned to a single Owner.
- Only the variable's Owner may sign and submit actions for that variable.
- Actions and Fast CCs are propagated once after verification.
- Every CC proves both the action and its order.
- Actions contain references to already executed versions, rather than the values themselves.
- Execution is performed by a deterministic function that uses only the action and its pinned input values.
- Shared state can be derived from the results of owned variables, and the derived state itself is not a consensus target of Core.

---

## 2. System Model and Assumptions

### 2.1. Nodes

The number of nodes is $n=2f+1$.

- There are at least $f+1$ honest nodes, which are non-faulty and follow the protocol.
- There are at most $f$ Byzantine nodes.
- Byzantine nodes may perform equivocation, remain silent, selectively remain silent, collude, and send arbitrary messages.

### 2.2. Clients and Owners

Clients may be Byzantine. A Byzantine client may attempt to submit false actions, replay actions, submit actions concurrently, or sign different actions for the same `(V, ver)`.

- A client can only use its own key and cannot impersonate another client.
- A variable's Owner is the client identity that has permission to modify that variable.
- An honest Owner signs only one action for the same `(V, ver)`.
- A Byzantine Owner may sign conflicting actions for the same `(V, ver)`. In this case, progress on the corresponding variable may halt.

### 2.3. Cryptographic Assumptions

1. Nodes and clients know one another's public keys and can verify signatures.
2. Signatures are unforgeable, and the hash function is collision-resistant.
3. Signatures are domain-separated by message type and target. For example: `"ACT"` and `"COMMIT"`.
4. Honest nodes do not forge their own signatures.

### 2.4. Synchrony Model

- $\Delta$: Upper bound on message delivery delay between honest nodes.
- $\epsilon$: Upper bound on verification and scheduling processing time.
- $\Delta'=\Delta+\epsilon$
- $W$: Waiting period before a Fast vote, where $W\geq\Delta'$.

The Fast Path uses only local relative time. Clock skew $\delta$ and epoch start times, which were required for round alignment in the Slow Path, are no longer necessary.

### 2.5. Proposer

The Proposer is the node to which a client sends an action.

1. It receives the action and forwards it to all nodes using `Propose`.
2. If it holds the action's CC, it forwards the CC to the client.

The Proposer cannot modify the action (because it is signed by the client) or forge a CC (because it requires node signatures). A Byzantine Proposer can only remain silent, selectively forward, delay, or withhold a CC. These behaviours affect liveness only, not safety.

After a timeout, a client may retry the same action through another Proposer. Assuming that Proposers are selected uniformly and independently, the probability of reaching an honest Proposer within $k$ attempts is:

$$
1-\left(\frac{b}{2f+1}\right)^k
\geq
1-\left(\frac{f}{2f+1}\right)^k
$$

Here, $b\leq f$ is the actual number of Byzantine nodes.

### 2.6. Failure Recovery

Core does not provide Failure Recovery itself. Recovery is implemented as a block built on top of Core. The fact that a node has not received a CC up to a certain point does not indicate failure; it indicates an unresolved result.

---

## 3. Variable Model and Operations

### 3.1. Variables and State

A variable is identified as follows:

$$
V=(owner,name)
$$

`owner` is a client key or `clientId`. The `GLOBAL` Owner does not exist.

The state is represented as follows:

```text
S : V ↦ (def = meta, val, ver)       // Undefined variables are absent from S
lastVer[V]                          // Last consumed version; persists after Retire
history[V, ver]                     // Execution history maintained by the implementation
```

- `ver` is the number of the version consumed by an action agreed upon for that variable.
- Versions increase monotonically for each variable.
- Versions continue across deletion and redeclaration.
- `(V, ver)` is a unique position in the history of that variable.
- `lastVer[V]` is retained as a tombstone even after deletion.

All variables follow the same update-authorisation rule.

| Owner | Update permission | Ordering |
|---|---|---|
| `clientId` | Only actions signed by that Owner | The version sequence of variable `V` |

### 3.2. Operation Definitions

| Type | Operation | Preconditions | Effect |
|---|---|---|---|
| OP | `Declare(V, meta, init)` | `V` is undefined | Define the variable and set its initial value |
| OP | `Redefine(V, meta')` | `V` is defined | Replace metadata while preserving the value and Owner |
| OP | `Retire(V)` | `V` is defined | Remove the definition while retaining `lastVer[V]` |
| WR | `Write(V, f, args)` | `V` is defined | `val(V)=f(pinned reads,args)` |
| WR | `Read(V', ver')` | The referenced version has already been executed | Use as an action input without changing state |

The Core defines the semantics of OP operations, variable existence, ownership, versions, and preconditions.

The value function `f` and business rules for WR writes are defined by the implementation. However, `apply` may only access the declared reads and the target variable. If this requirement is violated, the entire action is treated as a no-op.

Ownership transfer is not included as a Core operation. If required, it can be represented at the implementation level using `Retire` by the previous Owner and `Declare` by the new Owner. This process may require separate coordination between the version sequences of different Owners.

### 3.3. Action Format

```text
Action {
    author            // Signer (clientId); must equal target.owner
    target            // The single variable V affected
    ver               // Version to consume in V
    reads             // [(V', ver')] references to already executed versions
    effect            // OP.Declare | OP.Redefine | OP.Retire | WR.Write(f, args)
    authorSignature   // Sign(authorSK, "ACT" || author || target || ver || reads || effect)
}

proposeId = H(author || target || ver || reads || effect)
```

Static validation checks the signature, hash, format, and `target.owner == author`.

Since an action has only one `target`, it modifies only one variable.

### 3.4. Read Dependency Contract

An action identifies the variables it reads by referencing `(V', ver')`. The values themselves are not included in the action.

**Core restriction: Read may reference only versions whose execution has already completed.**

$$
Executed(V',ver') \prec Accept(a)
$$

Here, `Accept(a)` denotes the local acceptance point at which the action becomes eligible for Fast voting. The mere existence or commitment of a CC is insufficient. A CC proves the agreed order, but does not automatically mean that the corresponding state version has been applied.

The rules are as follows:

1. Every Read reference in an action must identify a valid variable and version.
2. The referenced version must already have been executed at the node.
3. An action containing a reference that has not yet been executed may be propagated, but it cannot become eligible for voting until the referenced version has been executed.
4. An honest client should submit an action only after confirming that the referenced version has been executed.
5. References to already executed versions do not create backward dependencies in the execution waiting graph.
6. To reference a historical version, its history must be retained.

This restriction is intended to prevent circular waiting and dependency deadlocks between versions that have not yet been executed.

### 3.5. Deterministic Execution Contract

The execution function has the following form:

$$
apply(action,pinned)\rightarrow state\ change\ |\ no\text{-}op
$$

- `pinned` contains the values of versions explicitly referenced by the action that have already been executed.
- The function cannot read any state other than the action and `pinned`.
- State-dependent validation (e.g. checking a balance) is performed deterministically inside `apply`.
- A failed validation results in a no-op. Once the CAS succeeds, the version is consumed regardless of whether the operation succeeds.

**Implementation determinism requirement:** All honest nodes must produce the same result from the same action, the same pinned inputs, and the same previous state.

The implementation must control the following:

- No asynchronous or node-specific Wall-clock dependencies.
- No use of uncoordinated randomness.
- Normalisation of serialisation and hash inputs.
- Control of floating-point arithmetic differences and non-deterministic data-structure iteration.
- External data access must be converted into agreed inputs.
- Identical initial state and application rule versions.
- Restriction or normalisation of operations whose results may differ across runtimes or platforms.

Core does not enforce the specific implementation of business operations, but the implementation must satisfy the determinism contract above.

### 3.6. Derived Shared State

Shared state derived from owned variables belonging to multiple Owners does not need to be declared as a Core variable.

$$
S=F(X_1^{v_1},X_2^{v_2},\ldots,X_m^{v_m})
$$

Here, $X_i^{v_i}$ is a specific execution version of a variable owned by Owner $i$.

Example: Joint account

```text
Alice.transactions = [+100, -30, +50]
Bob.transactions   = [-20, +200, -10]

Balance = initialBalance
        + sum(Alice.transactions)
        + sum(Bob.transactions)
```

`Balance` is calculated as a deterministic function of owned transaction histories rather than as an independent shared variable.

The meaning of this model is:

- Each Owner modifies only its own variables.
- Shared state is composed from the results of multiple owned variables.
- The consensus targets are individual variable state transitions.
- Derived state itself requires neither separate consensus nor an Owner.

#### 3.6.1. Consistency of Derived State

To calculate derived state identically, all honest nodes must use the same input versions.

For example, the following represents one explicit input snapshot:

$$
Balance=F(Alice^{(5)},Bob^{(8)})
$$

If each node uses a different combination of versions, it may calculate a different result. Therefore, queries and calculations of derived state must specify an input version vector.

This model does not automatically guarantee:

- Atomic simultaneous modification of multiple variables.
- Global invariants spanning multiple variables.
- Global linearisability for concurrent operations.
- Automatic freshness of derived state.

These requirements must be addressed through application business rules, partitioning strategies, reservation protocols, or separate coordination mechanisms.

### 3.7. Variable Declaration Rules

Declaration (`Declare`) is an ordinary action.

1. Reads and writes to undefined variables result in a deterministic no-op.
2. If the same `V` is declared multiple times, only the declaration executed first in version order is valid.
3. A Read referencing a version preceding declaration is invalid.
4. Declaration eligibility and naming rules are determined by the implementation's deterministic policy.
5. If a variable is redeclared under the same name after deletion, it continues from the version following the existing `lastVer[V]`.

---

## 4. Identification and Commit Certificate

### 4.1. Lanes and Conflicts

- A **lane** is a variable `V`.
- No default ordering exists between different variables.
- Precedence relationships between variables are explicitly specified through Read dependencies.
- A conflict occurs when different `proposeId` values exist for the same `(V, ver)`.
- The idempotency key is `(V, ver)`.

Version sequence rules:

- `(V, ver)` may be executed only after `(V, ver-1)` has been executed.
- As a default rule, an honest Owner submits the next version only after confirming the agreement and execution of the previous version.
- Pipelining is permitted only if the implementation satisfies additional safety conditions. Under the default specification, execution of the next version is not permitted before the previous version has been executed.

### 4.2. Commit Certificate

```text
CC {
    action
    order      // (V, ver)
    proof      // { (nodeId, sig) }
               // sig = Sign("COMMIT" || proposeId || order)
}
```

Fast CC validity conditions:

1. At least $f+1$ valid signatures from distinct nodes.
2. All signatures refer to the same `proposeId` and `order`.
3. `order == (action.target, action.ver)`.
4. `action.target.owner == action.author`.
5. The action's signature and static format validation are valid.

A CC can independently verify the action, order, and signers, regardless of who provides it.

A CC proves that the action has been agreed upon at the corresponding variable version position. Queries for execution results are handled by the Result Query block.

---

## 5. Consensus

### 5.1. Messages and Propagation

Message types:

- `Propose`: Forwarded from the Proposer to nodes.
- `Prepare`: Relays an action between nodes.
- `Vote`: A node's Commit signature.
- `CC`: Commit Certificate.

Messages are not rejected based on the time interval in which they are received.

```text
on receive(x):                       // Action or Fast CC
    id = proposeId(x)
    if forwarded[id, kind(x)]: return
    if !verifyStatic(x): return
    atomically:
        if forwarded[id, kind(x)]: return
        forwarded[id, kind(x)] = true
    broadcast(x, everyone)
    handle(x)
```

- `forwarded` is recorded only after state-independent validation succeeds.
- The propagation gate does not impose state-dependent conditions such as the current version or execution status.
- A valid action is propagated even if its Read dependencies have not yet been executed.
- Voting eligibility is checked separately.

### 5.2. Voting Rules

Let $t_r$ be the local time at which action $r$ is first observed, and let the waiting period be $W\geq\Delta'$.

The voting time must satisfy both of the following conditions:

1. At least $W$ has elapsed since the first observation.
2. All Read dependencies have already been executed at that node.

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

`seen[key]` records the distinct actions for that key that have passed state-independent validation. Conflict detection is performed again while waiting for Read dependencies to be executed.

### 5.3. CC Construction and Commit

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

Receiving a CC does not trigger immediate execution. Execution is scheduled once the variable version order and Read dependency execution conditions are satisfied.

### 5.4. Client Response

The reply is a CC.

The Proposer forwards the CC to the client once it holds the action's CC. The client independently verifies the CC to confirm that the action was committed at the specified order.

If the Proposer receives an action for which it already holds a CC, it may return the CC without initiating a new consensus process.

Queries for execution results are outside the scope of CC and are handled by the Result Query block.

---

## 6. Execution

### 6.1. Execution Conditions

The execution conditions for `(V, ver)` are:

1. `(V, ver-1)` has been executed. This condition is omitted for the first version of the variable.
2. All Read dependencies `(V', ver')` have been executed.
3. The values of referenced historical versions are available in the history.
4. A valid CC for the action exists.

Once these conditions are satisfied, `apply(action, pinned)` is executed.

### 6.2. Per-Variable Execution

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

Under the default rules, an honest Owner does not create version gaps. If a Byzantine Owner skips versions or submits conflicting actions, progress on that variable may halt.

### 6.3. Meaning of `apply`

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

All honest nodes must use the same execution order and the same pinned inputs. Applications that violate the implementation determinism contract are outside Salix's state convergence guarantees.

---

## 7. Byzantine Attack Scenarios

### 7.1. Byzantine Clients and Owners

| Attack | Result |
|---|---|
| Impersonating another client | Rejected by signature verification |
| Modifying another Owner's variable | Rejected by the `target.owner == author` check |
| Action replay | Treated as an already consumed version and ignored |
| Signing different actions for the same `(V, ver)` | Honest nodes do not vote, halting only the affected variable lane |
| Modifying multiple variables with one action | Prevented by the single-`target` format |
| Creating a version gap | Execution of subsequent versions of that variable is delayed or discarded |
| Referencing an unexecuted version | Voting eligibility is suspended |
| Using an undefined variable | Deterministic no-op |
| Duplicate declaration | Only the first valid declaration in version order is applied |
| Mass variable creation | Restricted by the Admission and Declaration Policy blocks |
| False business semantics (e.g. exceeding a balance) | Results in a no-op through the implementation's deterministic `apply` validation |

### 7.2. Byzantine Nodes and Proposers

| Attack | Result |
|---|---|
| Forwarding an action only to selected honest nodes | Once one honest node receives it, propagation forwards it to all honest nodes |
| Injecting conflicting actions at different times | Waiting with $W\geq\Delta'$ and propagation prevent honest nodes from voting for both |
| Partial CC collusion with selected nodes | Once one honest node receives the CC, CC propagation distributes it to all |
| Proposer silence or CC withholding | Client retries; safety is unaffected |
| Forged CC | Verification fails without $f+1$ valid signatures |
| Inducing signatures for different orders | Prevented by honest nodes' voting rules and conflict checks |

---

## 8. Guaranteed Properties

- **Validity:** The action in a valid CC has passed Owner signature verification and state-independent validation. Byzantine nodes alone cannot create a CC.
- **Uniqueness:** A single `proposeId` corresponds to only one action.
- **Totality:** A valid action or Fast CC observed by an honest node is observed by all honest nodes within $\Delta'$.
- **Fast Order Uniqueness:** Different actions for the same `(V, ver)` cannot both receive honest votes.
- **Order Authenticity:** A valid CC proves one action and one variable version order, and no different action can have a CC for the same order.
- **Single-Variable Effect:** An action affects only one variable definition or value.
- **OP Determinism:** The variable definition table is identical across all honest nodes.
- **Read Safety:** Actions reference only already executed versions and do not create cycles in execution dependencies.
- **Deterministic Execution:** The same action, previous state, and pinned inputs produce the same result at every honest node.
- **Convergence:** All honest nodes converge to the same version sequence and state for each variable.
- **Liveness:** A valid action from an honest Owner is delivered with a CC to all honest nodes, provided that conflict and execution dependency conditions are satisfied.
- **Proposer Safety:** Byzantine behaviour by a Proposer does not enable CC forgery or action modification.
- **Derived-State Agreement:** Given the same input version vector and the same deterministic function, all honest nodes calculate the same derived shared state.

---

## 9. Theorems and Proof Obligations

The following theorems are specification-level objectives. The final paper requires rigorous proofs for each assumption and state transition rule.

### T1. Validity

An action in a valid CC has passed valid Owner signature verification and format validation, and Byzantine nodes alone cannot create a CC.

**Proof sketch.** A CC requires at least $f+1$ signatures from distinct nodes. Since there are at most $f$ Byzantine nodes, at least one honest signature is required. Honest nodes sign only actions that pass validation.

### T2. Uniqueness

By collision resistance, a single `proposeId` corresponds to only one action.

### T3. Totality / Propagation

Let $t$ be the time at which the first honest node receives a valid action or Fast CC. All honest nodes receive it by $t+\Delta'$.

**Proof sketch.** The first honest node to receive the message forwards it to everyone after validation. Propagation is not blocked by state-dependent conditions.

### T4. Fast Order Uniqueness

Two different actions for the same `(V, ver)` cannot both receive honest votes, provided that $W\geq\Delta'$.

**Proof sketch.** Let the first honest observation times of two conflicting actions be $a$ and $b$, respectively. By T3, each action is propagated to all honest nodes within $\Delta'$ of its first observation. An honest node must wait at least $W$ after its first observation before voting. Assuming that both actions receive honest votes leads to the same time inequalities used in the existing Fast uniqueness argument, from which $W<\Delta'$ is derived, yielding a contradiction. Delays caused by Read dependencies can only postpone voting, and conflict detection prevents voting once a conflict has been observed.

### T5. Read Safety

An action referencing an unexecuted version is not voted on, and a dependency graph in which Read references point only to completed executions cannot contain an execution-waiting cycle.

**Proof obligation.** All honest nodes must consistently apply the rule that completion of the referenced version's execution precedes the action's voting eligibility. Since Read references point to already completed execution events, no backward dependency that waits for a later action's execution is formed.

### T6. Fast Liveness

For an action submitted by an honest Owner, if all Read dependencies have already been executed at every honest node before the action first reaches any honest node, and there is no conflict for the same `(V, ver)`, then all honest nodes hold a CC within $a+2\Delta'+W$, where $a$ is the time at which one honest node first receives the action.

**Proof sketch.** All honest nodes receive the action within $\Delta'$ of $a$, then vote within a further $W$. Votes are propagated within $\Delta'$. Since there are at least $f+1$ honest nodes, a CC can be constructed without Byzantine cooperation. The condition that all Read dependencies have already been executed at every honest node is essential. The above time bound does not apply to actions that do not satisfy this condition.

### T7. Convergence

All honest nodes converge to the same state for each variable.

**Proof obligations.**

1. Propagation of valid actions (T3).
2. Uniqueness of actions at the same order (T4).
3. Identical execution order for each version.
4. Completed execution of Read dependencies and references to identical versions (T5).
5. The implementation determinism contract.
6. Identical initial states.

If these conditions are satisfied, each variable's version sequence produces the same deterministic state transitions at all honest nodes.

### T8. CC Soundness — Honest Owner

For an action signed by an honest Owner, if a valid CC exists, the action is executed at the specified order by all honest nodes, provided that the Read dependencies and previous-version execution conditions are satisfied.

### T9. Order Authenticity

Two different actions cannot have valid CCs for the same `(V, ver)` order.

**Proof sketch.** Each CC contains at least one honest signer (among its $f+1$ signatures, at most $f$ can be Byzantine). Assuming that two CCs prove different actions, each action must have received at least one honest vote for that order. This contradicts Fast Order Uniqueness in T4. Quorum intersection alone is insufficient, since with $n=2f+1$ and quorum $f+1$, the intersection of two quorums may consist entirely of Byzantine nodes.

### T10. OP Determinism

The variable definition tables of all honest nodes are identical.

**Proof sketch.** OP operations are ordinary actions and are executed in a single order per variable. Since their preconditions and effects are deterministically defined by Core, the same initial state and the same action sequence produce the same definition table.

### T11. Derived-State Agreement

If all honest nodes use the same input variable version vector and execute the same deterministic function $F$, the derived shared state is identical.

$$
X_i^{v_i}=X_{i,\ honest}^{v_i}
\quad\forall i
\implies
F(X_1^{v_1},\ldots,X_m^{v_m})
\text{ is identical}
$$

This theorem assumes a deterministic derived function and matching input versions. It does not provide a global snapshot automatically.

---

## 10. Core and Blocks

### 10.1. Core Contract

| # | Contract | Basis |
|---|---|---|
| C1 | Self-verifying CC: $f+1$ signatures prove the action and order | T1, T9 |
| C2 | Propagation: Actions and Fast CCs observed by an honest node are observed by all honest nodes within $\Delta'$ | T3 |
| C3 | Ordering: One version sequence per variable, with no default global order across variables | T4, T9 |
| C4 | Deterministic execution: `apply(action,pinned)` complies with the implementation determinism contract | T7 |
| C5 | Progress: Conflict-free actions from an honest Owner with executed Read dependencies obtain a CC within constant time | T6 |
| C6 | Progress cursor: `cursor = {V → lastVer[V]}` | Section 6 |
| C7 | OP uniqueness: Variable definitions and Owners can be reconstructed from CCs | T10 |
| C8 | Single-variable action: An action modifies only one variable | Section 3.3 |

### 10.2. Block Interfaces

```text
Admission.onAction(act)             -> accept | delay
Persistence.onCC(cc), .onExecuted(order)
Application.apply(act, pinned)      -> delta | no-op
Recovery.fetch(cursor, peer)         -> [CC]
Checkpoint.propose(cursor, digest), .onStable(cp)
ResultQuery.get(V, ver)              -> value
DerivedQuery.get(F, versionVector)   -> derived value
Reaction.watch(V_src, rule)          -> submit subsequent action
Gateway.submit(act), .retry()         -> CC
Audit.record(cc, execution)
```

### 10.3. Block Constraints

| # | Constraint |
|---|---|
| B1 | Blocks must not block or delay the propagation of validated actions and Fast CCs. |
| B2 | Any policy that rejects a vote must be a deterministic rule applied identically by all honest nodes. Node-specific local rate limits may only be implemented through delay queues. |
| B3 | Blocks must not change a CC's order or invoke `apply` outside Core's ordering rules. |
| B4 | State data obtained from peers or storage must pass CC and execution history verification. |
| B5 | Any block that affects state must be deterministic. |
| B6 | Reaction must submit subsequent actions as ordinary actions and must not bypass Owner signatures or the single-variable rule. |
| B7 | Derived-state calculations must specify an input version vector and must not treat results from different version vectors as the same snapshot. |

### 10.4. Block Catalogue

| Block | Role | Required Core Guarantees |
|---|---|---|
| Failure Recovery | Compare cursors and peer states, retrieve missing CCs, verify, and re-execute them | C1, C3, C6, C7 |
| Checkpoint & GC | Checkpoint cursors, definition tables, tombstones, and state digests; clean up historical data | C1, C4, C6, C7 |
| Application | Implement business rules and deterministic `apply` | C4 |
| Admission / Rate Limit | Identity-based rate limiting and registration policy | C2, C5 |
| Declaration Policy | Variable naming, declaration eligibility, quotas, and fees | C4, C7 |
| Identity / Key Rotation | Key replacement and Owner identity policy | C3, C4 |
| Result Query | Query the value of an executed variable version | C1, C4 |
| Derived Query | Calculate derived state using an explicitly specified version vector | C4, B7 |
| Reaction | Observe the execution of a source variable and submit subsequent actions | C1, C3, C8, B6 |
| Gateway | Proposer selection, timeouts, and retries | T7 |
| Audit | Retain CCs and execution history | C1 |

### 10.5. Composition Rules

- Each block must specify the Core guarantees it requires, the interfaces it provides, and the constraints with which it must comply.
- Dependencies between blocks must be established only through Core contracts.
- Removing or replacing a block must not affect Core safety. However, it may affect liveness, availability, and the freshness of derived state.
- The retention period for CCs and history required for Recovery is determined by the Checkpoint block.

---

## 11. Limitations and Open Questions

### 11.1. Limitations

- Fast Path uniqueness depends on the synchrony assumption $W\geq\Delta'$.
- There is no default global ordering between variables. Snapshots that read the state of different variables cannot be guaranteed to be identical without an explicit version vector.
- Derived shared state is identical only when the same input version vector and deterministic function are used.
- Core does not guarantee automatic freshness of derived state.
- If an Owner equivocates for the same `(V, ver)` or creates a version gap, the corresponding variable lane may halt permanently.
- If an Owner goes offline, submission of new actions for that variable stops. A Leaderless architecture does not eliminate the availability dependency on the Owner itself.
- A model with explicitly assigned variable ownership does not automatically provide atomicity or global invariants for multi-party operations.
- Deterministic `apply` depends on the implementation complying with the contract. Implementation non-determinism may break state convergence.
- Atomic updates spanning multiple variables are not a default Core feature.
- Dynamic targets, meaning target variables calculated during execution, are not supported.

### 11.2. Open Questions

1. The size of the per-variable pending window $w$ and the discard rules for version gaps.
2. History retention periods and tombstone retention policies.
3. Checkpoint specification: interval, state digest, and deterministic handling of CCs arriving after cleanup.
4. Retention and proof policies when Read references access old history.
5. Formal specifications and compatibility verification for block interfaces.
6. Deterministic rules in Admission and Declaration Policy to prevent spam and mass variable creation.
7. Ownership transfer procedures in cases of Owner failure or key loss.
8. Construction and query consistency of version vectors for derived shared state.
9. Design patterns for applications requiring atomic invariants across multiple variables.
10. Comparison with prior work and communication complexity analysis.

---

## 12. Safety Verification and TLA+ Plan

Subsequent verification will combine mathematical proofs with model checking. TLA+ model checking is a means of exploring state transitions and invariant violations within a finite state space; it does not replace mathematical proofs for general infinite executions or all implementations.

### 12.1. Safety Invariants to Verify First

- **TypeOK:** All state variables and messages satisfy their defined types.
- **AtMostOneActionPerOrder:** Honest nodes do not commit conflicting actions for the same `(V, ver)`.
- **ValidCommitCertificate:** A CC contains $f+1$ valid signatures from distinct nodes.
- **OwnerAuthorization:** The action author matches the Owner of the target variable.
- **MonotonicVersion:** `lastVer[V]` never decreases.
- **SingleVariableMutation:** A single action modifies only one variable.
- **ReadExecutedBeforeVote:** An action is not voted on before all Read dependencies have been executed.
- **ReadVersionStable:** References to executed versions use the fixed value of that version.
- **DeterministicApply:** The same state, action, and pinned input produce the same result.
- **StateConvergence:** All honest nodes have the same state for the same variable execution history.
- **DerivedStateAgreement:** Derived state calculated using the same version vector and function is identical.

### 12.2. Liveness Properties to Verify

- A conflict-free valid action from an honest Owner obtains a CC after all Read dependencies have been executed.
- Valid actions and Fast CCs are propagated to all honest nodes.
- A conflict in one variable does not prevent progress in other variables.
- Buffered actions become executable once their preceding versions and Read dependencies have been executed.

### 12.3. Attacks to Model

1. Owner equivocation for the same `(V, ver)`.
2. Selective propagation and silence by Byzantine nodes.
3. Conflicting actions arriving at different times.
4. Invalid Read versions and references to unexecuted Reads.
5. Version gaps and duplicate actions.
6. Late CCs after restart and Checkpoint.
7. Submission interruption by a Byzantine Owner.
8. Use of different version vectors when calculating derived state across multiple variables.

---

## 13. Protocol Summary

| Component | Role |
|---|---|
| Node model | $n=2f+1$, at most $f$ Byzantine nodes |
| Owner | Sole authority permitted to modify each variable |
| Action | OP or WR operation on a single owned variable |
| Order | Per-variable version position `(V, ver)` |
| Propagation | Forward each valid action and Fast CC to everyone once upon receipt |
| Fast voting | At least $W\geq\Delta'$ after first observation, provided there is no conflict and Read dependencies have been executed |
| CC | Self-verifying commit proof `{action, order, proof}` |
| Execution | Deterministically applied after the previous version and all Read dependencies have been executed |
| Shared state | Deterministic function of multiple owned variables and an explicit version vector |
| Consensus path | Single Fast Path |
| Blocks | Recovery, Checkpoint, Application, Admission, Declaration Policy, Identity, Result Query, Derived Query, Reaction, Gateway, Audit |

## 14. Core Definition

Salix provides Leaderless Byzantine Consensus for versioned state transitions of owned variables. Shared state can be represented as a deterministic function of multiple owned variables, and Core does not require shared state itself to be a separate consensus target.

$$
\boxed{
\text{Consensus on owned state transitions}
\quad\Longrightarrow\quad
\text{Deterministically derived shared state}
}
$$

This proposition applies within the scope in which identical input versions, deterministic functions, and application invariants are satisfied.