# sRFC 39 Meta Intents Extension

This document extends [sRFC 39: Solana Clear Sign](https://github.com/solana-foundation/SRFCs/discussions/4) with cross-instruction value-flow tracing and transaction-level intent reduction.

---

## Motivation

sRFC 39 specifies per-instruction display: field formatting, intent strings, and skip rules. It does not address how a transaction is rendered as a whole.

A Solana transaction is a flat list of instructions. Most are scaffolding around the user's real intent, such as wrapping SOL or closing a token account. sRFC 39's only multi-instruction strategy is to concatenate per-instruction intents with `"and"`, which produces strings like:

> "Creates WSOL Token Account for toly.sol and Transfer 0.5 SOL and Wrap SOL and Swap 0.5 WSOL for at least 100 USDC and Close WSOL Token Account"

The user's actual intent is:

> "Swap 0.5 SOL for 100 USDC"

This extension defines how a wallet can recover that intent from the raw instruction list by tracing value flow across instructions and reducing the result to a minimal user-facing form.

### Summary of additions

| Capability                         | Mechanism                                                                            |
| ---------------------------------- | ------------------------------------------------------------------------------------ |
| Cross-instruction value flow       | `valueFlowNode` on `instructionDisplayNode`                                          |
| Multi-instruction merge            | Two-rule directional merge with transformer-aware survivor selection                 |
| Ephemeral account resolution       | `mintAssociationNode`, `ownerAssociationNode`                                        |
| Transaction-scoped hide conditions | `hideConditionsNode` (`createdInTransaction`, `isSigner`, `accountUsedElsewhere`, …) |

### Design principles

Every node introduced here is optional. An IDL that omits them remains a valid sRFC 39 document and renders with the base behavior. Wallets that do not implement the merge stage fall back to per-instruction interpolation; wallets without interpolation fall back to field lists.

The merge engine has no notion of instruction categories. Merge direction and behavior fall out of each instruction's declared ports.

Merge and hide are display optimizations and do not affect consensus. The transaction signed on-chain is always the full instruction list. A wallet implementing this extension MUST expose an opt-in raw view (see [Advanced-User Display](#advanced-user-display)).

---

## Overview

Each transaction runs through a four-stage pipeline: Annotate → Merge → Hide → Render. Every instruction's IDL `valueFlowNode` resolves to the uniform interface `inputs[(account, amount, token)]` / `outputs[(account, amount, token)]`. The merge engine chains, collapses, and filters those triples into a minimal set of user-facing intents.

### Pipeline walkthrough

The example below traces a typical Jupiter swap, which reduces 7 raw instructions to 1 displayed intent.

```
  7 raw instructions                                       1 displayed intent
  ┌────────────────────────────────┐                      ┌───────────────────────────────────────┐
  │  ComputeBudget:SetLimit        │                      │                                       │
  │  ComputeBudget:SetPrice        │                      │                                       │
  │  AToken:CreateIdempotent       │                      │                                       │
  │  System:Transfer               │       ═══════▶       │  "Swap 0.5 SOL for at least 100 USDC" │
  │  Token:SyncNative              │                      │                                       │
  │  Jupiter:Route                 │                      │                                       │
  │  Token:CloseAccount            │                      │                                       │
  └────────────────────────────────┘                      └───────────────────────────────────────┘
```

#### 1. Annotate

The IDL declares each instruction's consumed and produced value as `(account, amount, token)` triples. `balance` is a symbolic drain or fill operating on whatever the account currently holds. An order-independent pre-pass also seeds the mint and owner association maps from [`mintAssociationNode`](#mintassociationnode) and [`ownerAssociationNode`](#ownerassociationnode) declarations so ephemeral token accounts resolve to a mint and owner.

```
  Instruction                  Inputs                          Outputs
  ───────────────────────────────────────────────────────────────────────────────
  ComputeBudget × 2            —                               —
  AToken:CreateIdempotent      —                               (wsol_ata, 0,       null)
  System:Transfer              (signer,   0.5,     SOL)        (wsol_ata, 0.5,     SOL)
  Token:SyncNative             (wsol_ata, balance, SOL)        (wsol_ata, balance, WSOL)
  Jupiter:Route                (wsol_ata, 0.5,     WSOL)       (usdc_ata, 100,     USDC)
  Token:CloseAccount           (wsol_ata, balance, WSOL)       (signer,   balance, WSOL)
```

#### 2. Merge

The engine walks left to right, fusing items whose outputs match the next item's inputs through a shared account.

```
  ComputeBudget × 2     no value flow

  CreateIdempotent      stays separate

      ┌─────────── shared wsol_ata ───────────┐
      │                                       │
  Transfer  ──▶  SyncNative  ──▶  Route
                                    │
                                    ▼
                  ┌────────────────────────────────┐
                  │  ONE merged survivor           │
                  │    in : (signer,   0.5, SOL)   │
                  │    out: (usdc_ata, 100, USDC)  │
                  │    from: Jupiter:Route         │
                  └────────────────────────────────┘

  CloseAccount          stays separate
```

Transfer feeds SOL into `wsol_ata`, SyncNative converts it to WSOL, Route swaps WSOL for USDC; the three collapse into one merged Swap. `CreateIdempotent` and `CloseAccount` cannot fold in and fall through to step 3.

#### 3. Hide

Surviving lifecycle instructions are evaluated against `hideConditionsNode` rules. All conditions in a rule set must hold for the instruction to be hidden.

```
  AToken:CreateIdempotent
    ├─ accountUsedElsewhere(wsol_ata)
    └─ isSigner(owner)                   ──▶ HIDDEN

  Token:CloseAccount
    ├─ accountUsedElsewhere(wsol_ata)
    └─ isSigner(destination)             ──▶ HIDDEN
```

`wsol_ata` is touched by every instruction in the wrap–swap–unwrap chain, so `accountUsedElsewhere` holds, and the signer-owned destination and owner satisfy `isSigner`.

#### 4. Render

sRFC 39 formatters scale raw amounts and the survivor's `interpolatedIntent` produces the final string.

```
  500_000_000 lamports   ──▶  "0.5 SOL"    (amountFormatterNode, 9 decimals)
  100_000_000 USDC atoms ──▶  "100 USDC"   (amountFormatterNode, 6 decimals)

  ═════════════════════════════════════════════════════
   Final:  "Swap 0.5 SOL for at least 100 USDC"
  ═════════════════════════════════════════════════════
```

The next section formalizes the merge rules. [Worked Example 1](#example-1-jupiter-swap-with-wsol-wrap-and-close) re-traces this same swap once those rules are in scope; later examples cover native staking, multi-hop routing, and flash-fill orders.

---

## Merge Algorithm

The merge stage runs in three sub-steps:

1. [Pre-merge cleanup](#pre-merge-cleanup): drop recognized items with empty ports and no intent.
2. Safe-account-set construction: scan all items for [`accountResetNode`](#accountresetnode) declarations and build the set eligible for the [symbolic junction guard](#symbolic-junction-guard).
3. Left-to-right merge scan: run the [rule table](#rule-table) on the remaining items.

The rest of this section formalizes the scan.

### Concepts

#### Glossary

| Term | Meaning |
| --- | --- |
| Item | An instruction with resolved ports: `inputs[(addr, amount, token)]`, `outputs[(addr, amount, token)]`. |
| Numeric amount | A concrete integer decoded from instruction data. |
| Symbolic amount | A `balanceAmountNode` (drain or fill of the full account balance). Not numerically comparable. |
| Value transformation | Any item where (a) some port `k` has `inputs[k].token ≠ outputs[k].token` (both non-null, non-unknown), or (b) `len(inputs) ≠ len(outputs)` (asymmetric ports: sink or source), or (c) some port `k` has `inputs[k].valueKind ≠ outputs[k].valueKind` (both non-empty). |
| Pass-through transformer | Symmetric transformer (`len(inputs) == len(outputs)`) where every port amount is symbolic. Converts the *form* of value (e.g. lamports to WSOL) without declaring any concrete amount. |
| Account match | `len(A.outputs) == len(B.inputs)`, `A.outputs[k].account == B.inputs[k].account` for every `k`, and all port pairs have compatible `valueKind`. The kinds `native`, `splToken`, and `confidential` are mutually compatible: all match by address. |
| Survivor (S) / Consumed (C) | Roles assigned to A and B before a rule fires. S persists with updated ports; C is absorbed. |
| Consumed symmetry invariant | C must satisfy `len(C.inputs) == len(C.outputs)`. Applies to both rules; failure blocks the merge. |

#### How merging works

Items are processed left to right. For each item A with outputs, the engine scans forward for the first item B whose inputs match positionally, as defined under *Account match* above. When a match is found, the engine assigns the S/C roles and fires a rule. The rule consumes C and propagates `C.farside` into `S.junction`.

```
   item A (executes first)           item B (executes after)
   ┌─────────────────────┐           ┌─────────────────────┐
   │ A.inputs            │           │ B.outputs           │
   │   ("far side")      │           │      ("far side")   │
   │                     │           │                     │
   │ A.outputs ──────────┼── shared ─┼─ B.inputs           │
   │   ("junction side") │  account  │   ("junction side") │
   └─────────────────────┘           └─────────────────────┘
```

The *junction* and *far side* are defined relative to C:

- If `C = B`, then `C.junction = C.inputs`, `C.farside = C.outputs`, and `S.junction = S.outputs`.
- If `C = A`, then `C.junction = C.outputs`, `C.farside = C.inputs`, and `S.junction = S.inputs`.

Every port pair `(A.outputs[k], B.inputs[k])` is classified individually, and all pairs must agree on the same rule. If any pair disagrees, the merge is blocked.

An item with an unresolved port amount (for example an undecrypted `confidentialAmountNode`) is not mergeable: it cannot consume or be consumed. It still participates in forward-scan early termination if it references one of A's output accounts.

### Rules

#### Rule table

|                          | Rule 1: Numeric chain                                       | Rule 2: Symbolic junction                                                                                            |
| ------------------------ | ----------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| Fires when               | Both junction amounts numeric and equal                     | Junction tokens compatible; at least one junction amount is symbolic                                                 |
| Default C (no override)  | `A`                                                         | Item with more symbolic junction-side ports; tie defaults to `A`                                                     |
| Account propagation      | `S.junction[k].account ← C.farside[k].account`              | `S.junction[k].account ← C.farside[k].account`                                                                       |
| Amount propagation       | `S.junction[k].amount ← C.farside[k].amount` (numeric, equal) | `← C.farside[k].amount` if numeric, else keep `S.junction[k].amount`                                               |
| Token propagation        | `S.junction[k].token ← C.farside[k].token`                  | `← C.farside[k].token` if non-null, else keep `S.junction[k].token`                                                  |
| Guards                   | Amount Guard (implicit in "equal")                          | Amount Guard, Junction Token Guard, and [Symbolic Junction Guard](#symbolic-junction-guard) when a symbolic side is present |

#### Survivor assignment

Once a port match is found, the engine picks S and C *before* evaluating the rule:

1. Is exactly one of `{A, B}` a value transformer?
2. If yes and that transformer is not pass-through, it is S and the other is C.
3. If yes but it is a pass-through transformer, fall through to step 4.
4. Otherwise use the rule's default C from the table above.
5. Verify the consumed symmetry invariant on C. If it fails, no merge.

#### Per-rule details

Rule 1 (numeric chain): Both items agree on the exact numeric amount at the junction, so the three fields (account, amount, and token) all propagate unconditionally.

Rule 2 (symbolic junction): When at least one junction side is symbolic, the three fields propagate independently with safety fallbacks:

- Account: always propagated. The recipient address is never overridden.
- Amount: propagated only when numeric, otherwise S's value is kept. A numeric amount is never replaced by `balance`, and `balance` is never upgraded by an unrelated number.
- Token: propagated only when non-null, otherwise S's value is kept. A `nullTokenNode` from a creation step carries no token identity.

When both sides are symbolic, the default `C = A` makes the last-executed instruction survive, consistent with Rule 1.

If either S or C has a symbolic junction amount, the [Symbolic Junction Guard](#symbolic-junction-guard) must also pass.

### Guards and edge cases

#### Symbolic Junction Guard

This guard applies to any Rule 2 merge where either S or C has a symbolic junction-side amount. Both checks must pass.

1. Safe-account check. The junction account must be in the transaction's safe-account set, built from [`accountResetNode`](#accountresetnode) declarations. This guarantees a known balance: zero when `resetValue` is omitted, or the resolved `resetValue`.
2. Fully-accounted depositor check. Every instruction before C in transaction order that lists the junction account as writable in its raw account list must be S, C, or an instruction already merged into S or C. This ensures that every writable touch of the junction is traceable to the merge chain, regardless of whether the intermediate instruction declares a value-flow output on the junction.
   - The instruction that declared the applicable `accountResetNode` for the junction account is exempt. Its writable touch *is* the reset that establishes the known balance.

If either check fails, no merge occurs. See [Symbolic Balance Merge Safety](#symbolic-balance-merge-safety) for the rationale.

#### Pre-merge cleanup

Before the merge pass runs, items with a recognized descriptor that resolve to `inputs=[]`, `outputs=[]`, and no `intent` are removed. They carry no value flow and no user-facing meaning (account initialization is a typical case).

- Unrecognized instructions (no IDL match) are never cleaned up; see [Unrecognized Instructions](#unrecognized-instructions).
- Items with a declared `intent` but no value flow remain visible (`Stake:Deactivate`, `Stake:Authorize`, and similar).

#### When no rule fires

If amounts disagree, junction tokens are incompatible, or the consumed symmetry invariant fails, no merge occurs and both items remain visible. The engine does not collapse instructions it cannot classify.

#### Display source selection

The survivor's sRFC 39 display definition always governs the merged result. Because the transformer is assigned as S during [Survivor Assignment](#survivor-assignment), no separate display source override is needed. The merge only overrides runtime values (account addresses and amounts) that feed into S's display template.

#### Algorithmic properties

- Early termination on writable overlap. If any unconsumed intermediate instruction references one of A's output accounts as writable in its raw account list, whether or not it is declared as a value-flow port, the forward scan terminates immediately.
- Re-scan after first-item survival. Whenever A survives a merge with updated outputs (B was consumed into A), the engine re-scans forward from A. For example, in `Transfer → SyncNative → Swap`, Transfer absorbs SyncNative under Rule 2 and then matches Swap.
- Guards. Amount Guard (Rules 1 and 2), Junction Token Guard (Rule 2), and Symbolic Junction Guard (Rule 2 with a symbolic side). See [Security Considerations](#security-considerations).
- No backtracking. The merge is single-pass and left-to-right. Consumed items are never reconsidered.

---

## IDL Extension Schema

All nodes defined below are optional extensions to existing sRFC 39 nodes. An IDL that omits them remains valid sRFC 39 and renders with the base interpolation and fallback behavior.

### valueFlowNode

An optional property on `instructionDisplayNode`. Declares how the instruction moves value between accounts.

```json
{
  "kind": "instructionDisplayNode",
  "intent": "Swap",
  "interpolatedIntent": "Swap {inAmount} for at least {quotedOutAmount}",
  "valueFlow": {
    "kind": "valueFlowNode",
    "inputs": [ ... ],
    "outputs": [ ... ],
    "accountResets": [ ... ],
    "mintAssociation": { ... },
    "ownerAssociation": { ... },
    "hideConditions": { ... }
  }
}
```

| Property           | Required | Description                                                                          |
| ------------------ | -------- | ------------------------------------------------------------------------------------ |
| `kind`             | Yes      | Always `"valueFlowNode"`                                                             |
| `inputs`           | Yes      | Array of `valueFlowPortNode`: value consumed                                         |
| `outputs`          | Yes      | Array of `valueFlowPortNode`: value produced                                         |
| `accountResets`    | No       | Array of [`accountResetNode`](#accountresetnode): accounts created or reset          |
| `mintAssociation`  | No       | [`mintAssociationNode`](#mintassociationnode) for instructions that create token accounts |
| `ownerAssociation` | No       | [`ownerAssociationNode`](#ownerassociationnode) binding a token account to an owner       |
| `hideConditions`   | No       | [`hideConditionsNode`](#hideconditionsnode) for post-merge lifecycle hiding              |

### valueFlowPortNode

Each port describes one leg of value movement: which account, how much, what token.

```json
{
  "kind": "valueFlowPortNode",
  "valueKind": "splToken",
  "account": "source",
  "amount": { "kind": "argumentAmountNode", "argument": "amount" },
  "token": { "kind": "accountMintTokenNode" }
}
```

| Property     | Required | Description                                                                                                                |
| ------------ | -------- | -------------------------------------------------------------------------------------------------------------------------- |
| `kind`       | Yes      | Always `"valueFlowPortNode"`                                                                                               |
| `valueKind`  | Yes      | Discriminator telling the engine how to resolve and match the port (see below)                                             |
| `account`    | Yes      | Name of an instruction account (must match an entry in `accounts[]`)                                                       |
| `amount`     | Yes      | An [amount resolution node](#amount-resolution-nodes)                                                                      |
| `token`      | Yes      | A [token resolution node](#token-resolution-nodes)                                                                         |
| `activeWhen` | No       | Array of predicates evaluated at annotation time; port is active only if all pass (see [Port Activation](#port-activation)) |

`valueKind` semantics:

| `valueKind`    | Account matching   | Amount handling                      | Token identity    |
| -------------- | ------------------ | ------------------------------------ | ----------------- |
| `native`       | By account address | Normal                               | Always `"native"` |
| `splToken`     | By account address | Normal                               | Mint address      |
| `confidential` | By account address | Decrypt with signer key, then normal | Mint address      |

All three are mutually compatible for [account matching](#glossary).

Ports should not reference protocol-internal accounts such as vaults, reserves, or pool accounts that have no meaning in the user's value flow. For example, a flash-loan borrow from an order reserve to the taker should declare only an output on the taker's account, not an input on the reserve. Declaring such accounts causes the merge to propagate them through the chain, surfacing internal plumbing instead of the user's economic intent.

### Token resolution nodes

Token resolution nodes determine the identity of the asset being moved. They are used for value-chain matching and for display.

| Node kind               | Properties                 | Resolution                                                                                            |
| ----------------------- | -------------------------- | ----------------------------------------------------------------------------------------------------- |
| `fixedTokenNode`        | `symbol: string`           | Statically known at descriptor authoring time. Use real mint address for SPL, or `"native"` for lamports |
| `accountMintTokenNode`  | `fallbackAccount?: string` | RPC resolved `token_balances[account_address].mint`. Falls back to address of `fallbackAccount` if absent |
| `nullTokenNode`         | (none)                     | No token, port is a creation marker with no value identity (e.g. CreateATA)                           |
| `accountIndexTokenNode` | `accountIndex: integer`    | Mint of `accounts[accountIndex]` (used by `Token:CloseAccount`)         |

Examples:

```json
{"kind": "fixedTokenNode", "symbol": "native"}
{"kind": "fixedTokenNode", "symbol": "So11111111111111111111111111111111111111112"}
{"kind": "accountMintTokenNode", "fallbackAccount": "sourceMint"}
{"kind": "accountIndexTokenNode", "accountIndex": 0}
```

### Amount resolution nodes

Amount resolution nodes determine the quantity of value being moved.

| Node kind                | Properties         | Resolution                                                                                       |
| ------------------------ | ------------------ | ------------------------------------------------------------------------------------------------ |
| `argumentAmountNode`     | `argument: string` | Decoded from a named instruction argument; `argument` is a dotted path (see below)               |
| `balanceAmountNode`      | (none)             | Symbolic: drain/fill the full account balance. Not numerically comparable                        |
| `literalAmountNode`      | `value: integer`   | Literal constant. `value: 0` means creation with no value                                        |
| `confidentialAmountNode` | `argument: string` | Encrypted Token-2022 amount; same dotted-path resolution. Resolves only if decryption succeeds   |

Dotted paths: the segments of `argument`, separated by `.`, descend through nested `structTypeNode` fields. For example, `"args.amountIn"` reads `amountIn` from a struct-typed argument. A path that fails to resolve yields an unresolved amount, and the containing item is then not mergeable. The same applies to `confidentialAmountNode` when decryption fails or the key is unavailable.

Typical usage:

- `argumentAmountNode`: Transfer (`"lamports"`), Swap (`"inAmount"`).
- `balanceAmountNode`: `CloseAccount`, `SyncNative`, `Delegate`.
- `literalAmountNode`: `CreateATA`.

### Port activation

A port may declare `activeWhen`: an array of predicates evaluated at annotation time. If any predicate fails, the port is excluded and the instruction behaves as if the port were not declared.

Predicates are either string identifiers from the [condition vocabulary](#condition-vocabulary), shared with `hideConditionsNode`, or structured objects:

| Predicate kind  | Properties     | Meaning                                                     |
| --------------- | -------------- | ----------------------------------------------------------- |
| `mintPredicate` | `mint: string` | The port's resolved token must equal the given mint address |

Example: `CloseAccount` ports active only for WSOL.

```json
{
  "kind": "valueFlowPortNode",
  "account": "account",
  "amount": {"kind": "balanceAmountNode"},
  "token": {"kind": "accountIndexTokenNode", "accountIndex": 0},
  "activeWhen": [{"kind": "mintPredicate", "mint": "So11111111111111111111111111111111111111112"}]
}
```

For any other mint the port is excluded and `CloseAccount` has `inputs=[], outputs=[]`. The instruction is then either treated as infrastructure or remains visible as an intent-only item with `intent: "Close"`.

### accountResetNode

Declares that an instruction creates or resets an account, establishing a known balance. The engine uses these declarations for two purposes:

1. To build a transaction-level safe-account set for the [symbolic junction guard](#symbolic-junction-guard).
2. When `resetValue` is set, to resolve `balanceAmountNode` on input ports of subsequent instructions referencing this account.

All safety semantics live in the descriptor. The engine itself has no hard-coded instruction knowledge.

The known balance is scoped to the `valueKind` domain of the ports that reference the account. A `splToken` reset asserts a known SPL-token balance; a `native` reset asserts a known lamport balance. The two domains do not interact (a zero SPL-token reset says nothing about rent lamports, for instance).

```json
{
  "kind": "valueFlowNode",
  "inputs": [],
  "outputs": [],
  "accountResets": [
    { "kind": "accountResetNode", "account": "tokenAccount" },
    { "kind": "accountResetNode", "account": "ata", "requirePreBalanceZero": true },
    {
      "kind": "accountResetNode",
      "account": "ata",
      "resetValue": { "kind": "argumentAmountNode", "argument": "ataBalance" }
    }
  ]
}
```

| Property                | Required | Default | Description                                                                                          |
| ----------------------- | -------- | ------- | ---------------------------------------------------------------------------------------------------- |
| `kind`                  | Yes      | —       | Always `"accountResetNode"`                                                                          |
| `account`               | Yes      | —       | Name of the instruction account being created or reset                                               |
| `requirePreBalanceZero` | No       | `false` | If `true`, account is only safe when RPC resolved `preBalance == "0"` or absent (for idempotent creates) |
| `resetValue`            | No       | —       | Amount node declaring the known balance after execution; if omitted, balance is `0`                  |
| `resetScope`            | No       | —       | Restricts which consumer instructions may treat this account as safe (see [Scoped resets](#scoped-resets)) |

There are two reset categories:

- Unconditional (`requirePreBalanceZero: false`, default). The instruction either fails at runtime if the account already exists (non-idempotent variants like `System:CreateAccount`), or it explicitly resets the effective balance (as Jupiter's `setTokenLedger` does). The account is unconditionally added to the safe set.
- Conditional (`requirePreBalanceZero: true`). The instruction may silently succeed when the account already exists with a non-zero balance (idempotent variants like `AToken:CreateIdempotent`). The account is added to the safe set only when the RPC-resolved `preBalance` is `0` or absent. The RPC must be called through a trusted source known by the wallet.

A known balance set by `resetValue` is invalidated as soon as a subsequent instruction has an output port on the same account, since that output mutates the balance.

When `resetValue` is set, the on-chain program is responsible for asserting that the account holds the declared balance. If that assertion fails, the transaction MUST revert.

#### Scoped resets

Some program-level resets establish a known balance only for a specific family of consumer instructions within the same program, not for the on-chain balance. The canonical example is Jupiter's `setTokenLedger`: it does not zero the token account, it snapshots the balance into a Jupiter-owned ledger PDA. Only Jupiter's `*WithTokenLedger` entry points subtract that snapshot to compute the effective delta. Any other program still sees the full account balance.

`resetScope` narrows the safe-set entry to a predicate evaluated against the consumed-side item of a Rule 2 merge:

```json
{
  "kind": "accountResetNode",
  "account": "tokenAccount",
  "resetScope": {
    "kind": "programInstructionScope",
    "programId": "JUP6LkbZbjS1jKKwapdHNy74zcZ3tLUZoi5QNyVTaV4",
    "instructions": [
      "routeWithTokenLedger",
      "sharedAccountsRouteWithTokenLedger",
      "sharedAccountsExactOutRouteWithTokenLedger",
      "exactOutRouteWithTokenLedger"
    ]
  }
}
```

A `programInstructionScope` matches when:

- `consumed.programId == scope.programId`, and
- `scope.instructions` is omitted (any instruction of that program matches), or `consumed.instructionName ∈ scope.instructions`.

### mintAssociationNode

Ephemeral token accounts that are created and destroyed in the same transaction do not appear in RPC `token_balances`. The creating instruction's account list does carry the mint address, and this node declares the mapping.

```json
{
  "kind": "mintAssociationNode",
  "account": "tokenAccount",
  "mint": "mint"
}
```

| Property  | Required | Description                                                            |
| --------- | -------- | ---------------------------------------------------------------------- |
| `kind`    | Yes      | Always `"mintAssociationNode"`                                         |
| `account` | Yes      | Instruction account whose address is the new token account             |
| `mint`    | Yes      | Instruction account whose address is the mint                          |

A pre-pass before annotation seeds the token-balance map with `{address_of(account): {mint: address_of(mint)}}` so that downstream instructions can resolve the ephemeral ATA.

Example: `System:CreateAccount` followed by `Token:InitializeAccount`. The system program creates a raw account; `InitializeAccount` binds it to a mint; the `mintAssociationNode` makes that binding visible to downstream instructions.

```json
{
  "kind": "instructionNode",
  "name": "initializeAccount",
  "accounts": [
    {"kind": "instructionAccountNode", "name": "account", ...},
    {"kind": "instructionAccountNode", "name": "mint", ...},
    {"kind": "instructionAccountNode", "name": "owner", ...},
    {"kind": "instructionAccountNode", "name": "rent", ...}
  ],
  "display": {
    "kind": "instructionDisplayNode",
    "valueFlow": {
      "kind": "valueFlowNode",
      "inputs": [],
      "outputs": [],
      "mintAssociation": {
        "kind": "mintAssociationNode",
        "account": "account",
        "mint": "mint"
      }
    }
  }
}
```

The same applies to `InitializeAccount3`, in both the SPL Token and Token-2022 variants.

### ownerAssociationNode

Token accounts can be bound to an owner inside the same transaction, either via `InitializeAccount{,2,3}` or the ATA program. `ownerAssociationNode` declares the binding so that the engine can derive it during annotation. Structurally it parallels `mintAssociationNode` (same `account` field, same attachment point on `valueFlowNode`), but the second field is `owner` instead of `mint`, and `owner` accepts one of two shapes.

The first shape, `accountRefNode`, resolves to a named instruction account's pubkey. It fits `AToken:CreateIdempotent` and similar instructions where the owner is an explicit account:

```json
{ "kind": "ownerAssociationNode", "account": "ata", "owner": { "kind": "accountRefNode", "name": "owner" } }
```

The second shape, `argumentRefNode`, resolves through decoded arguments using the same dotted-path convention as `argumentAmountNode`. Only `publicKeyTypeNode` leaves produce a usable address. This fits instructions like `Token:InitializeAccount2`, where the owner is passed as an instruction argument:

```json
{ "kind": "ownerAssociationNode", "account": "account", "owner": { "kind": "argumentRefNode", "path": "owner" } }
```

### hideConditionsNode

Declares transaction-scoped constraints for post-merge hiding.

Single rule set (AND semantics):

```json
{
  "kind": "hideConditionsNode",
  "rules": [
    {"target": {"kind": "accountRefNode", "name": "account"}, "condition": "createdInTransaction"},
    {"target": {"kind": "accountRefNode", "name": "destination"}, "condition": "isSigner"}
  ]
}
```

Multiple rule sets (OR of ANDs):

```json
{
  "kind": "hideConditionsNode",
  "ruleSets": [
    [
      {"target": {"kind": "accountRefNode", "name": "account"}, "condition": "createdInTransaction"},
      {"target": {"kind": "accountRefNode", "name": "destination"}, "condition": "isSigner"}
    ],
    [
      {"target": {"kind": "argumentRefNode", "path": "owner"}, "condition": "isSigner"}
    ]
  ]
}
```

The instruction is hidden when any rule set passes (OR across sets). Within each rule set, all conditions are ANDed. `rules` is syntactic sugar for a single-element `ruleSets`.

| Property   | Required | Description                                                                  |
| ---------- | -------- | ---------------------------------------------------------------------------- |
| `kind`     | Yes      | Always `"hideConditionsNode"`                                                |
| `rules`    | No\*     | Array of condition rules; all must hold for the instruction to be hidden     |
| `ruleSets` | No\*     | Array of rule arrays (OR of ANDs); hidden if any rule set passes             |

\* Exactly one of `rules` or `ruleSets` must be present.

Each rule:

| Property    | Required | Description                                                                |
| ----------- | -------- | -------------------------------------------------------------------------- |
| `target`    | Yes      | Reference to the address the condition applies to (shapes below)           |
| `condition` | Yes      | Constraint identifier from the [condition vocabulary](#condition-vocabulary) |

The `target` field uses the same shapes as `ownerAssociationNode.owner`:

- `{"kind": "accountRefNode", "name": "<instruction account name>"}`.
- `{"kind": "argumentRefNode", "path": "<dotted path>"}`. Only `publicKeyTypeNode` leaves produce usable addresses.

A `target` that fails to resolve (missing argument, non-pubkey leaf, unknown account name) makes the rule evaluate to `false`.

#### Condition vocabulary

| Condition              | True when                                                                                                                                       |
| ---------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| `createdInTransaction` | The account appears as an output `(account, 0, null)` of any instruction in the transaction                                                     |
| `isSigner`             | The account address matches the signer context, or is an ATA whose owner is the signer                                                          |
| `accountUsedElsewhere` | The account address appears in any other instruction's input or output ports, or in any other instruction's raw account list                    |
| `isAnotherSigner`      | The account is a transaction signer and is neither the device signer nor an ATA owned by the device signer                                      |


### Schema examples

#### Example 1: `transferChecked`

A Token-2022 `transferChecked` instruction, enriched with sRFC 39 base display nodes and the value-flow extension. This is the minimal useful value flow: one input port, one output port, and no resets, associations, or hide conditions.

```json
{
  "kind": "instructionNode",
  "name": "transferChecked",
  "docs": [],
  "optionalAccountStrategy": "programId",
  "display": {
    "kind": "instructionDisplayNode",
    "intent": "Transfer",
    "interpolatedIntent": "Transfer {amount} to {destination}",
    "valueFlow": {
      "kind": "valueFlowNode",
      "inputs": [
        {
          "kind": "valueFlowPortNode",
          "account": "source",
          "amount": {"kind": "argumentAmountNode", "argument": "amount"},
          "token": {"kind": "accountMintTokenNode", "fallbackAccount": "mint"}
        }
      ],
      "outputs": [
        {
          "kind": "valueFlowPortNode",
          "account": "destination",
          "amount": {"kind": "argumentAmountNode", "argument": "amount"},
          "token": {"kind": "accountMintTokenNode", "fallbackAccount": "mint"}
        }
      ]
    }
  },
  "accounts": [
    {
      "kind": "instructionAccountNode",
      "name": "source",
      "isWritable": true, "isSigner": false, "isOptional": false,
      "display": {
        "kind": "accountDisplayNode",
        "skip": "always"
      }
    },
    {
      "kind": "instructionAccountNode",
      "name": "mint",
      "isWritable": false, "isSigner": false, "isOptional": false,
      "display": {
        "kind": "accountDisplayNode",
        "label": "Token Mint",
        "skip": "withAdditionalMetadata"
      }
    },
    {
      "kind": "instructionAccountNode",
      "name": "destination",
      "isWritable": true, "isSigner": false, "isOptional": false,
      "display": {
        "kind": "accountDisplayNode",
        "label": "To"
      }
    },
    {
      "kind": "instructionAccountNode",
      "name": "authority",
      "isWritable": false, "isSigner": "either", "isOptional": false,
      "display": {
        "kind": "accountDisplayNode",
        "skip": "always"
      }
    }
  ],
  "arguments": [
    {
      "kind": "instructionArgumentNode",
      "name": "discriminator",
      "defaultValueStrategy": "omitted",
      "type": {"kind": "numberTypeNode", "format": "u8", "endian": "le"},
      "defaultValue": {"kind": "numberValueNode", "number": 12},
      "display": {
        "kind": "argumentDisplayNode",
        "skip": "always"
      }
    },
    {
      "kind": "instructionArgumentNode",
      "name": "amount",
      "docs": ["The amount of tokens to transfer."],
      "type": {"kind": "numberTypeNode", "format": "u64", "endian": "le"},
      "display": {
        "kind": "argumentDisplayNode",
        "label": "Amount",
        "formatter": {
          "kind": "amountFormatterNode",
          "decimals": "decimals",
          "token": "mint"
        }
      }
    },
    {
      "kind": "instructionArgumentNode",
      "name": "decimals",
      "type": {"kind": "numberTypeNode", "format": "u8", "endian": "le"},
      "display": {
        "kind": "argumentDisplayNode",
        "skip": "always"
      }
    }
  ],
  "discriminators": [
    {"kind": "fieldDiscriminatorNode", "name": "discriminator", "offset": 0}
  ]
}
```

The value-flow layer describes the instruction as consuming value from `source` and producing it into `destination`. The amount is decoded from the `amount` argument, and the token is resolved from each account's mint. The engine uses this to chain `transferChecked` with adjacent instructions through shared accounts.

#### Example 2: `createIdempotent`

```json
{
  "kind": "instructionNode",
  "name": "createIdempotent",
  "docs": [],
  "optionalAccountStrategy": "programId",
  "display": {
    "kind": "instructionDisplayNode",
    "intent": "Create Token Account",
    "valueFlow": {
      "kind": "valueFlowNode",
      "inputs": [],
      "outputs": [
        {
          "kind": "valueFlowPortNode",
          "valueKind": "splToken",
          "account": "ata",
          "amount": {"kind": "literalAmountNode", "value": 0},
          "token": {"kind": "nullTokenNode"}
        }
      ],
      "accountResets": [
        {
          "kind": "accountResetNode",
          "account": "ata",
          "requirePreBalanceZero": true
        }
      ],
      "mintAssociation": {
        "kind": "mintAssociationNode",
        "account": "ata",
        "mint": "mint"
      },
      "ownerAssociation": {
        "kind": "ownerAssociationNode",
        "account": "ata",
        "owner": {"kind": "accountRefNode", "name": "owner"}
      },
      "hideConditions": {
        "kind": "hideConditionsNode",
        "rules": [
          {"target": {"kind": "accountRefNode", "name": "ata"}, "condition": "accountUsedElsewhere"},
          {"target": {"kind": "accountRefNode", "name": "owner"}, "condition": "isSigner"}
        ]
      }
    }
  },
  "accounts": [
    {
      "kind": "instructionAccountNode",
      "name": "payer",
      "isWritable": true, "isSigner": true, "isOptional": false,
      "display": {
        "kind": "accountDisplayNode",
        "skip": "always"
      }
    },
    {
      "kind": "instructionAccountNode",
      "name": "ata",
      "isWritable": true, "isSigner": false, "isOptional": false,
      "display": {
        "kind": "accountDisplayNode",
        "label": "Token Account"
      }
    },
    {
      "kind": "instructionAccountNode",
      "name": "owner",
      "isWritable": false, "isSigner": false, "isOptional": false,
      "display": {
        "kind": "accountDisplayNode",
        "label": "Owner"
      }
    },
    {
      "kind": "instructionAccountNode",
      "name": "mint",
      "isWritable": false, "isSigner": false, "isOptional": false,
      "display": {
        "kind": "accountDisplayNode",
        "label": "Token Mint"
      }
    },
    {
      "kind": "instructionAccountNode",
      "name": "systemProgram",
      "isWritable": false, "isSigner": false, "isOptional": false,
      "display": {
        "kind": "accountDisplayNode",
        "skip": "always"
      }
    },
    {
      "kind": "instructionAccountNode",
      "name": "tokenProgram",
      "isWritable": false, "isSigner": false, "isOptional": false,
      "display": {
        "kind": "accountDisplayNode",
        "skip": "always"
      }
    }
  ],
  "arguments": [
    {
      "kind": "instructionArgumentNode",
      "name": "discriminator",
      "defaultValueStrategy": "omitted",
      "type": {"kind": "numberTypeNode", "format": "u8", "endian": "le"},
      "defaultValue": {"kind": "numberValueNode", "number": 1},
      "display": {
        "kind": "argumentDisplayNode",
        "skip": "always"
      }
    }
  ],
  "discriminators": [
    {"kind": "fieldDiscriminatorNode", "name": "discriminator", "offset": 0}
  ]
}
```

Value-flow layer:

- The output port on `ata` uses `literalAmountNode(0)` and `nullTokenNode`. It surfaces a creation marker without contributing value to the chain. The empty `inputs` list combined with a zero-amount, null-token output distinguishes a creation from a value-producing instruction.
- `accountResets` with `requirePreBalanceZero: true` encodes the idempotent semantics. The ATA is added to the safe-account set only when the on-chain pre-balance is zero or absent. This unlocks downstream symbolic-junction merges, for example `SyncNative` followed by a swap that reads `balance` on the same ATA, as required by the [Symbolic Junction Guard](#symbolic-junction-guard).
- `mintAssociation` seeds the mint map at annotation time so that subsequent instructions can resolve `ata`'s mint without an RPC `token_balances` entry.
- `ownerAssociation` declares that `ata` is owned by `owner`. The runtime enforces this (the ATA address is `PDA(owner, mint, tokenProgram)`), so the engine can evaluate `isSigner(ata)` without on-chain state.
- `hideConditions` hides the create post-merge only when the ATA is referenced elsewhere in the transaction and the owner is the device signer. Together these prevent the [ATA-create attack](#ata-create-hiding-safety), in which a malicious creator opens an ATA for someone else.

---

## Security Considerations

### Advanced-user display

Merge and hide are display optimizations and not consensus rules. The transaction signed on-chain is always the full instruction list. A wallet implementing this extension MUST expose an opt-in advanced or expert view that disables both merge and hide and renders every instruction independently using sRFC 39 base, or raw when no display exists.

Whether the advanced view is on by default or hidden behind a "show details" affordance is a wallet UX decision, out of scope here. The requirement is only that the affordance exists and is reachable.

### Amount Guard

Rule 1's numeric-equality requirement prevents false merges between independent operations that happen to share an account. The engine must never understate user spending nor overstate what the user receives.

### Junction Token Guard

Rule 2 requires compatible junction tokens. Without this check, the engine could pair A's numeric amount with B's token identity, producing a mislabelled display such as "Transfer 2,000,000,000 USDC" when the actual intent was a 2 SOL transfer.

### Recipient propagation safety

Rule 2 always takes the concrete far-side account from the consumed item. If a `CloseAccount` directs funds away from the signer, that destination appears in the final output; it is never silently replaced or hidden.

Consider a swap that outputs WSOL followed by a malicious `CloseAccount` that redirects the unwrap to an attacker:

```
[CreateATA(wsol), JupiterSwap(USDC→WSOL), MaliciousCloseAccount(dest=attacker)]
```

`CloseAccount` merges via Rule 2, because both junction amounts are symbolic, and its destination propagates into the surviving output: `output: 0.5 WSOL to [attacker_address]`.

1. Through merge propagation, Rule 2 updates the survivor's output to `CloseAccount`'s destination, and the attacker address appears in the merged intent.
2. As a hide-condition fallback, even when no merge fires, `CloseAccount`'s `hideConditionsNode` requires `isSigner` on the destination. Since the attacker is not the device signer, `CloseAccount` stays visible as a separate item.

### Symbolic balance merge safety

Any merge in which either side has a symbolic junction amount (`balanceAmountNode`) can hide a value discrepancy when the junction account has a pre-existing balance: if C has the symbolic amount, the display shows S's numeric amount while C's drain actually reads the full balance.

The [symbolic junction guard](#symbolic-junction-guard) blocks this with two checks:

1. Safe-account check: the junction account must be in the safe set built from [`accountResetNode`](#accountresetnode) declarations, otherwise the merge is blocked.
2. Fully-accounted depositor check: every instruction before C that lists the junction account as writable in its raw account list, regardless of declared value-flow output, must be S, C, or already merged into either. Read-only references are ignored.

### Unrecognized instructions

An instruction is unrecognized when it lacks a recognized IDL `instructionNode` (unknown program, unknown discriminator, or a descriptor without a display overlay). Unrecognized instructions are a security risk because they may mutate accounts whose balance, owner, mint, reset state, or display/hide predicate the merge engine relies on.

Wallet vendors MAY disable pre-merge cleanup, merge, and hide for the whole transaction whenever any instruction is unrecognized. In that mode, each instruction is rendered with sRFC 39 base per-instruction display, or left as a raw instruction when no display exists. The wallet UX response (blind sign, list verbatim, warn) is out of scope for this specification.

Wallet vendors MAY run the merge engine around unrecognized instructions only when they can verify that those instructions do not write to any account used by the merge engine. Partial merging is unsafe when an unrecognized instruction can mutate any merge-relevant account.

Common sidecar instructions, including memo, compute-budget, priority-fee or tip patterns, and other ecosystem-standard programs, should still have curated IDLs so devices can distinguish benign instructions from unknown value movement.

### `isSigner` constraint

`isSigner` MUST match the device user's signing key, not the fee payer or any other transaction signer. It also matches ATAs whose owner is the device signer, which allows hiding lifecycle instructions such as `Token:CloseAccount` that reference a signer-owned ATA rather than the raw signer pubkey.

Two `ata_owners` sources are admissible:

1. Attested bindings, from a trusted source such as a signed-context backend or a chain-state RPC re-verified by a signer. These cover pre-existing accounts for which the transaction reveals no owner.
2. Transaction-derived bindings, produced by the association pre-pass from [`ownerAssociationNode`](#ownerassociationnode) declarations. These cover accounts the transaction creates or initializes, and are trustworthy because the Solana runtime enforces them. Either (a) the ATA is `PDA(owner, mint, tokenProgram)` and a mismatched `owner` argument makes the instruction fail, or (b) `InitializeAccount{,2,3}` writes `owner` into the token account header during execution.

On disagreement, the engine prefers the transaction-derived binding (because it is runtime-enforced) and warns that the attested binding may be stale or malicious.

### IDL authenticity (sRFC 39 alignment)

This extension inherits sRFC 39's IDL authenticity model and introduces no new trust assumptions. A malicious `valueFlowNode` could hide legitimate instructions or merge unrelated operations, but malicious `intent` strings carry the same risk in sRFC 39 base. IDL curation and authenticity verification mitigate both.

---

## Worked Examples

### Example 1: Jupiter swap with WSOL wrap and close

Transaction: `[ComputeBudget:SetLimit, ComputeBudget:SetPrice, AToken:CreateIdempotent(wsol_ata), System:Transfer(0.5 SOL → wsol_ata), Token:SyncNative(wsol_ata), Jupiter:Route(0.5 WSOL → 100 USDC), Token:CloseAccount(wsol_ata)]`

This is the canonical swap pattern from the [pipeline walkthrough](#pipeline-walkthrough), traced in full. The user pays 0.5 SOL and expects at least 100 USDC. Because Jupiter trades SPL tokens, the wallet adapter wraps SOL into WSOL with `CreateIdempotent → Transfer → SyncNative` before the swap, and unwraps with `CloseAccount` after. The ComputeBudget instructions affect fees but carry no value flow.

Step 0: Annotate.

```
ComputeBudget × 2             (no display overlay, handled at render only)

AToken:CreateIdempotent       inputs=[]                              outputs=[(wsol_ata, 0, null)]
                              accountResets=[(wsol_ata, requirePreBalanceZero=true)]
                              mintAssociation=(account=wsol_ata, mint=WSOL)
                              ownerAssociation=(account=wsol_ata, owner=accountRefNode("owner"))
                              hideConditions: accountUsedElsewhere(wsol_ata) AND isSigner(owner)

System:Transfer               inputs=[(signer, 0.5, native)]         outputs=[(wsol_ata, 0.5, native)]
                              intent="Transfer"

Token:SyncNative              inputs=[(wsol_ata, balance, native)]   outputs=[(wsol_ata, balance, WSOL)]
                              (pass-through value transformer: symmetric, all-symbolic)

Jupiter:Route                 inputs=[(wsol_ata, 0.5, WSOL)]         outputs=[(usdc_ata, 100, USDC)]
                              intent="Swap"

Token:CloseAccount            inputs=[(wsol_ata, balance, WSOL)]     outputs=[(signer, balance, WSOL)]
                              activeWhen=mintPredicate(WSOL)
                              hideConditions: isSigner(destination) AND accountUsedElsewhere(wsol_ata)
```

- `CreateIdempotent`'s `mintAssociationNode` seeds `wsol_ata = WSOL`, so the ephemeral ATA is resolvable.
- Its `accountResetNode` with `requirePreBalanceZero=true` adds `wsol_ata` to the safe set when the pre-TX balance is zero, which enables the [symbolic-junction guard](#symbolic-junction-guard) for downstream merges.
- `CloseAccount`'s `activeWhen` gates its ports on WSOL. For any other mint the ports are excluded.

Step 1: Merge.

```
ComputeBudget × 2          → no recognized display overlay, only used to compute fees.

CreateIdempotent           → forward scan from outputs (wsol_ata, 0, null):
                             First port match is SyncNative.inputs (wsol_ata, balance, native).
                             Junction amount 0 (numeric) on A vs balance (symbolic) on B
                               → Rule 2 candidate. Default C = SyncNative (symbolic side).
                             Symbolic-junction guard:
                               • safe-account check: wsol_ata IS in the safe set.       ✓
                               • fully-accounted-depositor check: Transfer writes
                                 wsol_ata before C (=SyncNative) and is NOT yet in any
                                 chain. CreateIdempotent itself is exempt as the
                                 reset declarer.                                        ✗
                             Guard fails → no merge. CreateIdempotent stays separate.

Transfer                   → forward scan from outputs (wsol_ata, 0.5, native):
                             Port match on SyncNative.inputs (wsol_ata, balance, native).
                             Junction amount 0.5 (numeric) on A vs balance (symbolic) on B
                               → Rule 2. SyncNative is a value transformer (native→WSOL)
                               but pass-through (symmetric, all-symbolic), so the
                               transformer override does NOT fire. Default C = SyncNative.
                             Symbolic-junction guard:
                               • safe-account check: wsol_ata in the safe set.          ✓
                               • depositor check: CreateIdempotent writes wsol_ata
                                 before C, but it IS the reset declarer for wsol_ata
                                 → exempt. Transfer is S itself.                        ✓
                             → Rule 2 fires. Consume SyncNative. S becomes:
                                 S.inputs  = (signer,   0.5,     native)   (unchanged)
                                 S.outputs = (wsol_ata, 0.5,     WSOL)     (token from C's far side)

                           → Re-scan from S (outputs changed):
                             Port match on Route.inputs (wsol_ata, 0.5, WSOL).
                             Junction amounts 0.5 == 0.5 (both numeric) → Rule 1.
                             Both items are value transformers (Transfer is now native→WSOL
                             after absorbing SyncNative; Route is WSOL→USDC). Tie →
                             default C = A = Transfer.
                             → Rule 1 fires. Consume Transfer. S = Route becomes:
                                 S.inputs  = (signer,   0.5, native)       (from C's far side)
                                 S.outputs = (usdc_ata, 100, USDC)         (unchanged)

                           → Re-scan from S = Route. S.outputs (usdc_ata, 100, USDC) vs
                             CloseAccount.inputs (wsol_ata, balance, WSOL): account
                             mismatch. No further matches.

CloseAccount               → forward scan from outputs (signer, balance, WSOL): no
                             later item exists. Stays separate.

Survivors after merge: ComputeBudget × 2, CreateIdempotent, Route (merged with Transfer
and SyncNative), CloseAccount.
```

Step 2: Hide.

```
ComputeBudget × 2:    no value flow                                       → Used to compute fees

CreateIdempotent:
  accountUsedElsewhere(wsol_ata) → ✓   (Transfer, SyncNative, Route, and CloseAccount
                                        all reference wsol_ata)
  isSigner(owner)                → ✓   (binding sourced from ownerAssociationNode +
                                        device signer key)
                                                                          → HIDDEN

Route (merged):       no hide rules                                       → VISIBLE

CloseAccount:
  isSigner(destination)          → ✓
  accountUsedElsewhere(wsol_ata) → ✓
                                                                          → HIDDEN
```

The wallet displays a single intent: `"Swap 0.5 SOL for at least 100 USDC"`.

The merge collapses `Transfer → SyncNative → Route` into Route's `Swap` intent. Transfer's signer-side SOL input propagates via Rule 1, and SyncNative's far-side WSOL token identity propagates via Rule 2. Both lifecycle instructions (`CreateIdempotent` and `CloseAccount`) are hidden: `wsol_ata` is used elsewhere in the transaction, and the signer-owned binding is verified through `mintAssociationNode` and `ownerAssociationNode`. The ComputeBudget instructions are used to compute fees.

`amountFormatterNode` scales raw amounts (`500_000_000 lamports → 0.5 SOL`, `100_000_000 USDC atoms → 100 USDC`), and the survivor's template `"Swap {inAmount} for at least {quotedOutAmount}"` produces the final string.

### Example 2: Native staking via CreateAccountWithSeed

Transaction: `[System:CreateAccountWithSeed(amount=49.6 SOL), Stake:Initialize, Stake:DelegateStake]`

Step 0: Annotate.

```
System:CreateAccountWithSeed  inputs=[(payer=signer, 49.6, native)]    outputs=[(stake_acct, 49.6, native)]
                              accountResets=[(stake_acct, resetValue=argumentAmountNode("amount") → 49.6)]
                              hideConditions: accountUsedElsewhere(newAccount) AND isSigner(payer)

Stake:Initialize              inputs=[]  outputs=[]
                              hideConditions: accountUsedElsewhere(stake)
                                              AND isSigner(arg0.staker)
                                              AND isSigner(arg0.withdrawer)

Stake:DelegateStake           inputs=[(stake_acct, balance, native)]   outputs=[]
                              intent="Stake"
```

`CreateAccountWithSeed`'s port amounts and its `accountResetNode.resetValue` all resolve to the same `amount` argument, so the reset asserts a known native balance of 49.6 SOL on `stake_acct`. `DelegateStake` is input-only (one input, zero outputs), a value transformer because of its asymmetric ports. No intermediate instruction has an output port on `stake_acct`, so the [known-balance map](#accountresetnode) seeded by the reset resolves the `balanceAmountNode` to the concrete `49.6` rather than leaving it symbolic.

Step 1: Merge.

```
CreateAccountWithSeed   → scan forward: Initialize is intermediate and lists stake_acct
                          in its raw writable account list (the runtime mutates the stake
                          account header).
    → writable-overlap early termination. The forward scan stops; CreateAccountWithSeed
      remains as a standalone item.

Stake:Initialize        → empty value-flow ports but srfc 39 base declares an intent.
                          Pre-merge cleanup does not apply.
                          → remains.

Stake:DelegateStake     → no outputs, no forward scan starts from it. Remains.

Survivors after merge: CreateAccountWithSeed, Initialize, DelegateStake.
```

No merge fires here. The writable-overlap guard is positional, and `Initialize` legitimately writes the stake-account header to record the staker and withdrawer authorities. Refusing to merge is safer than guessing which writes are harmless.

Step 2: Hide.

```
CreateAccountWithSeed:
  accountUsedElsewhere(newAccount) → ✓  (Initialize and DelegateStake reference stake_acct)
  isSigner(payer)                  → ✓
                                                                          → HIDDEN

Initialize:
  accountUsedElsewhere(stake)      → ✓  (DelegateStake references stake_acct)
  isSigner(arg0.staker)            → ✓
  isSigner(arg0.withdrawer)        → ✓
                                                                          → HIDDEN

DelegateStake:                     no hide rules                          → VISIBLE
```

`DelegateStake` survives alone, rendered as `"Stake 49.6 SOL"`. Its sole input port carries the resolved `(stake_acct, 49.6, native)` triple. No merge fired, but resolving `balanceAmountNode` against the [`accountResetNode`'s `resetValue`](#accountresetnode) surfaces the staked amount without consulting on-chain state. Without a preceding reset declaration, `DelegateStake` alone would render as a symbolic "Stake" with no quantity.

### Example 3: Jupiter two-hop swap and tip transfer

Transaction: `[Jupiter:setTokenLedger, Jupiter:sharedAccountsRoute, Jupiter:sharedAccountsRouteWithTokenLedger, System:Transfer]`

A real Jupiter v6 route splits a single user swap across two pools:

- Hop 1 publishes a fixed `quotedOutAmount` in the intermediate token.
- Hop 2 is a `WithTokenLedger` variant that reads the intermediate ATA's balance via `balanceAmountNode`, consuming whatever hop 1 produced.
- `setTokenLedger` runs first and snapshots the pre-hop balance into a program-owned ledger PDA so that hop 2 can compute the delta.
- A `System:Transfer` at the end pays a tip to a separate recipient.

Step 0: Annotate.

```
Jupiter:setTokenLedger             inputs=[]   outputs=[]
                                   accountResets=[(tokenAccount=eth_ata,
                                                   resetScope=programInstructionScope(
                                                     Jupiter,
                                                     [sharedAccountsRouteWithTokenLedger,
                                                      routeWithTokenLedger]))]
                                   hideConditions: accountUsedElsewhere(tokenAccount)

Jupiter:sharedAccountsRoute        inputs=[(usdc_ata, 14.69, USDC)]
                                   outputs=[(eth_ata, 0.006, ETH)]
                                   intent="Swap"

Jupiter:sharedAccountsRoute
       WithTokenLedger             inputs=[(eth_ata, balance, ETH)]
                                   outputs=[(usdc_ata, 14.71, USDC)]
                                   intent="Swap"

System:Transfer                    inputs=[(signer, 0.0000015, native)]
                                   outputs=[(tipper, 0.0000015, native)]
                                   intent="Transfer"
```

`usdc_ata` and `eth_ata` are the signer's USDC and ETH ATAs (the route round-trips back into the same USDC ATA). Both swaps are symmetric value transformers. Hop 2's input is a `balanceAmountNode`, a symbolic drain on `eth_ata`. `setTokenLedger`'s reset is scoped: it only authorizes a symbolic read on `eth_ata` when the consumer is one of Jupiter's `*WithTokenLedger` variants.

Step 1: Merge.

```
setTokenLedger              → empty value-flow ports, but it carries an applicable
                              accountResetNode. Pre-merge cleanup does NOT apply
                              (the reset declaration is meaningful state).
                              Safe-account set gets: {eth_ata, scoped to
                              [sharedAccountsRouteWithTokenLedger, routeWithTokenLedger]}.
                              No outputs → no forward scan from this item.

sharedAccountsRoute      → outputs (eth_ata, 0.006, ETH) match
   ↓                        sharedAccountsRouteWithTokenLedger.inputs
sharedAccountsRoute         (eth_ata, balance, ETH).
   WithTokenLedger          Junction tokens equal (ETH). hop2's amount is symbolic →
                            Rule 2 candidate.

                            Symbolic-junction guard:
                              • safe-account check: eth_ata is in the safe set AND
                                the consumed item's (programId, instructionName)
                                = (Jupiter, sharedAccountsRouteWithTokenLedger)
                                satisfies setTokenLedger's resetScope.            ✓
                              • fully-accounted-depositor check: hop1 writes eth_ata,
                                but hop1 IS the survivor (S). setTokenLedger writes
                                eth_ata, but is the reset declarer (exempt for this
                                consumer). No other writable touches.             ✓

                            Default C = hop2 (symbolic junction side). S = hop1.
                            Both items transformers (tie) → override does not fire.

                            → Rule 2 fires. Consume hop2. S becomes:
                                S.inputs  = (usdc_ata, 14.69, USDC)        (unchanged)
                                S.outputs = (usdc_ata, 14.71, USDC)        (from hop2)

System:Transfer             → forward scan from the merged swap: outputs (usdc_ata,
                              USDC, splToken) vs Transfer.inputs (signer, native,
                              native). Account mismatch AND valueKind mismatch
                              (splToken vs native).                          no match.
                              Transfer has no further forward items.

Survivors after merge: setTokenLedger, sharedAccountsRoute (merged), System:Transfer.
```

Step 2: Hide.

```
setTokenLedger:
  accountUsedElsewhere(tokenAccount=eth_ata) → ✓   (hop1 references eth_ata; the
                                                   consumed hop2 also references it)
                                                                          → HIDDEN

sharedAccountsRoute (merged): no hide rules                               → VISIBLE
System:Transfer:              no hide rules                               → VISIBLE
```

The wallet renders the route and the tip as separate items:

```
[1/2] Swap (Jupiter)
        You Pay:              14.69 USDC
        Minimum Received:     14.71 USDC
        Source / Destination: signer's USDC ATA
        Slippage Tolerance:   0.1%

[2/2] Transfer (System)
        Recipient:  tipper
        Amount:     0.0000015 SOL
```

The two-hop route collapses into a single `Swap` intent. `setTokenLedger` disappears because both swaps reference `eth_ata`, so `accountUsedElsewhere` holds. The tip stays visible: its only port lives on an unrelated account (`tipper`) and on a different `valueKind` (`native` versus `splToken`), so no port match against the merged swap exists.

The scoped reset is what makes the collapse sound. `setTokenLedger` does not zero the on-chain `eth_ata`, it only snapshots the balance into Jupiter's ledger PDA. A consumer outside `[sharedAccountsRouteWithTokenLedger, routeWithTokenLedger]` would fail the [`resetScope`](#scoped-resets) predicate during the symbolic-junction guard, blocking the merge.

### Example 4: Jupiter flash-fill order (no protocol-internal ports)

Transaction: `[AToken:Create, LimitOrder:PreFlashFillOrder, AToken:Create(wSOL), System:Transfer, Token:SyncNative, AToken:Create(JUP), AToken:Create(USDC), Jupiter:Route, Token:CloseAccount, LimitOrder:FlashFillOrder]`

This is a taker filling a limit order via Jupiter's flash-fill mechanism. The on-chain circuit is:

> order reserve → taker (input mint) → wSOL wrap → swap SOL → JUP → taker pays JUP to maker (output mint)

The reserve and maker accounts are protocol-internal and MUST NOT appear as ports.

Correct value flow (no protocol-internal ports):

```
PreFlashFillOrder    inputs=[]                                              outputs=[(takerOutputAccount, makingAmount, SOL)]
                     intent="Flash Pre-fill"
                     (no input port: reserve is protocol-internal)

FlashFillOrder       inputs=[(takerInputAccount, maxTakingAmount, JUP)]     outputs=[]
                     intent="Flash Fill"
                     (no output port: makerOutputAccount is protocol-internal)
```

Both instructions have asymmetric ports (one port each), so they cannot be consumed by the merge (the consumed symmetry invariant requires `len(inputs) == len(outputs)`). They remain as standalone visible items.

Why not declare both sides? If `PreFlashFillOrder` declared `reserve` as an input and `FlashFillOrder` declared `makerOutputAccount` as an output, the merge would chain `reserve → taker → wSOL → swap → JUP → maker`. The final display would show the order's reserve as "You Pay" and the maker's account as "Destination", and neither belongs to the signer.

Merge (middle chain only):

```
Transfer → SyncNative  → Rule 2 (symbolic junction). Consume SyncNative.
Transfer → Route       → Rule 1 (amounts match). Consume Transfer.

Route survives with: input=(signer, 0.02, SOL), output=(signer_jup_ata, 6.75, JUP)
```

Hide: signer-owned ATA creates are hidden (`accountUsedElsewhere` and `isSigner`); `CloseAccount` is hidden (`isSigner` on destination).

The taker sees this:

```
[1/3] Flash Pre-fill: Receive 0.02 SOL from order
[2/3] Swap:           0.02 SOL → 6.75 JUP
[3/3] Flash Fill:     Pay ≤6.75 JUP to order
```

The display matches the pre-fill, source, settle pattern as it actually executes.