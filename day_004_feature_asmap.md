# Day 4: Understanding feature_asmap.py - Smarter Address Grouping with ASN Mapping

**Test File:** `bitcoin/test/functional/feature_asmap.py`
**Category:** Feature Tests
**Complexity:** Moderate

---

## Introduction

### The Neighborhood Analogy

Imagine you're organizing a huge conference with 10,000 attendees from around the world. You need to split them into discussion groups, but here's the catch: you don't want any single organization or company to dominate multiple groups. If you group people by their email addresses randomly, you might accidentally put 50 employees from the same company in 50 different groups — giving that company disproportionate influence across the conference.

Here's a better approach: group people by their **company or organization**. All the Google employees go in one bucket, all the Microsoft employees in another, all the university researchers in another. This way, even if Google sends 1,000 people, they'll only have significant influence in a few groups, not everywhere.

Bitcoin Core faces the exact same problem with IP addresses. An attacker with access to a large network (like a datacenter or ISP) could generate thousands of IP addresses that look different but all come from the same place. The old method — grouping by `/16` prefix (the first 16 bits of an IP address) — is like grouping conference attendees by the first two letters of their email address. It's simple, but easily gamed.

**ASN mapping** (Autonomous System Number mapping) is the better approach. It's like grouping by company: all addresses that belong to the same network operator get grouped together, no matter how many individual IP addresses they control.

### Why This Matters for Developers

This test is about **Sybil attack resistance** — defending against an attacker who creates many fake identities to gain disproportionate influence. The same principle applies everywhere:

- **Social networks:** Detecting bot farms that create thousands of fake accounts from the same datacenter
- **Voting systems:** Preventing someone from stuffing the ballot box with sockpuppet accounts
- **Recommendation engines:** Ensuring one coordinated group can't manipulate ratings
- **Distributed systems:** Preventing a single entity from controlling too many nodes

The key insight: **physical network topology reveals hidden relationships between seemingly independent identities.** Two IP addresses that look completely different (say, `45.12.34.56` and `198.51.100.78`) might both belong to the same Amazon datacenter — and ASN mapping reveals that.

### The Technical Version

Bitcoin Core's Address Manager (`AddrMan`) organizes peer addresses into buckets to ensure diverse connections across the network. By default, it uses `/16` prefix bucketing (grouping IPv4 addresses by their first 16 bits), which provides basic diversity but can be circumvented by attackers with access to large IP ranges. **ASN mapping** is an optional enhancement that groups addresses by their Autonomous System Number (ASN) — the organization that controls the network — providing much stronger Sybil resistance. The `feature_asmap.py` test verifies that Bitcoin Core correctly loads, validates, and uses ASN map files for IP bucketing, and gracefully handles all error cases (missing files, unparsable files, empty files) without crashing.

---

## What Is an AS (Autonomous System)?

Before diving into the test, we need to understand the internet's organizational structure.

### The Internet's Hierarchical Structure

The internet isn't just a random mesh of computers. It's organized into **Autonomous Systems (AS)** — networks controlled by a single organization. Examples:

- **AS15169:** Google's network
- **AS16509:** Amazon Web Services (AWS)
- **AS174:** Cogent Communications (a major backbone provider)
- **AS13335:** Cloudflare

Each AS controls one or more blocks of IP addresses. When you connect to a website hosted on AWS, you're connecting to an IP address in AS16509. When you search on Google, you're connecting to AS15169.

### Why ASNs Matter for Bitcoin

An attacker who controls a datacenter or ISP has access to thousands (or millions) of IP addresses within the same AS. Using traditional `/16` bucketing, those addresses might appear diverse:

```
Traditional /16 bucketing:
203.0.113.1    → bucket for 203.0.x.x
198.51.100.50  → bucket for 198.51.x.x
192.0.2.200    → bucket for 192.0.x.x

These look different! Bitcoin Core would treat them as diverse peers.
```

But if all three belong to the same AS (say, a malicious ISP), they're actually controlled by the same entity:

```
ASN-based bucketing:
203.0.113.1    → AS64512 (Evil ISP)
198.51.100.50  → AS64512 (Evil ISP)
192.0.2.200    → AS64512 (Evil ISP)

These are NOT diverse! Bitcoin Core should group them together.
```

By bucketing by ASN instead of IP prefix, Bitcoin Core gains a much more accurate picture of network diversity.

### The Trade-off

There's a cost: ASN mapping requires an **external data file** that maps IP ranges to ASN numbers. This file:
- Must be generated from BGP routing data (which changes over time)
- Must be kept up to date (networks reorganize, ISPs merge, etc.)
- Adds complexity (file loading, validation, error handling)

That's why ASN mapping is **optional** — users can enable it with `-asmap=<file>` if they want the enhanced protection, or stick with the simpler `/16` bucketing if they don't.

---

## The Test: What Does It Do?

The test runs 9 independent scenarios, each verifying a different aspect of ASN map handling. Let's walk through them.

### Test Setup (Lines 41-44)

```python
def set_test_params(self):
    self.num_nodes = 1
    # Do addrman checks on all operations and use deterministic addrman
    self.extra_args = [["-checkaddrman=1", "-test=addrman"]]
```

The test uses a single node with:
- `-checkaddrman=1`: Run full AddrMan consistency checks on every operation (paranoid mode)
- `-test=addrman`: Use deterministic randomness for reproducible bucket placement

### Helper Function: `fill_addrman` (Lines 46-49)

```python
def fill_addrman(self, node_id):
    """Add 2 tried addresses to the addrman, followed by 2 new addresses."""
    for addr, tried in [[0, True], [1, True], [2, False], [3, False]]:
        self.nodes[node_id].addpeeraddress(address=f"101.{addr}.0.0", tried=tried, port=8333)
```

This populates the address manager with 4 test addresses:
- `101.0.0.0` and `101.1.0.0` marked as "tried" (we've successfully connected before)
- `101.2.0.0` and `101.3.0.0` marked as "new" (heard about, but never connected)

This setup is used in Test 5 to verify that ASN mapping works correctly with an existing address database.

---

## Test 1: Default Behavior Without `-asmap` (Lines 51-55)

```python
def test_without_asmap_arg(self):
    self.log.info('Test bitcoind with no -asmap arg passed')
    self.stop_node(0)
    with self.node.assert_debug_log(['Using /16 prefix for IP bucketing']):
        self.start_node(0)
```

**The anecdote:** You buy a new phone and it comes with default settings. The manufacturer chose sensible defaults that work for 95% of users. You don't have to configure anything — it just works.

Bitcoin Core's default is `/16` prefix bucketing. It's simple, requires no external files, and works reasonably well. The test confirms that without the `-asmap` flag, the node logs `"Using /16 prefix for IP bucketing"` and starts successfully.

**What this tests:** The baseline. Before testing all the ASN map scenarios, confirm that the default behavior still works.

---

## Test 2: Explicit `-noasmap` Flag (Lines 57-61)

```python
def test_noasmap_arg(self):
    self.log.info('Test bitcoind with -noasmap arg passed')
    self.stop_node(0)
    with self.node.assert_debug_log(['Using /16 prefix for IP bucketing']):
        self.start_node(0, ["-noasmap"])
```

**The anecdote:** Sometimes you want to explicitly say "no thanks" to a feature. Like when a website asks if you want push notifications, and you click "Block" instead of ignoring the popup.

The `-noasmap` flag explicitly disables ASN mapping, even if a default ASN map file exists. This is useful if:
- You previously used `-asmap` but want to temporarily disable it
- You have an `ip_asn.map` file in your data directory but don't want to use it

**What this tests:** That the "opt-out" mechanism works — users can explicitly disable ASN mapping.

---

## Test 3: ASMap with Absolute Path (Lines 63-70)

```python
def test_asmap_with_absolute_path(self):
    self.log.info('Test bitcoind -asmap=<absolute path>')
    self.stop_node(0)
    filename = os.path.join(self.datadir, 'my-map-file.map')
    shutil.copyfile(self.asmap_raw, filename)
    with self.node.assert_debug_log(expected_messages(filename)):
        self.start_node(0, [f'-asmap={filename}'])
    os.remove(filename)
```

**The anecdote:** You want to play a video stored in `/Users/Alice/Videos/movie.mp4`. You tell your video player the full path. It goes directly to that exact location and opens the file.

This test copies the skeleton ASN map file to a custom location and tells Bitcoin Core the absolute path: `-asmap=/full/path/to/my-map-file.map`.

**Expected output:**
```
Opened asmap file "/full/path/to/my-map-file.map" (59 bytes) from disk
Using asmap version fec61fa21a9f46f3b17bdcd660d7f4cd90b966aad3aec593c99b35f0aca15853 for IP bucketing
```

The version hash (`fec61fa2...`) is a SHA-256 hash of the entire ASN map contents. Two nodes with the same hash are guaranteed to bucket addresses identically.

**What this tests:** That Bitcoin Core can load an ASN map from any location on the filesystem.

---

## Test 4: ASMap with Relative Path (Lines 72-80)

```python
def test_asmap_with_relative_path(self):
    self.log.info('Test bitcoind -asmap=<relative path>')
    self.stop_node(0)
    name = 'ASN_map'
    filename = os.path.join(self.datadir, name)
    shutil.copyfile(self.asmap_raw, filename)
    with self.node.assert_debug_log(expected_messages(filename)):
        self.start_node(0, [f'-asmap={name}'])
    os.remove(filename)
```

**The anecdote:** Instead of saying "go to `/Users/Alice/Videos/movie.mp4`," you say "go to `Videos/movie.mp4`" from your home directory. The application figures out the full path by combining your home directory with the relative path.

This test passes just the filename (`ASN_map`) without a path. Bitcoin Core should:
1. Realize it's a relative path (no leading `/`)
2. Prefix it with the network-specific data directory (e.g., `~/.bitcoin/regtest/`)
3. Load the file from there

**What this tests:** That relative paths are resolved correctly relative to the data directory, not the current working directory.

---

## Test 5: Default ASMap File (Lines 82-89)

```python
def test_default_asmap(self):
    shutil.copyfile(self.asmap_raw, self.default_asmap)
    for arg in ['-asmap', '-asmap=']:
        self.log.info(f'Test bitcoind {arg} (using default map file)')
        self.stop_node(0)
        with self.node.assert_debug_log(expected_messages(self.default_asmap)):
            self.start_node(0, [arg])
    os.remove(self.default_asmap)
```

**The anecdote:** Your web browser has a default download folder. When you click "Download" without specifying where to save, it goes there automatically.

`DEFAULT_ASMAP_FILENAME` is defined as `'ip_asn.map'` (line 32 of the test). If you pass `-asmap` or `-asmap=` (with no file specified), Bitcoin Core looks for this file in the data directory.

The test verifies that both flag formats work:
- `-asmap` (no `=`)
- `-asmap=` (with `=` but no value)

**What this tests:** The convenience feature — users can just say `-asmap` and Bitcoin Core knows where to look.

---

## Test 6: ASMap Interaction with Existing AddrMan (Lines 91-105)

```python
def test_asmap_interaction_with_addrman_containing_entries(self):
    self.log.info("Test bitcoind -asmap restart with addrman containing new and tried entries")
    self.stop_node(0)
    shutil.copyfile(self.asmap_raw, self.default_asmap)
    self.start_node(0, ["-asmap", "-checkaddrman=1", "-test=addrman"])
    self.fill_addrman(node_id=0)
    self.restart_node(0, ["-asmap", "-checkaddrman=1", "-test=addrman"])
    with self.node.assert_debug_log(
        expected_msgs=[
            "CheckAddrman: new 2, tried 2, total 4 started",
            "CheckAddrman: completed",
        ]
    ):
        self.node.getnodeaddresses()  # getnodeaddresses re-runs the addrman checks
    os.remove(self.default_asmap)
```

**The anecdote:** You reorganize your music library by genre (Rock, Jazz, Classical). But you already have 10,000 songs organized by artist. You run a script to recategorize them all. At the end, you verify: "Do I still have all 10,000 songs? Did any get lost or duplicated?"

This is the most important test. It verifies that:
1. You can start Bitcoin Core with `-asmap`, add addresses, restart, and everything still works
2. The consistency check (`CheckAddrman`) passes after loading addresses that were bucketed using ASN mapping
3. The address counts match: 2 new, 2 tried, 4 total

**Why this matters:** When you switch from `/16` bucketing to ASN bucketing (or vice versa), addresses need to be re-bucketed. Bitcoin Core must handle this gracefully without losing addresses or corrupting the address database.

**What this tests:** That ASN mapping is compatible with AddrMan's persistence and consistency checks.

---

## Test 7: Missing Default ASMap File (Lines 107-111)

```python
def test_default_asmap_with_missing_file(self):
    self.log.info('Test bitcoind -asmap with missing default map file')
    self.stop_node(0)
    msg = f"Error: Could not find asmap file \"{self.default_asmap}\""
    self.node.assert_start_raises_init_error(extra_args=['-asmap'], expected_msg=msg)
```

**The anecdote:** You set your music player to use a specific equalizer preset file, but you forgot to download it. The player should say "Error: Can't find preset file" instead of crashing or using random settings.

If you explicitly request `-asmap` but the file doesn't exist, Bitcoin Core should **refuse to start** with a clear error message. This is the right choice because:
- You explicitly enabled a security feature
- Without the file, Bitcoin Core would silently fall back to weaker `/16` bucketing
- That's a security downgrade you didn't consent to

**What this tests:** That missing ASN map files are treated as a fatal error when ASN mapping is explicitly requested.

---

## Test 8: Empty (Unparsable) ASMap File (Lines 113-120)

```python
def test_empty_asmap(self):
    self.log.info('Test bitcoind -asmap with empty map file')
    self.stop_node(0)
    with open(self.default_asmap, "w", encoding="utf-8") as f:
        f.write("")
    msg = f"Error: Could not parse asmap file \"{self.default_asmap}\""
    self.node.assert_start_raises_init_error(extra_args=['-asmap'], expected_msg=msg)
    os.remove(self.default_asmap)
```

**The anecdote:** Someone emails you a corrupted ZIP file. When you try to extract it, your computer says "This archive is damaged and can't be opened" instead of extracting garbage.

An empty file fails the sanity check in `DecodeAsmap` (src/util/asmap.cpp:217):

```cpp
if (!SanityCheckASMap(bits, 128)) {
    LogPrintf("Sanity check of asmap file %s failed\n", ...);
    return {};
}
```

The sanity check verifies that:
- The ASN map is a valid state machine (no infinite loops, no unreachable code)
- Every possible 128-bit IP address has a defined mapping
- The file ends with a `RETURN` instruction

An empty file obviously fails these checks.

**What this tests:** That corrupted or invalid ASN map files are detected before they can cause problems.

---

## Test 9: ASMap Health Check (Lines 122-136)

```python
def test_asmap_health_check(self):
    self.log.info('Test bitcoind -asmap logs ASMap Health Check with basic stats')
    shutil.copyfile(self.asmap_raw, self.default_asmap)
    msg = "ASMap Health Check: 4 clearnet peers are mapped to 3 ASNs with 0 peers being unmapped"
    with self.node.assert_debug_log(expected_msgs=[msg]):
        self.start_node(0, extra_args=['-asmap'])
    raw_addrman = self.node.getrawaddrman()
    asns = []
    for _, entries in raw_addrman.items():
        for _, entry in entries.items():
            asn = entry['mapped_as']
            if asn not in asns:
                asns.append(asn)
    assert_equal(len(asns), 3)
    os.remove(self.default_asmap)
```

**The anecdote:** After reorganizing your music library, your player shows: "10,000 songs organized into 42 genres, 3 songs couldn't be categorized." That's useful feedback — you know the system is working and can see if something went wrong.

This test verifies that Bitcoin Core logs a health check summary when using ASN mapping:

```
ASMap Health Check: 4 clearnet peers are mapped to 3 ASNs with 0 peers being unmapped
```

This tells the user:
- **4 clearnet peers:** Total IPv4/IPv6 addresses (excludes Tor, I2P, etc.)
- **3 ASNs:** These 4 addresses belong to 3 different network operators
- **0 unmapped:** All addresses were successfully mapped

**Why this is useful:** If you see "100 clearnet peers mapped to 2 ASNs," that's a red flag — you're not as diversified as you thought. If you see "50 unmapped," your ASN map might be outdated.

**What this tests:** That the health check correctly counts and reports ASN diversity.

---

## The C++ Implementation: How ASN Mapping Works

### 1. The ASN Map File Format (`util/asmap.cpp`)

The ASN map is not a simple lookup table. It's a **compressed binary decision tree** — a tiny program that takes a 128-bit IP address and returns a 32-bit ASN.

**The instruction set:**
```cpp
enum class Instruction : uint32_t
{
    RETURN = 0,   // Return an ASN number
    JUMP = 1,     // Conditional jump based on the next IP bit
    MATCH = 2,    // Match several IP bits at once
    DEFAULT = 3,  // Set default ASN for this path
};
```

**How it works:**
1. Start at the beginning of the ASN map
2. Read the current instruction
3. If `RETURN`: return the ASN and stop
4. If `JUMP`: check the next bit of the IP address. If 1, jump forward. If 0, continue.
5. If `MATCH`: check the next N bits of the IP address against a pattern. If they match, continue. If not, return the default ASN.
6. If `DEFAULT`: set the default ASN for this path
7. Repeat until you hit a `RETURN`

**Example traversal for IP `203.0.113.5` (in AS64496):**
```
Start → JUMP → bit 0 = 1 → jump forward
      → MATCH → bits 1-8 = 11001011 → match!
      → JUMP → bit 9 = 0 → continue
      → RETURN AS64496
```

This encoding is incredibly space-efficient. The entire map for all 4 billion IPv4 addresses can fit in a few megabytes.

### 2. Sanity Checking the ASN Map (`util/asmap.cpp:133-195`)

Before using an ASN map, Bitcoin Core runs an exhaustive sanity check:

```cpp
bool SanityCheckASMap(const std::vector<bool>& asmap, int bits)
{
    // Verify:
    // - Every path through the decision tree ends with RETURN
    // - No instructions straddle the end of the file
    // - No infinite loops
    // - No unreachable code
    // - No excessive padding at the end
    // - All jumps point to valid locations
    // ...
}
```

**Why this is critical:** A malicious or corrupted ASN map could:
- Cause infinite loops (node hangs)
- Return invalid ASNs (breaks bucketing)
- Leak information (timing attacks)

The sanity check prevents all of these by verifying the map is well-formed before any IP addresses are processed.

### 3. Loading the ASN Map (`init.cpp:1514-1538`)

When Bitcoin Core starts, it loads the ASN map (if configured):

```cpp
std::vector<bool> asmap;
if (args.IsArgSet("-asmap") && !args.IsArgNegated("-asmap")) {
    fs::path asmap_path = args.GetPathArg("-asmap", DEFAULT_ASMAP_FILENAME);
    if (!asmap_path.is_absolute()) {
        asmap_path = args.GetDataDirNet() / asmap_path;  // Make relative paths absolute
    }
    if (!fs::exists(asmap_path)) {
        InitError(strprintf(_("Could not find asmap file %s"), ...));
        return false;  // Fatal error
    }
    asmap = DecodeAsmap(asmap_path);
    if (asmap.size() == 0) {
        InitError(strprintf(_("Could not parse asmap file %s"), ...));
        return false;  // Fatal error
    }
    const uint256 asmap_version = (HashWriter{} << asmap).GetHash();
    LogInfo("Using asmap version %s for IP bucketing", asmap_version.ToString());
} else {
    LogInfo("Using /16 prefix for IP bucketing");
}
```

**Key steps:**
1. Check if `-asmap` was specified
2. Resolve relative paths to absolute paths
3. Check if the file exists (error if not)
4. Decode the file (error if parsing fails)
5. Compute a version hash (so users can verify they're using the same map)
6. Pass the map to `NetGroupManager`

### 4. Using the ASN Map for Bucketing (`netgroup.cpp:18-79`)

Once loaded, the ASN map is used by `NetGroupManager::GetGroup()` to determine which bucket an address goes into:

```cpp
std::vector<unsigned char> NetGroupManager::GetGroup(const CNetAddr& address) const
{
    std::vector<unsigned char> vchRet;

    // Try to get ASN for this address
    uint32_t asn = GetMappedAS(address);
    if (asn != 0) {  // Successfully mapped to an ASN
        vchRet.push_back(NET_IPV6);  // Use IPv6 network class
        for (int i = 0; i < 4; i++) {
            vchRet.push_back((asn >> (8 * i)) & 0xFF);  // Push 4 bytes of ASN
        }
        return vchRet;
    }

    // Fallback to /16 or /32 bucketing
    // ...
}
```

**The key insight:** If ASN mapping succeeds, the returned "group" is just the ASN number. Two addresses in the same AS will always return the same group, regardless of how different their IP addresses look.

### 5. Looking Up an Address (`netgroup.cpp:81-112`)

```cpp
uint32_t NetGroupManager::GetMappedAS(const CNetAddr& address) const
{
    if (m_asmap.size() == 0 || (net_class != NET_IPV4 && net_class != NET_IPV6)) {
        return 0;  // No ASN map loaded, or address type not supported
    }

    // Convert address to 128-bit representation
    std::vector<bool> ip_bits(128);
    // ... convert IP to bit vector ...

    // Interpret the ASN map as a program
    uint32_t mapped_as = Interpret(m_asmap, ip_bits);
    return mapped_as;
}
```

`Interpret` is the virtual machine that executes the ASN map program (described earlier).

---

## The Full Execution Flow

```
User starts Bitcoin Core with -asmap=ip_asn.map
  ↓
init.cpp: Load and parse ASN map file
  ↓
SanityCheckASMap: Verify the map is well-formed
  ↓
Compute asmap_version hash
  ↓
Log: "Using asmap version <hash> for IP bucketing"
  ↓
Create NetGroupManager with the ASN map
  ↓
Create AddrMan with the NetGroupManager
  ↓
(Later) Peer address 203.0.113.5 is added
  ↓
AddrMan calls NetGroupManager::GetGroup(203.0.113.5)
  ↓
NetGroupManager::GetMappedAS converts IP to bit vector
  ↓
Interpret executes the ASN map program on the IP bits
  ↓
ASN 64496 is returned
  ↓
AddrMan buckets the address using AS64496 as the group identifier
```

---

## Why This Test Matters

### 1. **Sybil Attack Resistance**

Without ASN mapping, an attacker with access to multiple `/16` ranges can pollute many buckets:

```
Attacker controls:
10.0.x.x   (65,536 IPs) → bucket group "10.0"
10.1.x.x   (65,536 IPs) → bucket group "10.1"
10.2.x.x   (65,536 IPs) → bucket group "10.2"
...

That's 256 different bucket groups from a single /8!
```

With ASN mapping:

```
Attacker controls AS64512:
10.0.0.1   → AS64512
10.1.0.1   → AS64512
10.2.0.1   → AS64512
...

All map to the same bucket group: AS64512
```

The attacker's ability to pollute multiple buckets is dramatically reduced.

### 2. **Optional Security Enhancement**

Not everyone needs ASN mapping. Home users with a few connections? `/16` bucketing is fine. Nodes in adversarial environments (exchanges, mining pools, high-value targets)? ASN mapping provides crucial additional protection.

Making it optional respects both use cases.

### 3. **Graceful Error Handling**

All three error scenarios (missing file, empty file, unparsable file) produce clear error messages and refuse to start. This is the right choice — if the user requested ASN mapping, silently downgrading to `/16` bucketing would be a security failure.

### 4. **Versioning and Auditability**

The ASN map version hash (logged at startup) allows users to:
- Verify they're using an official map (compare hash with published values)
- Debug peer connection issues ("Did we update the map recently?")
- Coordinate with other operators ("Are we both using the same ASN map?")

---

## Real-World Scenarios

### Scenario 1: Eclipse Attack on an Exchange

An attacker wants to eclipse (isolate) a Bitcoin exchange to double-spend against them. The attacker:
1. Rents 10,000 IP addresses across 50 different `/16` ranges from various cloud providers
2. Connects to the exchange's node from all 10,000 IPs
3. Uses various tricks to get those IPs into the exchange's AddrMan
4. Eventually, the exchange restarts and primarily connects to the attacker's IPs

**With ASN mapping:** The exchange would see that all 10,000 IPs belong to just 3 or 4 cloud providers (Amazon, Google, DigitalOcean). AddrMan would deprioritize them, ensuring the exchange maintains diverse connections.

### Scenario 2: Running a Node Behind a Datacenter NAT

You run a Bitcoin Core node in a datacenter. Most of your peers are also in datacenters (because that's where many nodes are hosted). Without ASN mapping, your node might think it has 50 diverse peers, when in reality 40 of them are in the same Amazon region.

**With ASN mapping:** Your node realizes that most of its peers are in AS16509 (Amazon) and actively seeks connections to peers in other ASNs, improving network diversity.

### Scenario 3: Monitoring Network Health

You're a researcher studying Bitcoin's network topology. You want to know: "How centralized is the network? Are most nodes concentrated in a few ASNs?"

**With ASN mapping and the health check:** You can easily see that your node's 100 peers are spread across only 15 ASNs, with 60% in just 3 ASNs. That's valuable data about network centralization.

---

## Key Takeaways

1. **ASN mapping is a Sybil resistance mechanism** — it groups IP addresses by the organization that controls them, not just by IP prefix

2. **The ASN map is a compressed decision tree** — a tiny program that maps 128-bit IPs to 32-bit ASNs in just a few megabytes

3. **ASN mapping is optional** — users can enable it with `-asmap` or stick with simpler `/16` bucketing

4. **Error handling is strict** — missing or corrupted ASN maps are fatal errors when explicitly requested, preventing silent security downgrades

5. **The version hash enables coordination** — nodes can verify they're using the same ASN map by comparing hashes

6. **The health check provides visibility** — users can see how diverse their peer connections really are

7. **The test is comprehensive** — it covers all configuration methods (absolute path, relative path, default file), all error cases (missing, empty, unparsable), and the interaction with AddrMan persistence

---

## Conclusion

The `feature_asmap.py` test demonstrates Bitcoin Core's commitment to Sybil attack resistance. By making ASN mapping optional, well-tested, and error-resistant, Bitcoin Core gives users a powerful tool to defend against sophisticated attackers who control large IP ranges.

The test reflects a mature understanding of operational security:
- **Make security features opt-in** (don't break simple deployments)
- **Fail loudly when security features are requested but can't be delivered** (don't silently downgrade)
- **Provide visibility into how the feature is working** (health checks and version hashes)
- **Handle all error cases gracefully** (missing files, corrupted files, wrong paths)

For developers building any distributed system, the lessons here are universal: **network topology reveals hidden relationships between seemingly independent identities.** Whether you're defending against bot farms, sockpuppets, or Sybil attacks, understanding the physical or organizational structure beneath the logical identities is crucial.

---

## References

### Bitcoin Core Source Code
- **This test on GitHub:** https://github.com/bitcoin/bitcoin/blob/master/test/functional/feature_asmap.py - The test file analyzed in this essay
- **ASMap utility functions:** `src/util/asmap.h` and `src/util/asmap.cpp` - Core ASN map interpretation and validation logic
  - https://github.com/bitcoin/bitcoin/blob/master/src/util/asmap.h
  - https://github.com/bitcoin/bitcoin/blob/master/src/util/asmap.cpp
- **Network group management:** `src/netgroup.h` and `src/netgroup.cpp` - Integration of ASN mapping with peer bucketing
  - https://github.com/bitcoin/bitcoin/blob/master/src/netgroup.h
  - https://github.com/bitcoin/bitcoin/blob/master/src/netgroup.cpp
- **Initialization logic:** `src/init.cpp` (lines 1514-1538) - ASN map loading and error handling
  - https://github.com/bitcoin/bitcoin/blob/master/src/init.cpp#L1514-L1538
- **Address manager:** `src/addrman.h` and `src/addrman.cpp` - Uses network groups for peer bucketing
  - https://github.com/bitcoin/bitcoin/blob/master/src/addrman.h
  - https://github.com/bitcoin/bitcoin/blob/master/src/addrman.cpp
- **Test data:** `src/test/data/asmap.raw` - 59-byte skeleton ASN map used in tests
  - https://github.com/bitcoin/bitcoin/blob/master/src/test/data/asmap.raw

### Bitcoin Improvement Proposals
- **BIP 155 (addrv2):** https://github.com/bitcoin/bips/blob/master/bip-0155.mediawiki - Extended address format supporting Tor v3 and future address types

### Research Papers and Documentation
- **Eclipse Attacks on Bitcoin's Peer-to-Peer Network:** Heilman et al. (2015) - Foundational research on eclipse attacks that motivates ASN-based bucketing
  - Paper: https://www.usenix.org/node/190891
  - Shows how attackers with control over IP ranges can isolate Bitcoin nodes

- **Erebus Attack:** Tran et al. (2020) - Demonstrates large-scale eclipse attacks via BGP hijacking, further motivating ASN awareness
  - Paper: https://www.usenix.org/conference/usenixsecurity20/presentation/tran

### Internet Standards
- **RFC 7607:** Codification of BGP Operations - Defines AS0 as reserved
  - https://www.rfc-editor.org/rfc/rfc7607.html

- **Autonomous System Numbers:** IANA maintains the registry of ASN allocations
  - https://www.iana.org/assignments/as-numbers/as-numbers.xhtml

### BGP and ASN Mapping Resources
- **CAIDA ASRank:** Academic project ranking Autonomous Systems by connectivity
  - https://asrank.caida.org/

- **Team Cymru IP to ASN Mapping:** Historical service providing IP-to-ASN data
  - https://www.team-cymru.com/ip-asn-mapping

- **RIPEstat:** Real-time ASN and routing data from RIPE NCC
  - https://stat.ripe.net/

### Bitcoin Core Developer Resources
- **Default ASN map filename constant:** `DEFAULT_ASMAP_FILENAME = 'ip_asn.map'` defined in `src/init.cpp`

- **Command-line argument:** `-asmap=<file>` - Documented in `bitcoind -help` and `src/init.cpp:535`

- **ASN map generation tools:** Community-maintained scripts for generating ASN maps from BGP data
  - https://github.com/bitcoin/bitcoin/tree/master/contrib/asmap

### Related Bitcoin Core Tests
- **Unit tests:** `src/test/addrman_tests.cpp` - Tests AddrMan bucketing behavior with and without ASN mapping
- **Fuzz tests:** `src/test/fuzz/asmap.cpp` and `src/test/fuzz/asmap_direct.cpp` - Fuzzing coverage for ASN map parsing
- **Integration test:** This test (`feature_asmap.py`) - End-to-end functional testing

---

**Next Test:** feature_assumeutxo.py - Testing UTXO snapshot bootstrapping
