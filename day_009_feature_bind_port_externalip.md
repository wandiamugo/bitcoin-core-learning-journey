# Day 9: feature_bind_port_externalip.py - Port Priority When Advertising External Addresses

**Test File:** `test/functional/feature_bind_port_externalip.py`
**Category:** Feature Tests
**Complexity:** Medium

*Testing that when you tell Bitcoin Core "advertise this IP address," it also figures out the correct port number to pair with it - like making sure your business card shows the right extension number, not just the building's main phone number.*

---

## The Analogy: The Business Card Port Number Puzzle

Imagine you work in a large office building with a complex phone system:
- The building's main number is (555) 100-0000
- Your direct line is (555) 200-0000
- You can receive calls on various extensions: 1001, 1002, 1003, etc.
- You have an internal intercom on extension 5000

Now, you're printing business cards to hand out at a conference. You want to advertise your direct line `(555) 200-0000`, but which extension should be printed with it?

- Should it be extension 1001 (from `-port=1001`)?
- Should it be extension 1002 (from `-bind=192.168.1.5:1002`)?
- Should it be extension 1003 (from the direct line itself: `-externalip=555-200-0000:1003`)?
- Should it be the default extension for your building?

**This test ensures Bitcoin Core follows a clear priority hierarchy** to get the port number right when advertising an external IP address.

## Test Overview

The `feature_bind_port_externalip.py` test validates that Bitcoin Core correctly determines which port number to advertise alongside an external IP address when various combinations of `-externalip`, `-port`, `-bind`, and `-whitebind` are specified.

**The Setup:** Like the previous test, this requires a routable IP address (`1.1.1.1`) configured on the test machine - think of it as having an actual phone line to test with.

**The Test Matrix:** The test runs 12 different Bitcoin nodes, each with a different combination of configuration arguments, and verifies that each one advertises the correct port number with the external IP `2.2.2.2`.

## The Port Priority Hierarchy: Who Wins?

Bitcoin Core uses this priority order when determining which port to advertise with an external IP:

### Priority 1: Port specified in `-externalip` itself (Highest Priority)
```bash
-externalip=2.2.2.2:30006
```
**Winner:** Port 30006 - "You explicitly told me the extension to use with this number"

### Priority 2: Port from `-bind` if no port in `-externalip`
```bash
-externalip=2.2.2.2 -bind=1.1.1.1:30005
```
**Winner:** Port 30005 - "You're binding to this port, so that's what you're listening on"

### Priority 3: Port from `-whitebind` if no `-bind`
```bash
-externalip=2.2.2.2 -whitebind=1.1.1.1:30020
```
**Winner:** Port 30020 - "Your whitelist bind has a port, use that"

### Priority 4: Port from `-port` if no specific bind port
```bash
-externalip=2.2.2.2 -port=30001
```
**Winner:** Port 30001 - "You set a general port preference"

### Priority 5: Default P2P port (Lowest Priority)
```bash
-externalip=2.2.2.2
```
**Winner:** 18444 (regtest) or 8333 (mainnet) - "No preference given, use the network default"

## The Test Cases: A Comprehensive Port Priority Matrix

Let's walk through the 12 test scenarios defined in the test file (feature_bind_port_externalip.py:23-38):

### Test Case 1: External IP with `-port` only
```python
['-externalip=2.2.2.2', '-port=30001']
Expected port: 30001
```

**The Logic:** No port in `-externalip`, no `-bind`, so we fall through to `-port`.

**Real-world scenario:** "I'm behind NAT. My router forwards external `2.2.2.2:30001` to my internal machine. I want to advertise that."

### Test Case 2: External IP with `-port` and `-bind` (without port)
```python
['-externalip=2.2.2.2', '-port=30002', '-bind=1.1.1.1']
Expected port: 30002
```

**The Logic:** `-bind` doesn't specify a port, so we use `-port`.

**Real-world scenario:** "I want to bind to a specific interface (`1.1.1.1`) but still use my configured port number."

### Test Case 3: External IP with `-bind` only (no port specified)
```python
['-externalip=2.2.2.2', '-bind=1.1.1.1']
Expected port: default_p2p_port (18444 for regtest)
```

**The Logic:** No port anywhere, fall back to network default.

**Real-world scenario:** "Simple setup - bind to this interface, use standard Bitcoin port."

### Test Case 4: External IP with `-port` and `-bind` with port
```python
['-externalip=2.2.2.2', '-port=30003', '-bind=1.1.1.1:30004']
Expected port: 30004
```

**The Logic:** `-bind` has a port (30004), which takes priority over `-port` (30003).

**Key Insight:** When `-bind` has a port, it beats `-port`. You're saying "I'm listening on this specific interface:port combo."

### Test Case 5: External IP with `-bind` with port only
```python
['-externalip=2.2.2.2', '-bind=1.1.1.1:30005']
Expected port: 30005
```

**The Logic:** `-bind` port is the most specific binding information available.

### Test Case 6: External IP with port in the `-externalip` and `-port`
```python
['-externalip=2.2.2.2:30006', '-port=30007']
Expected port: 30006
```

**The Logic:** Port in `-externalip` (30006) beats `-port` (30007).

**Critical Insight:** This is like writing the extension number directly on the business card - it's the most explicit statement of what port should be advertised with this specific IP.

### Test Case 7: External IP with port, `-port`, and `-bind` (no port)
```python
['-externalip=2.2.2.2:30008', '-port=30009', '-bind=1.1.1.1']
Expected port: 30008
```

**The Logic:** Port in `-externalip` (30008) beats both `-port` (30009) and the port-less `-bind`.

### Test Case 8: External IP with port and `-bind` without port
```python
['-externalip=2.2.2.2:30010', '-bind=1.1.1.1']
Expected port: 30010
```

**The Logic:** Port in `-externalip` is explicit for this specific address.

### Test Case 9: External IP with port, `-port`, and `-bind` with port
```python
['-externalip=2.2.2.2:30011', '-port=30012', '-bind=1.1.1.1:30013']
Expected port: 30011
```

**The Logic:** Port in `-externalip` (30011) beats everything: `-port` (30012) and `-bind` port (30013).

**The Hierarchy Demonstrated:** `-externalip` port > `-bind` port > `-port` flag > default

### Test Case 10: External IP with port and `-bind` with different port
```python
['-externalip=2.2.2.2:30014', '-bind=1.1.1.1:30015']
Expected port: 30014
```

**The Logic:** Even though you're binding to port 30015, you're explicitly advertising 30014 with the external IP.

**Real-world scenario:** "I'm listening on port 30015 internally, but my NAT router forwards port 30014 to my 30015. Advertise 30014."

### Test Case 11: Complex case with `-bind` and `-whitebind`
```python
['-externalip=2.2.2.2', '-port=30016', '-bind=1.1.1.1:30017', '-whitebind=1.1.1.1:30018']
Expected port: 30017
```

**The Logic:** `-bind` port (30017) takes priority over `-whitebind` port (30018).

**Why?** The test uses the first port it finds from the hierarchy. `GetListenPort()` checks `-bind` before `-whitebind`.

### Test Case 12: External IP with `-port` and `-whitebind` (no `-bind`)
```python
['-externalip=2.2.2.2', '-port=30019', '-whitebind=1.1.1.1:30020']
Expected port: 30020
```

**The Logic:** When there's no `-bind`, `-whitebind` port (30020) beats `-port` (30019).

**The Pattern:** If you specify a port on a binding option (bind or whitebind), that's what you're listening on, so that's what should be advertised (unless `-externalip` has its own port).

## How It Works Under the Hood: The `GetListenPort()` Function

The magic happens in `GetListenPort()` (net.cpp:137-161):

```cpp
uint16_t GetListenPort()
{
    // Priority 1: If -bind= is provided with ":port" part, use that
    for (const std::string& bind_arg : gArgs.GetArgs("-bind")) {
        constexpr uint16_t dummy_port = 0;

        const std::optional<CService> bind_addr{Lookup(bind_arg, dummy_port, /*fAllowLookup=*/false)};
        if (bind_addr.has_value() && bind_addr->GetPort() != dummy_port)
            return bind_addr->GetPort();  // Found a -bind with port!
    }

    // Priority 2: If -whitebind= without NoBan flag is provided, use that port
    for (const std::string& whitebind_arg : gArgs.GetArgs("-whitebind")) {
        NetWhitebindPermissions whitebind;
        bilingual_str error;
        if (NetWhitebindPermissions::TryParse(whitebind_arg, whitebind, error)) {
            if (!NetPermissions::HasFlag(whitebind.m_flags, NetPermissionFlags::NoBan)) {
                return whitebind.m_service.GetPort();  // Found a -whitebind with port!
            }
        }
    }

    // Priority 3: Otherwise, use -port= or default
    return static_cast<uint16_t>(gArgs.GetIntArg("-port", Params().GetDefaultPort()));
}
```

This is like checking your office phone system configuration:
1. First, check if you specified a direct extension when setting up your desk phone (bind port)
2. Then, check if you specified an extension for your priority line (whitebind port)
3. Finally, use your general extension preference (port flag) or the building default

## The External IP Processing: Where the Port Gets Paired

During initialization, `init.cpp:1709-1715` processes the `-externalip` arguments:

```cpp
for (const std::string& strAddr : args.GetArgs("-externalip")) {
    const std::optional<CService> addrLocal{Lookup(strAddr, GetListenPort(), fNameLookup)};
    if (addrLocal.has_value() && addrLocal->IsValid())
        AddLocal(addrLocal.value(), LOCAL_MANUAL);
    else
        return InitError(ResolveErrMsg("externalip", strAddr));
}
```

**The Key Line:** `Lookup(strAddr, GetListenPort(), fNameLookup)`

This says: "Parse the external IP string. If it has a port, use that port. If it doesn't have a port, call `GetListenPort()` to figure out what port to use."

**The `Lookup()` Function Logic:**
```cpp
// Pseudocode for understanding
if (strAddr contains ":port") {
    use_the_port_from_strAddr;  // e.g., "2.2.2.2:30006" → port 30006
} else {
    use_GetListenPort_result;    // e.g., "2.2.2.2" → port from GetListenPort()
}
```

Then it adds this IP:port combination to `mapLocalHost` with `LOCAL_MANUAL` priority (the highest priority score) so it will definitely be advertised to peers.

## The Business Card Printing Process: How Peers Learn Your Address

Once the external IP and port are added to `mapLocalHost`:

**Step 1: Peer Requests Your Address**
When you connect to a peer, they can ask "what's your address?" via the Bitcoin P2P protocol.

**Step 2: Find Best Address to Advertise**
The `GetLocal()` function (net.cpp:164) scans `mapLocalHost` and picks the best address based on:
- Reachability from the peer's network
- Priority score (`LOCAL_MANUAL` = 4 is highest)

**Step 3: Send Address to Peer**
Your node sends an `addr` message containing the IP:port pair.

**Step 4: Peer Gossips Your Address**
The peer shares your address with other peers, who share it further, populating the network's address database.

**Step 5: Other Nodes Try to Connect**
Eventually, other nodes use your advertised IP:port to connect to you.

**The Critical Point:** If you advertise the wrong port, other nodes will try to connect to you on that port and fail. It's like printing the wrong extension on your business card - people call and can't reach you.

## Why This Test Matters: The NAT Router Scenario

The most common real-world use case for `-externalip` is **NAT traversal**:

### The Problem
You run Bitcoin Core on your home computer:
- **Internal IP:** `192.168.1.100:8333` (your apartment number)
- **Router's External IP:** `203.0.113.50` (the apartment building's address)
- **Port Forwarding:** Router forwards `203.0.113.50:8333` → `192.168.1.100:8333`

### Without `-externalip`
Bitcoin Core might advertise `192.168.1.100:8333` to peers. This is useless - no one on the internet can route to `192.168.x.x`.

### With `-externalip`
```bash
bitcoind -externalip=203.0.113.50
```

Bitcoin Core advertises `203.0.113.50:8333`, which other nodes can actually reach via the port forwarding.

### But What If You Use a Non-Standard Port?

**Scenario:** Your ISP blocks port 8333, so you forward port 18333 instead:
```bash
# Router forwards: external 203.0.113.50:18333 → internal 192.168.1.100:8333
```

**Wrong Configuration:**
```bash
bitcoind -externalip=203.0.113.50 -port=8333
```
This advertises `203.0.113.50:8333`, but your router isn't forwarding that port!

**Correct Configuration:**
```bash
bitcoind -externalip=203.0.113.50:18333 -bind=192.168.1.100:8333
```

This:
- Binds to internal `192.168.1.100:8333` (where it actually listens)
- Advertises external `203.0.113.50:18333` (where others should connect)

**The test validates this works correctly**: port 18333 from `-externalip` takes priority over the bind port 8333.

## Real-World Use Cases: When Ports Get Complicated

### Use Case 1: The Cloud Server with Multiple IPs
```bash
bitcoind -bind=10.0.1.5:8333      # Internal cloud network
         -externalip=203.0.113.10:8333   # Public IP #1
         -externalip=198.51.100.20:9333  # Public IP #2 (non-standard port)
```

- Internal services connect via `10.0.1.5:8333`
- Public peers can use either `203.0.113.10:8333` or `198.51.100.20:9333`
- Each external IP can have its own port!

### Use Case 2: The Tor + Clearnet Hybrid
```bash
bitcoind -externalip=203.0.113.5:8333               # Clearnet
         -externalip=abc123...xyz.onion:8334       # Tor hidden service
```

Different network types, different ports, all advertised correctly.

### Use Case 3: The Testing Lab
```bash
bitcoind -port=18444              # Regtest default
         -bind=127.0.0.1:18445    # Local bind with different port
         -externalip=1.1.1.1:18446  # External with yet another port
```

Each layer has its own port, and the test ensures the external advertisement gets the right one (18446).

## What Could Go Wrong Without This Test: The Unreachable Node

### Scenario 1: The Ignored Externalip Port
**Bug:** Code ignores port in `-externalip`, always uses `-port` or default.

**Result:**
```bash
bitcoind -externalip=203.0.113.5:9000 -port=8333
# Advertises: 203.0.113.5:8333 (WRONG!)
# Actually listening on: 9000
```

Other nodes try to connect to your port 8333, fail, mark you as unreachable.

### Scenario 2: The Bind Port Confusion
**Bug:** Code uses bind port even when externalip has its own port.

**Result:**
```bash
bitcoind -externalip=203.0.113.5:9000 -bind=192.168.1.5:8333
# Advertises: 203.0.113.5:8333 (WRONG!)
```

Your NAT router forwards port 9000, but you advertise 8333. Connections fail.

### Scenario 3: The Whitebind Priority Error
**Bug:** Code treats `-whitebind` same as `-bind` even when bind is specified.

**Result:** Unpredictable behavior when both are specified.

## The Test Methodology: Exhaustive Combination Testing

The test uses a brilliant approach:

**1. Define Test Matrix** (feature_bind_port_externalip.py:23-38)
```python
EXPECTED = [
    [[arguments], expected_port],
    [[arguments], expected_port],
    # ... 12 different combinations
]
```

**2. Launch One Node Per Scenario**
```python
self.num_nodes = len(EXPECTED)  # 12 nodes
self.extra_args = list(map(lambda e: e[0], EXPECTED))  # Each node gets different args
```

**3. Verify Each Node Advertises Correct Port**
```python
for i in range(len(EXPECTED)):
    expected_port = EXPECTED[i][1]
    for local in self.nodes[i].getnetworkinfo()['localaddresses']:
        if local['address'] == '2.2.2.2':
            assert_equal(local['port'], expected_port)  # Check the port!
```

This is like testing 12 different business card printing configurations simultaneously to ensure each one prints the right phone number and extension combination.

## The Technical Requirement: Why Routable Addresses Again?

Just like the previous test, this one requires `1.1.1.1` configured because:

```cpp
if (!addr.IsRoutable())
    return false;  // Won't add to mapLocalHost
```

Non-routable addresses (like `127.0.0.1` or `192.168.x.x`) won't be advertised, so the test couldn't verify anything.

It's like testing business card printing with fake phone numbers - the printer might reject them before printing.

## The Special Case: `default_p2p_port` in Test Case 3

Notice this test case (feature_bind_port_externalip.py:26):
```python
[['-externalip=2.2.2.2', '-bind=1.1.1.1'], 'default_p2p_port']
```

The expected port is a string, not a number! The test handles this (feature_bind_port_externalip.py:64-65):

```python
if expected_port == 'default_p2p_port':
    expected_port = p2p_port(i)  # Get the default port for this node's network (regtest)
```

**Why?** Because the default port varies by network:
- Mainnet: 8333
- Testnet: 18333
- Regtest: 18444
- Signet: 38333

The test runs in regtest mode, so `p2p_port(i)` returns 18444.

## Conclusion: The Right Phone Number on the Right Business Card

`feature_bind_port_externalip.py` ensures that Bitcoin Core's port selection hierarchy works correctly when advertising external IP addresses. Like printing business cards with the correct phone number and extension combination, this test validates that:

1. **Explicit beats implicit** - Port in `-externalip` beats everything else
2. **Specific beats general** - Bind ports beat the generic `-port` flag
3. **First specific wins** - When multiple binds exist, the first one with a port wins
4. **Defaults work** - When nothing is specified, the network default is used

The test covers 12 different configuration combinations, ensuring that whether you're running a simple home node or a complex multi-IP cloud setup, Bitcoin Core will advertise the port where peers can actually reach you.

Without this test, a subtle regression could cause your node to advertise an unreachable port, making your node invisible to the network - like handing out business cards with the wrong phone number.

The test embodies the principle: **"When I say where others should connect to me, make sure you tell them the right port number."**

---

## References

**Test File:**
- `test/functional/feature_bind_port_externalip.py:23-38` - Test matrix with 12 configuration combinations
- `test/functional/feature_bind_port_externalip.py:54-58` - Skip mechanism requiring manual setup
- `test/functional/feature_bind_port_externalip.py:62-72` - Port verification loop

**Source Code:**
- `src/init.cpp:1709-1715` - Processing `-externalip` arguments with port lookup
- `src/net.cpp:137-161` - `GetListenPort()` - Port priority hierarchy implementation
- `src/net.cpp:164-214` - `GetLocal()` - Finding best address to advertise to peer
- `src/net.cpp:304-307` - `AddLocal()` overload that uses `GetListenPort()`
- `src/net.h:147-154` - `LOCAL_*` priority score definitions
