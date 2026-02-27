# Day 8: feature_bind_port_discover.py - Controlling Which Network Addresses Get Advertised

**Test File:** `test/functional/feature_bind_port_discover.py`
**Category:** Feature Tests
**Complexity:** Medium

*Testing that `-bind` controls not just where Bitcoin Core listens, but also which addresses it advertises to the network - like choosing which phone numbers to print on your business card instead of listing them all.*

## The Analogy: Choosing Which Phone Numbers to Print

Imagine you're a business owner with multiple phone numbers:
- A main office line
- A personal cell phone
- An internal intercom extension
- A fax line

When you print business cards to hand out at a networking event, you don't want to list ALL your numbers - especially not the internal intercom that outsiders can't even call! You want to choose which numbers appear on your card.

**This test verifies that Bitcoin Core gives you the same control**: when your computer has multiple network addresses, you can choose which ones to advertise to the Bitcoin network, not just broadcast all of them indiscriminately.

## Test Overview

The `feature_bind_port_discover.py` test validates how Bitcoin Core advertises its local network addresses to peers based on the `-bind` and `-discover` configuration options. It ensures that when you explicitly bind to a specific interface, only that interface's address is advertised to the network.

**The Setup:** This test requires special network configuration - the test machine must have two routable IP addresses configured:
- `1.1.1.1` (configured as an interface alias)
- `2.2.2.2` (configured as another interface alias)

Think of this like temporarily installing two separate phone lines to test which ones get printed on your business card.

## What This Test Does

The test creates two Bitcoin nodes with different configurations:

### Node 0: The Enthusiastic Networker (No `-bind` specified)

```python
['-discover', f'-port={BIND_PORT}']  # bind on any interface
```

This node is like someone who puts ALL their phone numbers on their business card - office, cell, home, fax, everything. With `-discover` enabled but no specific `-bind`, the node advertises every network interface it finds.

**Test Verification (feature_bind_port_discover.py:59-67):**
```python
for local in self.nodes[0].getnetworkinfo()['localaddresses']:
    if local['address'] == ADDR1:  # Found 1.1.1.1?
        found_addr1 = True
    if local['address'] == ADDR2:  # Found 2.2.2.2?
        found_addr2 = True
assert found_addr1
assert found_addr2
```

Both addresses should appear in the `localaddresses` list - all phone numbers on the business card.

### Node 1: The Selective Networker (With `-bind` specified)

```python
['-discover', f'-bind={ADDR1}:{BIND_PORT}']  # bind only to 1.1.1.1
```

This node is like someone who carefully chooses to only print their office number on their business card, even though they have other numbers. With `-bind=1.1.1.1:31001`, the node should ONLY advertise that specific address.

**Test Verification (feature_bind_port_discover.py:72-78):**
```python
for local in self.nodes[1].getnetworkinfo()['localaddresses']:
    if local['address'] == ADDR1:  # Should find 1.1.1.1
        found_addr1 = True
    assert_not_equal(local['address'], ADDR2)  # Should NOT find 2.2.2.2
assert found_addr1
```

Only `1.1.1.1` should appear - just the chosen phone number on the business card.

## Why This Matters: The House with Multiple Doors Analogy

Think of your Bitcoin node like a house with multiple doors:
- **Front door**: Your public-facing IP address
- **Back door**: A private network address
- **Side door**: Maybe a VPN connection
- **Basement door**: Perhaps a Tor connection

When Bitcoin Core runs with `-discover` but no `-bind`, it's like putting signs on ALL the doors saying "Bitcoin node - come on in!" But here's the problem:

1. **The basement door (Tor) should stay secret** - you don't want that address announced to the whole network
2. **The back door (private IP) is useless to strangers** - addresses like `192.168.1.100` can't be reached from the internet
3. **You might only want visitors using the front door** - your carefully chosen public interface

The `-bind` option is like saying: "Only put a 'welcome' sign on the front door, even though I have other doors."

## The Address Discovery Dance: How It Works Under the Hood

### The Automatic Discovery Process

When Bitcoin Core starts with `-discover` enabled, it goes through this process:

**Step 1: Interface Enumeration** - The `GetLocalAddresses()` function (common/netif.cpp:318) asks the operating system: "What network interfaces do we have?"

```cpp
std::vector<CNetAddr> GetLocalAddresses()
{
    // On Windows: Uses GetAdaptersAddresses() API
    // On Unix-like systems: Uses getifaddrs() system call

    // Returns all network interfaces that are:
    // - UP and running
    // - NOT loopback (127.0.0.1)
    // - NOT multicast or anycast
}
```

This is like doing a house inspection and making a list of all the doors.

**Step 2: The Discovery Function** - `Discover()` (net.cpp:3188) processes this list:

```cpp
void Discover()
{
    if (!fDiscover)
        return;  // Discovery disabled, don't advertise anything automatically

    for (const CNetAddr &addr: GetLocalAddresses()) {
        if (AddLocal(addr, LOCAL_IF))
            LogPrintf("%s: %s\n", __func__, addr.ToStringAddr());
    }
}
```

This is like taking your list of doors and deciding which ones get "welcome" signs.

### The Bind Override: Selective Advertisement

When you use `-bind=<specific-address>`, the `CConnman::Bind()` function (net.cpp:3245) takes control:

```cpp
bool CConnman::Bind(const CService& addr_, unsigned int flags, NetPermissionFlags permissions)
{
    // First, actually bind to the socket (open the door)
    if (!BindListenPort(addr, strError, permissions)) {
        return false;
    }

    // Then, decide whether to advertise this address
    if (addr.IsRoutable() && fDiscover && !(flags & BF_DONT_ADVERTISE) &&
        !NetPermissions::HasFlag(permissions, NetPermissionFlags::NoBan)) {
        AddLocal(addr, LOCAL_BIND);  // Add with BIND priority
    }

    return true;
}
```

**The Key Insight:** When you explicitly specify `-bind`, Bitcoin Core:
- **Only** calls `Bind()` for your specified address
- Does **not** run `Discover()` to enumerate all interfaces
- Therefore only that one address gets added to the advertisement list

It's the difference between:
- "Put signs on all doors I can find" (automatic discovery)
- "Only put a sign on this specific door I'm pointing at" (explicit bind)

### The Address Book: `mapLocalHost`

Bitcoin Core maintains a global map called `mapLocalHost` (net.cpp:118) - think of it as the "addresses to advertise" phonebook:

```cpp
std::map<CNetAddr, LocalServiceInfo> mapLocalHost;
```

Every address in this map gets announced to peers. The `getnetworkinfo` RPC (rpc/net.cpp:627) that the test uses simply reads this map (rpc/net.cpp:718):

```cpp
UniValue localAddresses(UniValue::VARR);
{
    LOCK(g_maplocalhost_mutex);
    for (const std::pair<const CNetAddr, LocalServiceInfo> &item : mapLocalHost)
    {
        UniValue rec(UniValue::VOBJ);
        rec.pushKV("address", item.first.ToStringAddr());
        rec.pushKV("port", item.second.nPort);
        rec.pushKV("score", item.second.nScore);
        localAddresses.push_back(std::move(rec));
    }
}
```

The test is essentially checking: "How many entries are in the phonebook?"

## The Address Reputation System: Scoring

Bitcoin Core uses a priority scoring system for addresses - like having different categories of business cards:

- **`LOCAL_NONE = 0`**: Scratch paper (not actually advertised)
- **`LOCAL_IF = 1`**: Standard business card (auto-discovered from interfaces)
- **`LOCAL_BIND = 2`**: Premium business card (you specifically chose this)
- **`LOCAL_MAPPED = 3`**: Gold-embossed card (discovered via UPnP/NAT-PMP port mapping)
- **`LOCAL_MANUAL = 4`**: Platinum card (you manually specified with `-externalip`)

Higher scores mean "trust this address more" - if the same address appears multiple times with different scores, the highest score wins.

## Why This Test Requires Manual Setup: The Lab Experiment Problem

The test includes this skip mechanism (feature_bind_port_discover.py:47):

```python
def skip_test_if_missing_module(self):
    if not self.options.ihave1111and2222:
        raise SkipTest(
            f"To run this test make sure that {ADDR1} and {ADDR2} (routable addresses) are "
            "assigned to the interfaces on this machine and rerun with --ihave1111and2222")
```

**Why can't the test just configure the addresses itself?**

Think of it like a chemistry experiment that requires working with dangerous materials. The test **could** theoretically:
1. Run commands to add network interface aliases
2. Perform the test
3. Clean up the aliases

But this would require:
- **Root/administrator privileges** (like needing a special license)
- **Risk of leaving the network broken** if the test crashes (like a chemical spill)
- **Platform-specific commands** (different procedures for Linux, BSD, macOS, Windows)
- **Potential conflicts** with existing network configuration

Instead, the test documents the manual setup and skips itself unless you explicitly say "I've done the setup with `--ihave1111and2222`".

**Setup Example (Linux):**
```bash
# Add two temporary IP aliases to loopback interface
sudo ifconfig lo:0 1.1.1.1/32 up
sudo ifconfig lo:1 2.2.2.2/32 up

# Run the test
./test/functional/feature_bind_port_discover.py --ihave1111and2222

# Clean up afterward
sudo ifconfig lo:0 down
sudo ifconfig lo:1 down
```

## Real-World Use Cases: When You Need This Control

### Use Case 1: The Tor + Public IP Setup (The Secret Door and Public Door)

Imagine a house with both a visible front entrance and a hidden underground tunnel (Tor):

```bash
bitcoind -bind=203.0.113.5:8333    # Public IP - advertise this
         -bind=127.0.0.1:9050       # Tor SOCKS - DON'T advertise this
```

You want the public IP advertised so clearnet peers can connect, but the Tor address should stay hidden (it's only for outbound connections anyway).

### Use Case 2: The Multi-IP VPS (The Business with Multiple Storefronts)

A VPS provider gives you 5 IP addresses, but you only want to advertise one:

```bash
bitcoind -bind=203.0.113.10:8333   # Only advertise this specific IP
```

Why not advertise all 5?
- Some might be temporary assignments
- Some might be on different networks with different routing costs
- You might rotate them for other services

### Use Case 3: The Corporate Firewall (The Building with Public and Private Wings)

An organization runs Bitcoin Core on a server with both internal and external interfaces:

```bash
bitcoind -bind=192.168.1.100:8333  # Internal network only
```

This prevents advertising the public interface, keeping the Bitcoin node as an internal-only service.

### Use Case 4: The Home Router Confusion (The Apartment Complex Problem)

Home users behind NAT might have:
- External IP: `198.51.100.50` (the apartment building's address)
- Internal IP: `192.168.1.5` (your apartment number)

Without `-bind`, Bitcoin Core might advertise `192.168.1.5` to the network, which is useless to external peers (they can't route to your apartment directly). Using `-externalip` or UPnP helps, but `-bind` gives manual control.

## What Could Go Wrong Without This Test: The Cautionary Tales

### Tale 1: The Privacy Leak

Without this behavior, a user configuring `-bind=<tor-address>` might expect it to ONLY advertise that address, but if discovery also ran, their clearnet IP would leak to the network, defeating the privacy goal.

### Tale 2: The Network Pollution

Every Bitcoin node gossips addresses to peers. If everyone's node advertised unreachable addresses (internal IPs, link-local addresses, etc.), the network's address database would be polluted with junk, making it harder to find connectable peers.

### Tale 3: The Configuration Confusion

A user sees their node advertising addresses they didn't expect. They file a bug report: "I used `-bind` but it's still advertising all my IPs!" Without this test catching regressions, this could actually happen.

## The Test Philosophy: Trust But Verify

This test embodies a core principle: **"When a user says 'only use this address,' we mean ONLY this address - for both binding AND advertising."**

The `-bind` option has an implicit contract:
- **Explicit**: Bind to this socket
- **Implicit**: Don't advertise other addresses

Without this test, a code change might break the implicit part while keeping the explicit part working. The node would still *listen* on the right address but *advertise* the wrong addresses.

## The Technical Gotcha: Why Routable Addresses?

You might wonder: "Why use `1.1.1.1` and `2.2.2.2` instead of `127.0.0.2` and `127.0.0.3`?"

Because of this filter in `AddLocal()` (net.cpp:276):

```cpp
if (!addr.IsRoutable())
    return false;
```

Bitcoin Core refuses to advertise non-routable addresses (loopback, private IPs, etc.) because they're useless to remote peers. The test needs routable addresses to actually exercise the advertisement logic.

It's like testing whether your business card printing process works - you need to use real phone numbers, not fake ones that would be filtered out before printing.

## Conclusion: The Business Card You Choose to Print

`feature_bind_port_discover.py` validates that Bitcoin Core respects your choice of which network addresses to advertise. Like choosing which phone numbers appear on your business card, you should have precise control over your network presence.

The test may require manual setup and can't run in standard CI, but it serves as both:
1. **Validation**: Ensures the feature works correctly
2. **Documentation**: Shows how `-bind` and `-discover` interact

In a world where network configurations are increasingly complex - with Tor, VPNs, cloud hosting, and multi-homed servers - this granular control isn't a nice-to-have, it's essential for privacy, functionality, and network health.

The test ensures that when you say "only this door gets a welcome sign," Bitcoin Core doesn't go putting signs on all the other doors behind your back.

---

## References

**Test File:**
- `test/functional/feature_bind_port_discover.py:47` - Skip mechanism requiring manual setup
- `test/functional/feature_bind_port_discover.py:59-67` - Node 0 verification (all addresses)
- `test/functional/feature_bind_port_discover.py:72-78` - Node 1 verification (selective address)

**Source Code:**
- `src/common/netif.cpp:318` - `GetLocalAddresses()` - Enumerates network interfaces
- `src/net.cpp:118` - `mapLocalHost` - Global map of advertised addresses
- `src/net.cpp:276` - `AddLocal()` - Routability filter
- `src/net.cpp:3188` - `Discover()` - Auto-discovery of local addresses
- `src/net.cpp:3245` - `CConnman::Bind()` - Explicit bind handling
- `src/rpc/net.cpp:627` - `getnetworkinfo` RPC implementation
- `src/rpc/net.cpp:718` - Reading `mapLocalHost` for RPC response
