# Day 2: Understanding feature_addrman.py - Bitcoin Core's Address Book

**Test File:** `bitcoin/test/functional/feature_addrman.py`
**Category:** Feature Tests
**Complexity:** Moderate

---

## Introduction

### The Contact List Analogy

Imagine you have a phone. You've saved hundreds of contacts over the years — friends, family, work colleagues, the guy who fixed your car. Now imagine your phone crashes and recovers from a backup. When it loads your contact list back, a few things could go wrong:

1. **The backup file is from a future version of the app** — maybe your friend backed up their contacts on a newer phone and sent you the file. Your phone doesn't understand the new format. What should it do? Crash? No — it should say "I can't read this, but I'll keep my old contacts safe as a backup and start fresh."

2. **The backup is from the wrong phone** — you accidentally loaded contacts meant for your Android phone onto your iPhone. The file looks like a contact list, but the header says "Android." Your phone should reject it immediately.

3. **Someone tampered with the file** — maybe it got corrupted during download, or someone mischievously edited it. The file has a "seal" (like a wax seal on a letter) that you can check. If the seal is broken, you know something is wrong.

4. **The backup says it has 500 contacts, but there are actually 50** — the numbers in the header don't match reality. That's a red flag for corruption.

5. **The backup file is simply missing** — no problem. Just start with an empty contact list and build it up again.

This is exactly what `feature_addrman.py` tests. Bitcoin Core keeps a file called `peers.dat` — its address book of other nodes on the network. This test deliberately breaks that file in every conceivable way and makes sure Bitcoin Core handles each failure gracefully.

### Why This Matters for Developers

Every application that persists state to disk needs to survive a corrupted save file. Think about it: you've probably seen a video game that corrupted your save file, or a Word document that opened as garbled text. The difference is stakes. If your save file is corrupt, you lose a few hours of gameplay. If Bitcoin Core's address book is corrupt and the node doesn't notice, it might connect to the wrong peers, or worse — silently operate with bad data.

This test teaches a universal engineering principle: **validate everything you load from disk, fail loudly when something is wrong, and give the user a clear path to recovery.**

### The Technical Version

Bitcoin Core's Address Manager (`AddrMan`) is the subsystem responsible for discovering, tracking, and connecting to peer nodes on the Bitcoin network. It maintains two tables — a "new" table for recently discovered addresses and a "tried" table for addresses that have been successfully connected to. All of this state is persisted to a file called `peers.dat`. The `feature_addrman.py` test verifies that Bitcoin Core correctly validates `peers.dat` on startup, rejecting every possible form of corruption while gracefully handling version mismatches and missing files.

---

## What Is AddrMan and Why Does It Matter?

Before diving into the test, let's understand what we're protecting.

### The Two Tables

Think of AddrMan as a two-tier filing system:

- **The "New" table** is like your "Maybe" pile. These are addresses you've heard about but haven't actually connected to yet. Someone on the network told you about them, but you don't know if they're real or reliable. There are **1,024 buckets**, each holding up to **64 addresses** — a maximum of 65,536 slots.

- **The "Tried" table** is your "Trusted" list. These are addresses you've actually connected to and confirmed are real. There are **256 buckets**, each holding up to **64 addresses** — a maximum of 16,384 slots.

### The Secret Key (Bucket Key)

Here's where it gets interesting. Addresses aren't just dumped randomly into buckets. Bitcoin Core uses a **secret 256-bit key** (`nKey`) — generated randomly when AddrMan is first created — to determine which bucket each address goes into. The key is like a secret recipe: given the same key and the same address, you'll always get the same bucket. But without the key, you can't predict the placement.

Why does this matter? It's an **anti-manipulation defense**. If an attacker knew exactly which bucket an address would land in, they could flood that bucket with fake addresses to crowd out real ones (an attack called "poisoning"). The secret key makes the bucket placement unpredictable to outsiders.

### The File Format

`peers.dat` is not just a raw dump of addresses. It's a carefully structured file with:

1. **Network magic bytes** — 4 bytes that identify which Bitcoin network this file belongs to (mainnet, testnet, regtest, etc.)
2. **Version and compatibility bytes** — so that newer versions of Bitcoin Core can gracefully handle files from older (or newer) versions
3. **The secret bucket key** — the 256-bit `nKey` that determines bucket placement
4. **Address counts** — how many addresses are in the new and tried tables
5. **The actual address data** — every address with its metadata
6. **Bucket assignments** — which addresses are in which buckets
7. **A SHA-256d checksum** — a cryptographic "seal" over everything above, so any tampering is detected

---

## The Test: What Does It Do?

The test runs a single node and manipulates `peers.dat` in various ways before each startup, checking that Bitcoin Core responds correctly every time.

### Helper Functions (Lines 16-46)

```python
def serialize_addrman(
    *,
    format=1,
    lowest_compatible=4,
    net_magic="regtest",
    bucket_key=1,
    len_new=None,
    len_tried=None,
    mock_checksum=None,
):
```

This is the test's "forgery kit." It constructs a `peers.dat` file from scratch with controllable parameters. By default it creates a valid (but empty) file. By tweaking individual parameters, the test can introduce exactly one specific type of corruption at a time — like a surgeon making a precise incision.

`write_addrman` simply writes the output of `serialize_addrman` to the actual `peers.dat` path.

### Test 1: A Valid (Empty) AddrMan Loads Successfully (Lines 62-67)

```python
self.log.info("Check that mocked addrman is valid")
self.stop_node(0)
write_addrman(peers_dat)
with self.nodes[0].assert_debug_log(["Loaded 0 addresses from peers.dat"]):
    self.start_node(0, extra_args=["-checkaddrman=1"])
assert_equal(self.nodes[0].getnodeaddresses(), [])
```

**The anecdote:** Before you test all the ways a contact list can break, you first make sure a perfectly normal contact list loads fine. This is the "sanity check" — the baseline. If this fails, everything else is meaningless.

The `-checkaddrman=1` flag forces Bitcoin Core to run its full consistency check (`CheckAddrman`) on every operation, not just probabilistically. This is like turning on "paranoid mode" for testing.

### Test 2: Negative `lowest_compatible` Is Rejected (Lines 69-78)

```python
self.log.info("Check that addrman with negative lowest_compatible cannot be read")
write_addrman(peers_dat, lowest_compatible=-32)
self.nodes[0].assert_start_raises_init_error(...)
```

**The anecdote:** Imagine someone hands you a document and says "this was written for version -32 of our software." That doesn't make sense — there's no such thing as a negative version. It's like finding a receipt dated the year -500. The document is clearly nonsense.

The `lowest_compatible` field tells Bitcoin Core the *minimum* version that can read this file. After the `INCOMPATIBILITY_BASE` (32) is added, the value should never go below 32. A value of 0 (from -32 + 32) is below that floor, so the node refuses to start.

### Test 3: A File From the Future Gets Backed Up (Lines 80-89)

```python
self.log.info("Check that addrman from future is overwritten with new addrman")
write_addrman(peers_dat, lowest_compatible=111)
assert_equal(os.path.exists(peers_dat + ".bak"), False)
with self.nodes[0].assert_debug_log([
    'Creating new peers.dat because the file version was not compatible...'
]):
    self.start_node(0)
assert_equal(os.path.exists(peers_dat + ".bak"), True)
```

**The anecdote:** You download a file from a friend who has a newer version of the app. Your app can't read it. But instead of just deleting it (losing it forever), the app says: "I can't read this, but I'll rename it to `peers.dat.bak` so you don't lose it, and I'll start fresh." That's exactly what Bitcoin Core does here — graceful degradation with data preservation.

`lowest_compatible=111` means the file requires at least version 111 + 32 = 143 to read. Bitcoin Core's current version is far below that, so it can't read the file. Rather than crashing, it backs up the file and starts with an empty address book.

### Test 4: A Truncated File Is Rejected (Lines 91-98)

```python
self.log.info("Check that corrupt addrman cannot be read (EOF)")
with open(peers_dat, "wb") as f:
    f.write(serialize_addrman()[:-1])  # remove the last byte
self.nodes[0].assert_start_raises_init_error(
    expected_msg=init_error("AutoFile::read: end of file.*"),
)
```

**The anecdote:** Imagine downloading a movie and the download cuts off at 99.9%. You try to play it — it starts fine but then abruptly stops and crashes. That's exactly this: the file is valid up until it suddenly ends one byte too early. Bitcoin Core hits the end of the file while still expecting more data and refuses to continue.

### Test 5: Wrong Network Magic Is Rejected (Lines 100-106)

```python
self.log.info("Check that corrupt addrman cannot be read (magic)")
write_addrman(peers_dat, net_magic="signet")
self.nodes[0].assert_start_raises_init_error(
    expected_msg=init_error("Invalid network magic number"),
)
```

**The anecdote:** This is like putting a US passport into the Canadian border scanner. The document might be perfectly valid — but it's for the wrong country. The first 4 bytes of `peers.dat` are a "magic number" that identifies which Bitcoin network the file belongs to. If you're running a regtest node and someone drops in a signet `peers.dat`, the magic number won't match and the node rejects it immediately. No guessing, no "maybe it'll work" — instant rejection.

### Test 6: A Tampered Checksum Is Detected (Lines 108-114)

```python
self.log.info("Check that corrupt addrman cannot be read (checksum)")
write_addrman(peers_dat, mock_checksum=b"ab" * 32)
self.nodes[0].assert_start_raises_init_error(
    expected_msg=init_error("Checksum mismatch, data corrupted"),
)
```

**The anecdote:** Every time you order a product online, the package has a tracking number. When it arrives, the warehouse scans it and checks: does the number on the package match what we're expecting? If someone swapped out the package with a different one, the numbers won't match.

The checksum in `peers.dat` works the same way. Bitcoin Core computes a SHA-256d hash over the entire file content, then compares it to the checksum stored at the end of the file. Here, the test replaces the real checksum with `"ababab..."` — obviously wrong — and Bitcoin Core catches it instantly.

### Tests 7 & 8: Out-of-Range Address Counts Are Caught (Lines 116-147)

```python
# len_tried = -1 (negative)
write_addrman(peers_dat, len_tried=-1)
# len_tried = max + 1 (too large)
write_addrman(peers_dat, len_tried=max_len_tried + 1)
# Same for len_new
```

**The anecdote:** Imagine a restaurant's inventory system says it has -5 eggs, or 1,000,000 eggs. Both are obviously wrong. The header of `peers.dat` declares how many addresses are in each table. Bitcoin Core knows the physical limits: the tried table can hold at most `256 buckets × 64 slots = 16,384` addresses, and the new table can hold at most `1024 × 64 = 65,536`. Any count outside those bounds — negative or too large — is immediately flagged as corruption.

### Test 9: A Consistency Check Failure Is Caught (Lines 149-155)

```python
self.log.info("Check that corrupt addrman cannot be read (failed check)")
write_addrman(peers_dat, bucket_key=0)
self.nodes[0].assert_start_raises_init_error(
    expected_msg=init_error("Corrupt data. Consistency check failed with code -16: .*"),
)
```

**The anecdote:** Remember the secret key that determines which bucket each address lands in? Setting `bucket_key=0` (all zeros) is like using a "no key" on a lock — it's not a valid key. Bitcoin Core's consistency checker (`CheckAddrman`) re-derives every bucket placement using the key and compares it to what's actually in the file. A null key triggers error code `-16`, which specifically means "nKey is null." The check catches this before the node ever starts serving connections.

### Test 10: A Missing File Is Gracefully Recreated (Lines 157-159)

```python
self.log.info("Check that missing addrman is recreated")
self.restart_node(0, clear_addrman=True)
assert_equal(self.nodes[0].getnodeaddresses(), [])
```

**The anecdote:** You delete all your contacts. Your phone doesn't crash — it just starts with an empty contact list and you build it back up over time. Bitcoin Core does the same: if `peers.dat` is simply missing, it creates a fresh empty one and starts discovering peers from scratch using DNS seeds.

---

## The C++ Implementation: How It All Works Under the Hood

### 1. The File Envelope (`addrdb.cpp`)

`peers.dat` isn't just raw address data — it's wrapped in an envelope with integrity checks. The reading path in `DeserializeFileDB` (`addrdb.cpp`) does three things in order:

```cpp
// 1. Check network magic (first 4 bytes)
verifier >> pchMsgTmp;
if (pchMsgTmp != Params().MessageStart())
    throw std::runtime_error{"Invalid network magic number"};

// 2. Deserialize the AddrMan payload (calls Unserialize)
verifier >> data;

// 3. Verify the trailing checksum
stream >> hashTmp;
if (hashTmp != verifier.GetHash())
    throw std::runtime_error{"Checksum mismatch, data corrupted"};
```

The `HashVerifier` wrapper transparently hashes every byte it reads, so by the time we reach step 3, it has already computed the hash of the entire file content. The stored checksum is compared against this computed hash. If even a single bit was flipped, the checksums won't match.

### 2. Version Compatibility (`addrman.cpp` — Unserialize)

The first two bytes of the AddrMan payload are the version and compatibility fields:

```cpp
uint8_t uFormatVersion;
uint8_t uFormatCompatVersion;
s >> uFormatVersion >> uFormatCompatVersion;

if (uFormatCompatVersion < INCOMPATIBILITY_BASE)
    throw std::runtime_error{"Corrupted addrman database: The compat value is lower than expected..."};

if (uFormatCompatVersion > INCOMPATIBILITY_BASE + FILE_FORMAT)
    throw InvalidAddrManVersionError{...};  // caught specially: backup and recreate
```

Two very different outcomes for two very different problems:
- **Too old** (below `INCOMPATIBILITY_BASE`): the file is corrupt — hard error, node won't start.
- **Too new** (above current version): the file is from the future — soft error, back it up and start fresh.

This is a deliberate design choice. A file from the future *might* be valid — we just can't read it yet. A file with a corrupted version field is definitely broken.

### 3. The Consistency Check (`addrman.cpp` — `CheckAddrman`)

After the file is fully deserialized, Bitcoin Core runs an exhaustive audit of the entire address database. This function checks over 20 different invariants, including:

- **Every address in a bucket actually belongs there** — it re-derives the bucket placement using `nKey` and verifies it matches where the address was placed
- **Reference counts are correct** — each "new" address knows how many bucket slots point to it; this is recomputed and verified
- **No orphans** — every address in `mapInfo` must be reachable from at least one bucket
- **The key is not null** — `nKey` must be a real random value (error code `-16`)
- **Per-network counts match reality** — the counts for each network (IPv4, IPv6, Tor, etc.) are verified against the actual data

The bucket placement functions use the secret key like a cryptographic hash:

```cpp
// Which tried bucket does this address go in?
int AddrInfo::GetTriedBucket(const uint256& nKey, ...) const {
    uint64_t hash1 = Hash(nKey || address_key);
    uint64_t hash2 = Hash(nKey || address_group || (hash1 % 8));
    return hash2 % 256;  // one of 256 tried buckets
}
```

If someone tampers with `nKey` in the file, every single bucket placement will be wrong, and `CheckAddrman` will catch it.

### 4. Error Handling in `LoadAddrman` (`addrdb.cpp`)

The loading function has three distinct exception handlers, each with a different recovery strategy:

```cpp
try {
    DeserializeFileDB(path_addr, *addrman);
} catch (const DbNotFoundError&) {
    // File missing → start fresh (no error)
} catch (const InvalidAddrManVersionError&) {
    // File from the future → backup and start fresh (warning, no error)
} catch (const std::exception& e) {
    // Everything else → fatal, node does not start
}
```

This is a beautiful example of **graduated error handling** — different problems get different responses based on severity and recoverability.

---

## The Full Execution Flow

```
Test manipulates peers.dat
   ↓
Node starts up
   ↓
LoadAddrman() is called (addrdb.cpp)
   ↓
DeserializeFileDB() opens the file
   ↓
┌─ Check 1: Network magic bytes match? ──── No → "Invalid network magic number" (FAIL)
│
├─ Check 2: Deserialize payload
│   ├─ Version compatibility OK? ────────── Too old → "compat value too low" (FAIL)
│   │                                       Too new → InvalidAddrManVersionError (backup + fresh start)
│   ├─ nNew / nTried in valid range? ────── No → "nNew/nTried out of range" (FAIL)
│   └─ Full data deserializes without EOF? ─ No → "end of file" (FAIL)
│
├─ Check 3: Checksum matches? ──────────── No → "Checksum mismatch" (FAIL)
│
└─ Check 4: CheckAddrman() passes? ──────── No → "Consistency check failed" (FAIL)
                                            Yes → Node starts successfully
```

---

## Why This Test Matters

### 1. **Defense Against Disk Corruption**
Hard drives fail. SSDs have bit-flip errors. Flash drives corrupt data silently. Every one of the checks in this test maps to a real-world corruption scenario that could happen on any machine.

### 2. **Defense Against Attacks**
`peers.dat` is a high-value target. If an attacker can manipulate a node's address book, they can isolate it from the real network (an "eclipse attack"). The checksum, magic number, and consistency checks make this dramatically harder.

### 3. **Graceful Degradation**
Not every problem is fatal. A missing file? No big deal — start fresh. A file from a newer version? Back it up and start fresh. A corrupted file? That's serious — stop and tell the user. The test verifies that Bitcoin Core makes the *right* decision for each scenario.

### 4. **Clear Error Messages**
Each failure mode produces a specific, actionable error message that tells the user exactly what went wrong and what to do about it. The init error even includes the file path and suggests the user can rename or delete the file as a workaround.

---

## Real-World Scenarios

### Scenario 1: You Move Your Bitcoin Data to a New Hard Drive
You copy your Bitcoin Core data directory to a new drive, but the copy gets interrupted halfway through. `peers.dat` is truncated. Without this test's EOF check, Bitcoin Core might crash mysteriously or behave unpredictably. With it, you get a clear error message telling you exactly what happened.

### Scenario 2: You Restore From a Cloud Backup
Your backup software grabbed `peers.dat` from a machine running a newer version of Bitcoin Core. When you restore it on an older machine, the version check kicks in, saves the file as `.bak`, and starts fresh. No data loss, no confusion.

### Scenario 3: Ransomware or Malicious Software Tampers With Your Files
Malware modifies `peers.dat` to point your node to attacker-controlled addresses. The checksum check detects the tampering immediately and the node refuses to start — alerting you that something is very wrong.

### Scenario 4: Filesystem Bit Rot
Rarely, a single bit in a stored file flips due to cosmic radiation or electrical interference. This is not science fiction — it happens. The SHA-256d checksum catches even a single-bit flip, because the hash of the corrupted data will be completely different from the stored checksum.

---

## Key Takeaways

1. **`peers.dat` is Bitcoin Core's address book** — it stores all known peer addresses in two tables (new and tried), organized into buckets using a secret key

2. **The secret bucket key (`nKey`) is a security feature** — it makes bucket placement unpredictable to outsiders, defending against address-poisoning attacks

3. **The file has multiple layers of validation** — magic number, version compatibility, data range checks, checksum, and a full consistency audit

4. **Different errors get different responses** — missing file (recreate), future version (backup + recreate), corruption (refuse to start)

5. **The consistency check (`CheckAddrman`) is exhaustive** — it re-derives every bucket placement from scratch and verifies the entire database structure

6. **Clear error messages and recovery paths** — the user always knows what went wrong and how to fix it

---

## Conclusion

`feature_addrman.py` is a masterclass in defensive programming. It takes Bitcoin Core's most critical peer-discovery data structure and subjects it to every conceivable form of corruption, verifying that the node responds correctly every single time. The test reflects a deep engineering philosophy: **data you load from disk should never be trusted until it's been verified — and if verification fails, the failure should be loud, clear, and recoverable.**

For developers, this test is an excellent template for how to think about file loading and validation in any application. The graduated error handling — recreate, backup-and-recreate, or refuse-to-start — based on the severity of the problem, is a pattern worth emulating far beyond Bitcoin Core.

---
