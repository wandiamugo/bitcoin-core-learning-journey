# Day 6: Understanding feature_assumevalid.py - Skipping Signature Validation on Old Blocks

**Test File:** `test/functional/feature_assumevalid.py`
**Category:** Feature Tests
**Complexity:** Medium-High

---

## Introduction

### The Audit Analogy

Imagine you're a new accountant joining a company that's been operating for 50 years. Your job is to verify the books are correct. You have two options:

**Option A (No Trust):** Audit every single receipt, invoice, and ledger entry from the company's founding in 1975 through today. This will take months.

**Option B (Selective Trust):** The company has been publicly audited every quarter by reputable firms, with results on file. You trust those past audits and focus only on verifying entries from the last quarter — work that hasn't been independently reviewed yet.

Option B doesn't mean you're being careless. It means you're being efficient: the old work has already been verified by the market, by time, by scrutiny. What's *new* hasn't been, so that's where your effort belongs.

Bitcoin's `-assumevalid` parameter works exactly this way. For blocks deep in Bitcoin's history — buried under years of work, reviewed by millions of nodes — Bitcoin Core can skip signature verification and trust that those blocks are valid. For recent blocks, full validation still occurs.

### Why This Matters for Developers

The `-assumevalid` optimization touches on a universal principle in software engineering: **trust proportional to evidence**. The older a piece of data, the more it has been tested, scrutinized, and validated by independent parties. New data — sitting at the edge of what's been verified — deserves more scrutiny.

You see this pattern across engineering:

- **Caches:** Frequently accessed data is assumed to be correct until a cache miss forces re-validation
- **TLS Certificate Chains:** Root CAs are trusted because they've been vetted; new leaves get more scrutiny
- **Package signing:** Packages from long-trusted publishers get less manual review than new ones
- **Database checkpoints:** Recovery skips re-validating blocks before the last checkpoint
- **Git shallow clones:** You trust that history before the shallow point was verified; only new commits need your attention

`feature_assumevalid.py` tests whether Bitcoin Core correctly implements this optimization — accepting old blocks while still catching invalid new ones.

### The Technical Version

`-assumevalid` is a Bitcoin Core parameter that accepts a block hash. When set, Bitcoin Core skips script/signature validation for blocks in the chain up to and including that block — *but only if those blocks are buried by at least two weeks of proof-of-work*. If a block is flagged as assumevalid but is near the tip (not deeply buried), Bitcoin Core still validates it fully.

The test builds a 2202-block chain where block 102 contains a transaction with an **intentionally invalid signature**. Three nodes are tested under different configurations to verify that the assumevalid logic is applied correctly — accepting the bad block when it's deeply buried and assumevalid-flagged, but rejecting it when it's not.

---

## What Is Script Validation?

Before exploring the test, we need to understand what's being skipped.

### Bitcoin's Script System

Every Bitcoin transaction output is locked by a **scriptPubKey** — a small program that defines who can spend it. Every transaction input must provide a **scriptSig** (and for SegWit, a witness) that satisfies that script.

A simple Pay-to-Public-Key (P2PK) transaction works like this:

```
scriptPubKey: <pubkey> OP_CHECKSIG
scriptSig:    <signature>

Execution:
1. Push <signature> onto stack
2. Push <pubkey> onto stack
3. OP_CHECKSIG: pop pubkey and signature, verify signature, push 1 (success) or 0 (fail)
```

Signature verification is **cryptographic work** — it requires:
1. Hashing the transaction data (SHA-256 twice)
2. Performing elliptic curve math (secp256k1 point operations)
3. Verifying the result matches the claimed signature

For Bitcoin's entire transaction history — hundreds of millions of inputs — this verification adds up to significant CPU work during Initial Block Download (IBD).

### Why Skip Signature Verification for Old Blocks?

For blocks buried deep in the chain, the economic argument for trusting them is overwhelming:

1. **Proof-of-work commitment:** Thousands of blocks worth of computational work sit on top of these blocks. Rewriting them would require outpacing the entire Bitcoin mining network.

2. **Community verification:** Every Bitcoin node that synced from genesis already validated these blocks. If they were invalid, they would have been rejected long ago.

3. **Time:** Bitcoin's oldest blocks have been validated billions of times by independent nodes over 15 years.

4. **The marginal risk is tiny:** A malicious signature in block 100,000 is economically useless. Even if the signature was technically invalid, the coins are already "spent" in Bitcoin's history — creating new valid transactions on top would require creating an entire alternative chain.

`-assumevalid` formalizes this reasoning: if you trust that a particular block hash is on the honest chain (based on its burial depth), you can skip re-verifying all the signatures in blocks up to that point.

---

## The Test: What Does It Do?

### The Setup

```python
def set_test_params(self):
    self.setup_clean_chain = True
    self.num_nodes = 3
    self.rpc_timeout = 120
```

A clean chain with 3 nodes, each playing a different role in verifying assumevalid behavior.

```python
def setup_network(self):
    self.add_nodes(3)
    self.start_node(0)
```

Crucially, only node 0 is started initially. The test needs to mine the "bad block" with an invalid signature *before* starting nodes 1 and 2, so it can pass the hash of that block as their `-assumevalid` parameter.

### Building the Chain

```python
# Get a pubkey for the coinbase TXO
_, coinbase_pubkey = generate_keypair()

# Create block 1 with a coinbase output to our key
height = 1
block = create_block(self.tip, create_coinbase(height, coinbase_pubkey), self.block_time)
block.solve()
self.block1 = block  # save for later
```

The test generates a keypair and creates block 1 with a coinbase transaction that pays to `coinbase_pubkey`. This is saved as `self.block1` because we'll need to spend it later.

```python
# Bury the block 100 deep so the coinbase output is spendable
for _ in range(100):
    block = create_block(self.tip, create_coinbase(height), self.block_time)
    block.solve()
    self.blocks.append(block)
```

Bitcoin's coinbase maturity rule requires that coinbase outputs are buried by at least 100 blocks before they can be spent. The test mines 100 empty blocks to satisfy this.

### The Invalid Transaction (Block 102)

```python
# Create a transaction spending the coinbase output with an invalid (null) signature
tx = CTransaction()
tx.vin.append(CTxIn(COutPoint(self.block1.vtx[0].txid_int, 0), scriptSig=b""))
tx.vout.append(CTxOut(49 * 100000000, CScript([OP_TRUE])))

block102 = create_block(self.tip, create_coinbase(height), self.block_time, txlist=[tx])
block102.solve()
```

This creates a transaction that:
- **Spends** the coinbase output from block 1 (the one locked to `coinbase_pubkey`)
- **Provides an empty scriptSig** (`b""`) — no signature at all
- **Outputs** 49 BTC locked with `OP_TRUE` (spendable by anyone)

This transaction is **cryptographically invalid**. Spending a P2PK output requires a valid ECDSA signature over the transaction data. An empty scriptSig will fail validation immediately.

The proof-of-work for block 102 is still valid (`.solve()` handles that). Only the transaction signature is wrong.

### Burying the Bad Block

```python
# Bury the assumed valid block 2100 deep
for _ in range(2100):
    block = create_block(self.tip, create_coinbase(height), self.block_time)
    block.solve()
    self.blocks.append(block)
```

2100 blocks are mined on top of block 102. This is the "burial" that makes assumevalid apply.

**Why 2100 blocks?** Bitcoin's assumevalid logic requires the assumed-valid block to be buried by at least two weeks of proof-of-work. On regtest, each block has a small target, so 2100 blocks provides sufficient burial. The final chain height is 2202 (1 genesis + 1 block1 + 100 maturity blocks + 1 block102 + 2100 burial blocks).

### Starting Nodes 1 and 2 with `-assumevalid`

```python
self.start_node(1, extra_args=["-assumevalid=" + block102.hash_hex])
self.start_node(2, extra_args=["-assumevalid=" + block102.hash_hex])
```

Now that we know `block102.hash_hex`, we can start nodes 1 and 2 configured to assume that block is valid.

### The Three Test Cases

**Node 0 — No assumevalid (strict validation):**

```python
p2p0 = self.nodes[0].add_p2p_connection(BaseNode())
p2p0.send_header_for_blocks(self.blocks[0:2000])
p2p0.send_header_for_blocks(self.blocks[2000:])

self.send_blocks_until_disconnected(p2p0)
self.wait_until(lambda: self.nodes[0].getblockcount() >= COINBASE_MATURITY + 1)
assert_equal(self.nodes[0].getblockcount(), COINBASE_MATURITY + 1)
```

Node 0 has no `-assumevalid` and validates every signature. When it reaches block 102, it finds the invalid signature and **rejects the block**. The P2P connection is disconnected (Bitcoin Core disconnects peers that send invalid blocks). Node 0 ends up at height 101 — it accepted block 1 through 101, but stopped at 102.

`COINBASE_MATURITY + 1 = 101`: genesis block (height 0) + block 1 (height 1) + 100 maturity blocks = height 101.

**Node 1 — assumevalid with deep burial (2202 blocks):**

```python
p2p1 = self.nodes[1].add_p2p_connection(BaseNode())
p2p1.send_header_for_blocks(self.blocks[0:2000])
p2p1.send_header_for_blocks(self.blocks[2000:])
with self.nodes[1].assert_debug_log(expected_msgs=[
    'Disabling signature validations at block #1',
    'Enabling signature validations at block #103'
]):
    for i in range(2202):
        p2p1.send_without_ping(msg_block(self.blocks[i]))
    p2p1.sync_with_ping(timeout=960)
assert_equal(self.nodes[1].getblock(self.nodes[1].getbestblockhash())['height'], 2202)
```

Node 1 has `-assumevalid=<block102 hash>` and receives all 2202 blocks. It syncs to height 2202, accepting block 102 despite the invalid signature.

The `assert_debug_log` check is critical — it verifies that Bitcoin Core actually *turned off* signature validation at some point (block #1, just after genesis) and turned it back on (block #103, just after block 102). This confirms the optimization is active, not just that the block happened to pass by chance.

**Node 2 — assumevalid but only syncing to block 200 (insufficient burial):**

```python
p2p2 = self.nodes[2].add_p2p_connection(BaseNode())
p2p2.send_header_for_blocks(self.blocks[0:200])

self.send_blocks_until_disconnected(p2p2)
self.wait_until(lambda: self.nodes[2].getblockcount() >= COINBASE_MATURITY + 1)
assert_equal(self.nodes[2].getblockcount(), COINBASE_MATURITY + 1)
```

Node 2 also has `-assumevalid=<block102 hash>`, but only receives headers and blocks up to block 200. At this point, block 102 is only buried by ~98 blocks — nowhere near two weeks of work.

Bitcoin Core's logic: **assumevalid only applies if the block is buried deeply enough.** Since node 2 only has 200 blocks in its chain, block 102 is near the tip. Full signature validation runs, the invalid signature is caught, and block 102 is rejected. Node 2 also ends at height 101.

---

## The C++ Implementation

### How `-assumevalid` is Stored

In `src/kernel/chainparams.h`, the chainparams hold the assumed-valid block hash:

```cpp
class CChainParams {
public:
    uint256 defaultAssumeValid;  // Block hash to assume is valid
    // ...
};
```

For mainnet, this is set to a hardcoded recent block hash in `src/kernel/chainparams.cpp`:

```cpp
// Mainnet (as of a recent release)
consensus.defaultAssumeValid = uint256S(
    "00000000000000000002a7c4c1e48d76c5a37902165a270156b7a8d72728a054"
);
```

Users can override this with `-assumevalid=<hash>` or disable it with `-assumevalid=0`.

### The Signature Check Gate (`src/validation.cpp`)

Deep in block validation, Bitcoin Core calls script validation for each transaction input. The check is gated on whether the current block is within the assumevalid range:

```cpp
// src/validation.cpp - ConnectBlock()
bool fScriptChecks = true;

if (!hashAssumeValid.IsNull()) {
    // Check if the block's ancestor chain contains the assumevalid block
    // AND if that block is buried by at least nMinimumChainWork
    const CBlockIndex* pindexAssumeValid =
        m_blockman.LookupBlockIndex(hashAssumeValid);

    if (pindexAssumeValid &&
        pindexAssumeValid->GetAncestor(pindex->nHeight) == pindex &&
        pindexAssumeValid->nHeight >= pindex->nHeight &&
        pindexAssumeValid->nChainWork >= nMinimumChainWork)
    {
        // This block is at or before the assumevalid point,
        // and that point is buried by enough work. Skip scripts.
        fScriptChecks = false;
    }
}

// ... later in ConnectBlock:
if (fScriptChecks) {
    // Full script/signature validation
    if (!CheckInputScripts(tx, state, view, flags, ...)) {
        return state.Invalid(
            BlockValidationResult::BLOCK_CONSENSUS,
            "bad-txns-inputs-missingorspent"
        );
    }
}
```

This is the gate that the test probes. When `fScriptChecks = false`, the invalid signature in block 102 passes silently. When `fScriptChecks = true`, it fails and the block is rejected.

### The Burial Depth Check

The condition `pindexAssumeValid->nChainWork >= nMinimumChainWork` is the burial depth check:

```cpp
// src/kernel/chainparams.cpp
// Minimum amount of chain work required before assumevalid applies
consensus.nMinimumChainWork = uint256S(
    "000000000000000000000000000000000000000052b2559353df4329b2e42b3a6"
);
```

`nChainWork` is a cumulative measure of all proof-of-work in the chain. If the chain from genesis to the assumevalid block doesn't meet the minimum work threshold, assumevalid is not applied. This prevents an attacker from creating a short chain with a fake "assumed valid" block and tricking a new node into accepting it without signature verification.

On regtest (used in tests), minimum chain work is very low, and the 2100-block burial satisfies it easily. With only 200 blocks, it doesn't.

### The Debug Log Messages

The test's `assert_debug_log` check verifies two specific log messages:

```
'Disabling signature validations at block #1'
'Enabling signature validations at block #103'
```

These are emitted by Bitcoin Core when it transitions into and out of the assumevalid range:

```cpp
// src/validation.cpp
if (!fScriptChecks && fScriptChecksWasPreviouslyEnabled) {
    LogInfo("Disabling signature validations at block #%d\n", pindex->nHeight);
}
if (fScriptChecks && !fScriptChecksWasPreviouslyEnabled) {
    LogInfo("Enabling signature validations at block #%d\n", pindex->nHeight);
}
```

- **Disabling at block #1:** Block 1 is the first block for which assumevalid applies (it's before block 102 and the chain is buried deep enough)
- **Enabling at block #103:** Block 103 is the first block *after* block 102 (the assumevalid block), so full validation resumes

This confirms that the optimization actually engaged — the test isn't just lucky that the bad block passed, it's explicitly verifying the code path was taken.

---

## The Complete Execution Flow

```
1. Node 0 starts (no assumevalid)
   ↓
2. Mine block 1 with coinbase to coinbase_pubkey
   ↓
3. Mine 100 empty blocks (maturity)
   ↓
4. Construct transaction spending block1 coinbase with EMPTY scriptSig (invalid!)
   ↓
5. Mine block102 containing the invalid transaction
   ↓
6. Mine 2100 more blocks on top of block102
   ↓
7. Start node1 and node2 with -assumevalid=<block102.hash>
   ↓
8. [Node 0 test] Send all headers + all blocks
   → Node 0 validates signatures → rejects block102 → stops at height 101
   ↓
9. [Node 1 test] Send all headers + all 2202 blocks
   → Node 1 sees assumevalid with deep burial
   → Disables signatures for blocks 1-102
   → Accepts block102 despite invalid signature
   → Re-enables signatures at block 103
   → Syncs to height 2202
   ↓
10. [Node 2 test] Send first 200 headers + 200 blocks
    → Node 2 sees assumevalid but burial is only ~98 blocks (insufficient)
    → Full signature validation runs for block102
    → Rejects block102 → stops at height 101
```

---

## Why This Test Matters

### 1. Significant IBD Performance Improvement

Script validation is one of the most CPU-intensive parts of IBD. On a typical modern machine:

- **Without assumevalid:** Full IBD can take 1-7 days
- **With assumevalid:** IBD time is dramatically reduced because the cryptographic work for old blocks is skipped

The exact speedup depends on hardware and network, but signature verification represents a significant fraction of total IBD CPU time. Skipping it for millions of historical inputs is a meaningful optimization.

### 2. No Security Compromise (When Used Correctly)

The key invariant is: **assumevalid blocks must be deeply buried**.

A block buried under two weeks of proof-of-work is, for practical purposes, immutable. To rewrite it would require:
1. More computational power than the entire Bitcoin mining network
2. Outpacing that network continuously until the rewritten chain surpasses the honest chain

This is economically infeasible. The assumevalid optimization correctly identifies that the signature verification for such blocks is redundant.

The test with node 2 — where assumevalid is set but the block isn't buried enough — is the critical safety check. It proves that Bitcoin Core doesn't blindly trust the assumevalid flag; it also verifies the burial depth.

### 3. The Subtle Attack Vector Being Tested

Consider this attack scenario:

1. Attacker creates a fake chain of 100 blocks
2. Block 50 contains a transaction that "creates" coins out of thin air (or steals someone's coins with a forged signature)
3. Attacker sets `-assumevalid=<block50 hash>` and tries to trick a node into accepting the chain

The node 2 test case verifies this attack fails: even if `-assumevalid` is set to block 50, if the chain is only 100 blocks long, signature validation still runs on block 50 and rejects the forgery.

### 4. Correct vs. Incorrect Handling of Edge Cases

The test carefully distinguishes between two scenarios that look similar but behave differently:

| Scenario | assumevalid set? | Block buried? | Result |
|----------|-----------------|---------------|--------|
| Node 0 | No | N/A | Validate all signatures; reject block 102 |
| Node 1 | Yes | Yes (2100 blocks) | Skip signatures for blocks 1-102; accept |
| Node 2 | Yes | No (only ~98 blocks) | Still validate signatures; reject block 102 |

The test verifies all three cases, ensuring each code path is correct.

---

## Comparison with AssumeUTXO

Having covered both `feature_assumevalid.py` and `feature_assumeutxo.py`, it's worth contrasting these two related optimizations:

| Aspect | `-assumevalid` | AssumeUTXO |
|--------|---------------|------------|
| What is skipped | Script/signature validation | Building the UTXO set from scratch |
| When is it verified | Never (trusted based on burial depth) | Background validation verifies it completely |
| Speed improvement | Moderate (CPU savings on old sigs) | Large (entire IBD from snapshot) |
| Trust requirement | Implicit (burial depth) | Explicit (hardcoded snapshot hashes) |
| Introduced in | Bitcoin Core 0.14 (2017) | Bitcoin Core 26.0 (2023) |
| What happens if wrong | You accepted a technically-invalid chain (extremely unlikely with deep burial) | Node detects mismatch and aborts |

Both optimizations target the same problem (slow IBD) but at different layers. `-assumevalid` skips cryptographic work but still validates the chain structure, proof-of-work, and UTXO set consistency. AssumeUTXO skips building the UTXO set entirely but validates it asynchronously.

Used together, they dramatically reduce IBD time while maintaining Bitcoin's security model.

---

## Key Takeaways

1. **`-assumevalid` skips signature validation for old, deeply-buried blocks** — not because those signatures don't matter, but because the burial depth makes forgery economically impossible

2. **Burial depth is a hard requirement** — assumevalid without sufficient burial (as in the node 2 test) triggers full validation anyway, preventing abuse

3. **The debug log messages are the test's smoking gun** — "Disabling/enabling signature validations" proves the optimization actually engaged, not just that the block passed

4. **Three nodes, three cases** — no assumevalid (strict), assumevalid with deep burial (skipped), assumevalid with shallow burial (still strict)

5. **This is a "trust proportional to evidence" pattern** — old, widely-verified, heavily-buried data deserves more trust than new, unverified data at the chain tip

6. **The optimization is significant** — script validation is CPU-intensive; skipping it for hundreds of millions of historical inputs meaningfully speeds up IBD

---

## Conclusion

`feature_assumevalid.py` tests one of Bitcoin Core's most elegant performance optimizations. By formalizing the observation that deeply-buried blocks are effectively immutable, it allows nodes to skip millions of redundant cryptographic operations during IBD without compromising security.

The test's three-node design is clean and complete: one node proves the invalid signature would normally fail (correctness baseline), one proves assumevalid works when properly applied (the optimization), and one proves assumevalid doesn't apply blindly without sufficient burial (the safety check).

For developers, this test teaches an important lesson: **not all validation is equally necessary at all times**. Old, well-established data that has been independently verified thousands of times doesn't need to be re-verified on every read. New data, sitting at the edge of what's been confirmed, deserves full scrutiny. Designing systems that apply trust proportionally — more scrutiny for new/untested data, less for old/established data — is a pattern that appears across every layer of engineering.

Bitcoin's `-assumevalid` implements this principle at the protocol level, and `feature_assumevalid.py` ensures it's implemented correctly.

---

## References

- **Bitcoin Core PR #9484** - Original `-assumevalid` implementation: https://github.com/bitcoin/bitcoin/pull/9484
- **Test file:** `test/functional/feature_assumevalid.py`
- **Core validation logic:** `src/validation.cpp` — `ConnectBlock()` (the `fScriptChecks` gate)
- **Chain parameters:** `src/kernel/chainparams.cpp` — `defaultAssumeValid`, `nMinimumChainWork`
