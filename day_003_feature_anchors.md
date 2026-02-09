# Day 3: Understanding feature_anchors.py - Bitcoin Core's Emergency Contacts

**Test File:** `bitcoin/test/functional/feature_anchors.py`
**Category:** Feature Tests
**Complexity:** Moderate

---

## Introduction

### The Emergency Contact Analogy

Imagine your phone crashes and reboots. When it comes back up, it needs to reconnect to your mobile network. But here's the problem — there are thousands of cell towers out there, and most of them might be unreliable or even malicious. So your phone has a clever trick: before it shut down last time, it saved the addresses of the two most reliable towers it was connected to. When it reboots, it tries those first.

Bitcoin Core does exactly the same thing. When a Bitcoin node shuts down gracefully, it saves a small list of "anchor" peers — specifically, the peers it had **block-relay-only connections** with. When it starts back up, it immediately tries to reconnect to those anchors before doing anything else.

Why only block-relay-only connections? Because those are the most trustworthy. They don't gossip about transactions or send unsolicited data. They just relay blocks — the bare minimum needed to stay in sync with the network. If you're going to trust any peers on restart, you trust those.

### Why This Matters for Developers

This is a masterclass in **graceful degradation** and **bootstrap security**. Consider the alternatives:

1. **No anchors at all** — Every time your node restarts, it starts from scratch with DNS seeds or hardcoded addresses. This is slow and makes you vulnerable during those critical first moments of reconnection.

2. **Save all peers** — This bloats your anchor file and defeats the purpose. You want a small, curated list of the best connections, not every random peer you've ever met.

3. **Save inbound connections** — Bad idea. Inbound connections are initiated by others, which means an attacker can flood your node with connections just before shutdown, pollute your anchor file, and then be your first connections on restart. Block-relay-only outbound connections are initiated by you, which makes them harder to manipulate.

This test teaches a principle that applies everywhere in distributed systems: **when bootstrapping, start with your most trustworthy connections first, and keep that list small and curated.**

### The Technical Version

Bitcoin Core's anchor mechanism persists a subset of peer addresses to `anchors.dat` on graceful shutdown. On startup, it attempts to reconnect to these anchors before initiating normal peer discovery. The test verifies that:
1. Only block-relay-only outbound connections are saved to `anchors.dat`
2. Corrupted anchor files don't crash the node
3. The file supports both IPv4 and addrv2 formats (including Tor v3 addresses)
4. The node correctly validates and attempts reconnection to anchors on restart

---

## What Are Anchors and Why Do They Matter?

Before diving into the test, let's understand the problem anchors solve.

### The Cold Start Problem

When a Bitcoin node starts up, it needs to connect to peers. But where does it find them? There are a few options:

1. **DNS seeds** — Hardcoded DNS names that return lists of Bitcoin nodes. This works, but it's slow and relies on external infrastructure.

2. **Hardcoded IPs** — Some Bitcoin implementations have fallback IP addresses baked into the code. This is brittle and can become outdated quickly.

3. **Peer address database** — Bitcoin Core maintains `peers.dat`, which stores thousands of addresses it's learned about over time. But when you first start up, you don't know which of these are currently online or reliable.

4. **Anchors** — A small list (typically 2) of the peers you were most recently connected to via block-relay-only connections. These are likely still online, trustworthy, and fast.

Anchors are your "emergency contacts." They're not a replacement for peer discovery — they're a head start.

### What Makes a Good Anchor?

Not all connections are created equal. Here's why Bitcoin Core specifically saves **block-relay-only** connections:

**Block-relay-only connections:**
- Don't relay transactions (less attack surface)
- Are initiated by you (harder for attackers to pollute)
- Are typically long-lived and stable
- Serve one purpose: keep you in sync with the blockchain

**Inbound connections:**
- Are initiated by others (attackers can flood you)
- Might be short-lived or opportunistic
- Could be adversarial (eclipse attack risk)

**Regular outbound connections:**
- Relay transactions and addresses (more chatty)
- More likely to churn (disconnect/reconnect frequently)

By saving only block-relay-only peers, Bitcoin Core ensures that your anchors are the most reliable, least manipulable subset of your peer set.

---

## The Test Structure

The test runs through three major scenarios:

### Part 1: Basic Anchor Persistence (Lines 26-75)
**What it tests:** Do only block-relay-only connections get saved to `anchors.dat`?

### Part 2: Corruption Handling (Lines 77-88)
**What it tests:** Does a corrupted anchor file crash the node, or is it handled gracefully?

### Part 3: Addrv2 Support (Lines 90-140)
**What it tests:** Can the anchor file store modern address formats, including 256-bit Tor v3 addresses?

Let's walk through each.

---

## Part 1: Basic Anchor Persistence

### The Setup (Lines 26-44)

```python
node_anchors_path = self.nodes[0].chain_path / "anchors.dat"

self.log.info("When node starts, check if anchors.dat doesn't exist")
assert not os.path.exists(node_anchors_path)
```

The test confirms that on a fresh start, there's no `anchors.dat` file. This makes sense — you can't have anchors if you've never shut down before.

Next, the test adds two types of connections:

```python
self.log.info(f"Add {BLOCK_RELAY_CONNECTIONS} block-relay-only connections to node")
for i in range(BLOCK_RELAY_CONNECTIONS):
    self.nodes[0].add_outbound_p2p_connection(
        P2PInterface(), p2p_idx=i, connection_type="block-relay-only"
    )

self.log.info(f"Add {INBOUND_CONNECTIONS} inbound connections to node")
for i in range(INBOUND_CONNECTIONS):
    self.nodes[0].add_p2p_connection(P2PInterface())
```

Now the node has:
- **2 block-relay-only outbound connections** (the anchors-to-be)
- **5 inbound connections** (should NOT be saved as anchors)

The test confirms the connection counts:

```python
check_node_connections(node=self.nodes[0], num_in=5, num_out=2)
```

### Extracting Peer Information (Lines 46-58)

The test uses a clever trick to identify which peers are which. Since all connections in the test environment use `127.0.0.1`, the test distinguishes peers by their port numbers:

```python
ip = "7f000001"  # 127.0.0.1 in hex

block_relay_nodes_port = []
inbound_nodes_port = []
for p in self.nodes[0].getpeerinfo():
    addr_split = p["addr"].split(":")
    if p["connection_type"] == "block-relay-only":
        block_relay_nodes_port.append(hex(int(addr_split[1]))[2:])
    else:
        inbound_nodes_port.append(hex(int(addr_split[1]))[2:])
```

This builds two lists:
- `block_relay_nodes_port`: ports of the 2 block-relay-only connections
- `inbound_nodes_port`: ports of the 5 inbound connections

### The Shutdown and Verification (Lines 60-75)

```python
self.log.debug("Stop node")
self.stop_node(0)

# It should contain only the block-relay-only addresses
self.log.info("Check the addresses in anchors.dat")

with open(node_anchors_path, "rb") as file_handler:
    anchors = file_handler.read()

anchors_hex = anchors.hex()
for port in block_relay_nodes_port:
    ip_port = ip + port
    assert ip_port in anchors_hex
for port in inbound_nodes_port:
    ip_port = ip + port
    assert ip_port not in anchors_hex
```

**Key assertion:** The anchor file contains the 2 block-relay-only connections and NONE of the 5 inbound connections.

This confirms the core behavior: **anchors are exclusively block-relay-only peers.**

---

## Part 2: Corruption Handling

### The Perturbation (Lines 77-88)

```python
self.log.info("Perturb anchors.dat to test it doesn't throw an error during initialization")
with self.nodes[0].assert_debug_log(["0 block-relay-only anchors will be tried for connections."]):
    with open(node_anchors_path, "wb") as out_file_handler:
        tweaked_contents = bytearray(anchors)
        tweaked_contents[20:20] = b'1'
        out_file_handler.write(bytes(tweaked_contents))

    self.log.debug("Start node")
    self.start_node(0)
```

**What's happening here:**
1. The test reads the valid `anchors.dat` file
2. It inserts a single byte (`b'1'`) at position 20, corrupting the structure
3. It restarts the node

**Expected behavior:**
- The node should NOT crash
- The node should log: `"0 block-relay-only anchors will be tried for connections."`
- The node should delete the corrupted file

```python
self.log.info("When node starts, check if anchors.dat doesn't exist anymore")
assert not os.path.exists(node_anchors_path)
```

**Why this matters:** In production, `anchors.dat` can become corrupted due to disk errors, unexpected shutdowns, or filesystem issues. The node must detect this and recover gracefully rather than crashing on startup.

**The recovery strategy:**
1. Try to parse `anchors.dat`
2. If parsing fails, log a warning
3. Delete the corrupted file
4. Continue with normal peer discovery (no anchors this time)

This is the same philosophy we saw in `feature_addrman.py` — **fail loudly, recover gracefully, and never let corrupted data take down the node.**

---

## Part 3: Addrv2 Support for Tor v3

### Why Addrv2?

Bitcoin addresses come in many flavors:
- **IPv4:** 32-bit addresses (e.g., `127.0.0.1`)
- **IPv6:** 128-bit addresses (e.g., `::1`)
- **Tor v3:** 256-bit addresses (e.g., `pg6mmjiyjmcrsslvykfwnntlaru7p5svn6y2ymmju6nubxndf4pscryd.onion`)

The original Bitcoin address format couldn't handle Tor v3's 256-bit addresses. **Addrv2** (BIP 155) was introduced to support arbitrary-length network addresses.

The test ensures that `anchors.dat` correctly stores and retrieves Tor v3 addresses.

### Setting Up the Tor Proxy (Lines 90-99)

```python
self.log.info("Ensure addrv2 support")
onion_conf = Socks5Configuration()
onion_conf.auth = True
onion_conf.unauth = True
onion_conf.addr = ('127.0.0.1', p2p_port(self.num_nodes))
onion_conf.keep_alive = True
onion_proxy = Socks5Server(onion_conf)
onion_proxy.start()
self.restart_node(0, extra_args=[f"-onion={onion_conf.addr[0]}:{onion_conf.addr[1]}"])
```

**What's happening:**
- A SOCKS5 proxy is started on a local port
- The node is restarted with `-onion=<proxy>`, telling it to route Tor connections through the proxy
- This allows the test to simulate a Tor connection without actually connecting to the Tor network

### Adding a Tor Block-Relay-Only Connection (Lines 101-102)

```python
self.log.info("Add 256-bit-address block-relay-only connections to node")
self.nodes[0].addconnection(ONION_ADDR, 'block-relay-only', v2transport=False)
```

`ONION_ADDR` is defined at the top of the file:

```python
ONION_ADDR = "pg6mmjiyjmcrsslvykfwnntlaru7p5svn6y2ymmju6nubxndf4pscryd.onion:8333"
```

The node now has a block-relay-only connection to a Tor v3 address.

### Shutdown and Verification (Lines 104-127)

```python
self.log.debug("Stop node")
with self.nodes[0].assert_debug_log(["DumpAnchors: Flush 1 outbound block-relay-only peer addresses to anchors.dat"]):
    self.stop_node(0)
onion_proxy.stop()
```

The node shuts down and writes the Tor address to `anchors.dat`.

Now the test verifies the file format:

```python
self.log.info("Check for addrv2 addresses in anchors.dat")
caddr = CAddress()
caddr.net = CAddress.NET_TORV3
caddr.ip, port_str = ONION_ADDR.split(":")
caddr.port = int(port_str)
# TorV3 addrv2 serialization:
# time(4) | services(1) | networkID(1) | address length(1) | address(32)
expected_pubkey = caddr.serialize_v2()[7:39].hex()
```

**What's happening:**
1. A `CAddress` object is constructed with the Tor v3 address
2. The test serializes it using the addrv2 format
3. It extracts bytes 7-39, which contain the 32-byte Tor public key

**The file structure:**
```
[4 bytes: timestamp]
[1 byte: services]
[1 byte: networkID]  <- identifies this as Tor v3
[1 byte: address length]  <- should be 32
[32 bytes: address]  <- the Tor public key
```

The test reads `anchors.dat` and confirms:

```python
services_index = 4 + 1 + 4 + 4  # magic, vector length, version, nTime
data = bytes()
with open(node_anchors_path, "rb") as file_handler:
    data = file_handler.read()
    assert_equal(data[services_index], 0x00)  # services == NONE
    anchors2 = data.hex()
    assert expected_pubkey in anchors2
```

**Key assertions:**
1. The Tor public key is present in the anchor file
2. The services byte is `0x00` (because the test never completed a handshake)

### Modifying the Services Byte (Lines 129-136)

Here's where it gets interesting:

```python
with open(node_anchors_path, "wb") as file_handler:
    # Modify service flags for this address even though we never connected to it.
    # This is necessary because on restart we will not attempt an anchor connection
    # to a host without our required services, even if its address is in the anchors.dat file
    new_data = bytearray(data)[:-32]
    new_data[services_index] = P2P_SERVICES
    new_data_hash = hash256(new_data)
    file_handler.write(new_data + new_data_hash)
```

**Why this step?**

Bitcoin Core won't attempt to connect to an anchor peer that doesn't advertise the services it needs (like `NODE_NETWORK`). Since the test never completed a handshake, the services byte is `0x00`. To test that the node actually tries to reconnect, the test manually sets the services byte to `P2P_SERVICES`.

**The checksum update:**
- The anchor file has a trailing 32-byte checksum (double SHA-256 hash)
- After modifying the services byte, the test recomputes the hash and appends it

### Reconnection Test (Lines 138-140)

```python
self.log.info("Restarting node attempts to reconnect to anchors")
with self.nodes[0].assert_debug_log([f"Trying to make an anchor connection to {ONION_ADDR}"]):
    self.start_node(0, extra_args=[f"-onion={onion_conf.addr[0]}:{onion_conf.addr[1]}"])
```

The node restarts and immediately tries to connect to the Tor anchor. The test confirms this by checking the debug log for the expected message.

**This proves:**
1. `anchors.dat` correctly stores Tor v3 addresses in addrv2 format
2. On restart, Bitcoin Core reads the addrv2 format and attempts reconnection
3. The anchor mechanism works for non-IPv4 addresses

---

## Key Takeaways for Developers

### 1. Curate Your Bootstrap Peers

When your application needs to reconnect to a network or cluster, don't save every connection you've ever made. Save a small list of the most trustworthy, long-lived connections. In Bitcoin Core's case, that's block-relay-only peers.

### 2. Graceful Degradation on Corruption

Corrupted files happen. Your application should:
- Detect corruption early (checksums, magic bytes, version checks)
- Log the error clearly
- Fall back to a safe default (in this case, normal peer discovery)
- Never crash on startup due to corrupted saved state

### 3. Future-Proof Your Serialization Format

The addrv2 format is a great example of extensible design. Instead of hardcoding IPv4 support, it includes a "network type" byte and a "length" byte, allowing it to support any address format — even ones that don't exist yet.

### 4. Think About Attack Vectors

Why does Bitcoin Core only save outbound block-relay-only connections? Because inbound connections are attacker-controlled. If you saved inbound connections, an attacker could:
1. Open many inbound connections to your node
2. Wait for you to shut down
3. Be the first peers you connect to on restart (eclipse attack)

By limiting anchors to outbound block-relay-only connections, Bitcoin Core makes this attack much harder.

---

## Connections to Other Tests

### Relationship to feature_addrman.py

While `feature_addrman.py` tests the long-term peer address database (`peers.dat`), `feature_anchors.py` tests the short-term emergency contact list (`anchors.dat`).

- **peers.dat:** Thousands of addresses learned over time, used for general peer discovery
- **anchors.dat:** 1-2 addresses from your most recent session, used for fast reconnection on restart

Both tests share the same philosophy: validate everything you load from disk, and never let corrupted data crash the node.

### Relationship to p2p_* Tests

Many p2p tests verify connection behavior (handshakes, message handling, peer selection). This test is unique in that it focuses on the **persistence and recovery** of connections across restarts.

---

## The Bigger Picture: Network Resilience

Anchors are part of Bitcoin Core's defense-in-depth strategy for network resilience. Consider the attack scenario:

**Without anchors:**
1. Attacker isolates your node (firewall rules, DNS poisoning, etc.)
2. You restart your node
3. Your node uses DNS seeds to find peers
4. Attacker controls the DNS response or floods your connection slots
5. You're now eclipsed (only connected to attacker nodes)

**With anchors:**
1. Attacker isolates your node
2. You restart your node
3. Your node immediately tries to reconnect to the 2 block-relay-only peers from your last session
4. If even one succeeds, you have a trustworthy view of the blockchain
5. Eclipse attack is much harder

Anchors don't prevent eclipse attacks entirely, but they raise the bar. An attacker must not only isolate you *now* but also ensure you weren't connected to any honest peers *before* you restarted.

---

## Conclusion

`feature_anchors.py` is a short test (145 lines), but it encodes deep wisdom about distributed systems:

1. **Bootstrap security matters.** The first connections you make on startup are critical. Choose them carefully.
2. **Small, curated lists are powerful.** You don't need to save everything. A few high-quality anchors are better than a hundred random peers.
3. **Corruption is inevitable.** Plan for it. Detect it. Recover from it gracefully.
4. **Extensibility is key.** The addrv2 format future-proofs Bitcoin Core for network types that don't exist yet.

Next time you build a distributed system — whether it's a blockchain node, a database cluster, or a chat application — ask yourself: *"What are my anchors? How do I bootstrap securely? What happens if my saved state is corrupted?"*

Bitcoin Core's answer is elegant: save a tiny list of your best connections, validate it on load, and fail gracefully if anything goes wrong.

---

## Further Reading

- **BIP 155 (addrv2):** https://github.com/bitcoin/bips/blob/master/bip-0155.mediawiki
- **Bitcoin Core's anchor code:** `src/net.cpp` (search for `DumpAnchors`)
- **Eclipse attacks on Bitcoin:** Heilman et al., "Eclipse Attacks on Bitcoin's Peer-to-Peer Network" (2015)

---

