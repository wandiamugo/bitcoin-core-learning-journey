# Day 7: Understanding feature_bind_extra.py - Network Binding and Tor Integration

**Test File:** `test/functional/feature_bind_extra.py`
**Category:** Feature Tests
**Complexity:** Medium

---

## Introduction

### The Port Authority Analogy

Imagine you run a shipping company with multiple warehouses in the same city. Each warehouse needs its own loading dock so trucks can deliver goods. You can't have all deliveries go to the same address — some customers need express delivery through warehouse A, others need standard delivery through warehouse B.

When you file your business license, you specify:
- **Warehouse A:** 123 Main Street (standard deliveries)
- **Warehouse B:** 456 Oak Avenue (express/premium deliveries)

The city verifies you actually have docks at both addresses. If you claim to have a dock at Oak Avenue but don't, shipments fail and your business license is revoked.

Bitcoin Core's `-bind` flag works the same way: it tells bitcoind which network interfaces and ports to "open for business" for different types of P2P connections. The `-bind=...=onion` variant specifically designates which address should handle Tor connections.

`feature_bind_extra.py` tests whether Bitcoin Core actually opens network sockets on the ports you specified — especially when you configure separate bindings for clearnet and Tor traffic.

### Why This Matters for Developers

Network binding configuration is fundamental to any networked application. The pattern tested here appears across every server architecture:

- **Web servers:** Nginx/Apache bind to `:80` for HTTP, `:443` for HTTPS
- **Databases:** PostgreSQL binds to `:5432`, MySQL to `:3306`, with separate interfaces for local vs. remote access
- **SSH servers:** sshd binds to `:22`, but you might run a second instance on `:2222` for admin access
- **Docker containers:** Each container exposes specific ports, mapped to host interfaces
- **Tor hidden services:** Require binding a local port that the Tor daemon forwards to the hidden service

`feature_bind_extra.py` tests a specific requirement: **verifying that the software actually binds to the ports it claims to, and that binding configuration correctly handles specialized network targets (like Tor)**.

### The Technical Version

Bitcoin Core accepts multiple `-bind` arguments to control which network interfaces and ports it listens on for P2P connections. The `-bind=<addr>:<port>=onion` syntax tells bitcoind: "This address/port is specifically for Tor connections."

The test launches three nodes with different binding configurations:
1. **Node 0:** Only `-bind=127.0.0.1:port=onion` (Tor-only binding)
2. **Node 1:** Both `-bind=127.0.0.1:port` and `-bind=127.0.0.1:port2=onion` (clearnet + Tor)
3. **Node 2:** Only `-bind=127.0.0.1:port` (clearnet-only binding)

For each node, the test queries the operating system to see which ports the bitcoind process has actually bound to, then verifies they match the expected configuration.

**Platform constraint:** This test only runs on Linux, because it uses Linux-specific `/proc` filesystem queries to inspect network bindings.

---

## What Is Network Binding?

Before exploring the test, we need to understand what "binding to a port" means at the OS level.

### Sockets and Binding

A network socket is an endpoint for sending/receiving data. Creating a server socket involves three steps:

1. **Create a socket:**
   ```c
   int sockfd = socket(AF_INET, SOCK_STREAM, 0);
   ```
   This allocates a socket descriptor but doesn't associate it with any address yet.

2. **Bind to an address/port:**
   ```c
   struct sockaddr_in addr;
   addr.sin_family = AF_INET;
   addr.sin_addr.s_addr = inet_addr("127.0.0.1");
   addr.sin_port = htons(8333);
   bind(sockfd, (struct sockaddr*)&addr, sizeof(addr));
   ```
   This tells the OS: "Any packets arriving at `127.0.0.1:8333` should be routed to this socket."

3. **Listen for connections:**
   ```c
   listen(sockfd, 10);  // queue up to 10 pending connections
   ```

After `bind()` succeeds, the OS reserves that address/port combination for this process. No other process can bind to the same `127.0.0.1:8333` until this socket is closed.

### Why Multiple Bindings?

Bitcoin Core may need to listen on multiple interfaces for different purposes:

**Clearnet vs. Tor:**
- **Clearnet:** Standard internet connections (IPv4/IPv6)
- **Tor:** Anonymous connections through the Tor network

If you want both types of connections, you need to bind to at least two ports: one for clearnet, one for Tor.

**Multiple network interfaces:**
A server might have:
- `127.0.0.1` (localhost only)
- `192.168.1.10` (local network)
- `203.0.113.42` (public internet)

You can choose to bind only to localhost (for testing), only to the public IP (for internet peers), or both.

### The `-bind` Flag

Bitcoin Core's `-bind` flag syntax:
```
-bind=<addr>:<port>[=onion]
```

Examples:
- `-bind=0.0.0.0:8333` — Bind to all interfaces on port 8333 (default P2P port)
- `-bind=127.0.0.1:18444` — Bind only to localhost on port 18444 (regtest)
- `-bind=192.168.1.10:8333=onion` — Bind to local network address, mark as Tor target

The `=onion` suffix tells Bitcoin Core: "This binding is for Tor connections. Route .onion peer connections through this address."

---

## The Test: What Does It Do?

### The Setup

```python
def set_test_params(self):
    self.setup_clean_chain = True
    self.bind_to_localhost_only = False
    self.num_nodes = 3
```

Three nodes, clean chain. Crucially, `bind_to_localhost_only = False` prevents the test framework from automatically adding `-bind=127.0.0.1` to every node — we're manually controlling all bind arguments.

```python
def skip_test_if_missing_module(self):
    self.skip_if_platform_not_linux()
```

This test only runs on Linux because the implementation uses Linux-specific system calls to query bound ports.

### Building the Node Configurations

```python
def setup_network(self):
    loopback_ipv4 = addr_to_hex("127.0.0.1")
    port = p2p_port(self.num_nodes)

    self.expected = []
```

`addr_to_hex("127.0.0.1")` converts the IP address to a hexadecimal representation (for later comparison with `/proc` data).

`port = p2p_port(self.num_nodes)` picks an unused port number starting from the test framework's port pool.

#### Node 0: Tor-only binding

```python
self.expected.append(
    [
        [f"-bind=127.0.0.1:{port}=onion"],
        [(loopback_ipv4, port)]
    ],
)
port += 1
```

Node 0 is configured with **only** a Tor binding. There's no clearnet binding specified.

**Expected behavior:** Bitcoin Core should bind to `127.0.0.1:port` (because it's the only `-bind` specified), and that binding should be marked as the Tor target.

The expected bindings list contains one entry: `(loopback_ipv4, port)`.

#### Node 1: Clearnet + Tor bindings

```python
self.expected.append(
    [
        [f"-bind=127.0.0.1:{port}", f"-bind=127.0.0.1:{port + 1}=onion"],
        [(loopback_ipv4, port), (loopback_ipv4, port + 1)]
    ],
)
port += 2
```

Node 1 has two `-bind` arguments:
1. `-bind=127.0.0.1:port` — standard clearnet P2P
2. `-bind=127.0.0.1:port+1=onion` — Tor connections

**Expected behavior:** Two bindings, one for each specified port.

#### Node 2: Clearnet-only binding

```python
self.expected.append(
    [
        [f"-bind=127.0.0.1:{port}"],
        [(loopback_ipv4, port)]
    ],
)
port += 1
```

Node 2 has only a standard clearnet binding, no Tor target.

**Expected behavior:** One binding on the specified port.

### Extracting Extra Args and Starting Nodes

```python
self.extra_args = list(map(lambda e: e[0], self.expected))
self.setup_nodes()
```

`self.extra_args` is set to the first element of each `self.expected` entry — the list of command-line arguments. The test framework passes these to `bitcoind` when starting each node.

Example for node 1:
```python
self.extra_args[1] = ["-bind=127.0.0.1:18446", "-bind=127.0.0.1:18447=onion"]
```

`self.setup_nodes()` starts all three nodes with their respective `-bind` configurations.

---

## The Actual Test: Verifying Bound Ports

```python
def run_test(self):
    for i, (args, expected_services) in enumerate(self.expected):
        self.log.info(f"Checking listening ports of node {i} with {args}")
        pid = self.nodes[i].process.pid
        binds = set(get_bind_addrs(pid))
```

For each node, the test:
1. Gets the process ID (PID) of the running bitcoind instance
2. Calls `get_bind_addrs(pid)` to query which addresses/ports that process has bound

### The `get_bind_addrs()` Function

This is the core of the test. It's defined in `test_framework/netutil.py`:

```python
def get_bind_addrs(pid):
    """
    Get the addresses a process is bound to.

    Returns a list of tuples: (addr_hex, port)
    """
    # On Linux, parse /proc/<pid>/net/tcp and /proc/<pid>/net/tcp6
    binds = []

    # Read IPv4 bindings from /proc/<pid>/net/tcp
    with open(f'/proc/{pid}/net/tcp', 'r') as f:
        for line in f:
            # Each line: sl local_address rem_address st tx_queue:rx_queue ...
            # Example: "0: 0100007F:1F90 00000000:0000 0A ..."
            #          -> 0100007F = 127.0.0.1 in hex (reversed byte order)
            #          -> 1F90 = port 8080 in hex
            parts = line.split()
            if len(parts) < 2:
                continue

            local_addr = parts[1]
            if ':' not in local_addr:
                continue

            addr_hex, port_hex = local_addr.split(':')
            port = int(port_hex, 16)

            # Only include LISTEN state (st = 0A)
            if parts[3] == '0A':
                binds.append((addr_hex, port))

    # Similar parsing for /proc/<pid>/net/tcp6 ...

    return binds
```

The `/proc/<pid>/net/tcp` file lists all TCP sockets for a process. Each line represents one socket. The second column `local_address` shows the bound address/port in hex format.

**Example `/proc/<pid>/net/tcp` entry:**
```
  sl  local_address rem_address   st tx_queue rx_queue tr tm->when retrnsmt   uid  timeout inode
   0: 0100007F:47F0 00000000:0000 0A 00000000:00000000 00:00000000 00000000   999        0 12345
```

Breaking down `0100007F:47F0`:
- `0100007F` → `127.0.0.1` (in little-endian hex: `7F 00 00 01`)
- `47F0` → port `18416` in decimal

The `st` field `0A` means the socket is in LISTEN state (waiting for connections).

`get_bind_addrs(pid)` parses these files and returns a list of `(addr_hex, port)` tuples for all listening sockets.

### Filtering Out Irrelevant Bindings

```python
# Remove IPv6 addresses
ipv6_addr_len_bytes = 32
binds = set(filter(lambda e: len(e[0]) != ipv6_addr_len_bytes, binds))

# Remove RPC ports
binds = set(filter(lambda e: e[1] != rpc_port(i), binds))
```

**Why remove IPv6?**
On some systems, IPv6 is configured but `::1` (loopback) isn't available. Bitcoin Core may bind to `::` (all IPv6 interfaces) even when we didn't explicitly request it. This creates unpredictable test results. To avoid flakiness, the test ignores all IPv6 bindings.

**Why remove RPC ports?**
Every Bitcoin Core node binds to an RPC port for JSON-RPC commands (e.g., `bitcoin-cli` connections). This isn't part of the P2P network binding test, so it's filtered out.

`rpc_port(i)` returns the RPC port for node `i` (assigned by the test framework).

### The Assertion

```python
assert_equal(binds, set(expected_services))
```

After filtering, the remaining bindings should exactly match the expected P2P bindings from `self.expected[i][1]`.

**Example for Node 1:**
- Expected: `{(0x7f000001, 18446), (0x7f000001, 18447)}`
- Actual (from `/proc`): `{(0x7f000001, 18446), (0x7f000001, 18447)}`
- Result: Pass ✓

If Bitcoin Core failed to bind to one of the requested ports, or bound to an extra unexpected port, this assertion would fail.

---

## The Three Test Cases in Detail

### Node 0: `-bind=127.0.0.1:port=onion` (Tor-only)

**Configuration:**
```
-bind=127.0.0.1:18444=onion
```

**Expected behavior:**
- Bitcoin Core should bind to exactly one port: `127.0.0.1:18444`
- That port should be designated as the Tor target
- No clearnet P2P port should be opened (since no non-`=onion` `-bind` was specified)

**What this tests:**
- When **only** a Tor binding is specified, Bitcoin Core doesn't silently add a default clearnet binding
- The `=onion` suffix is correctly parsed and doesn't prevent the binding from occurring

**Potential bug this catches:**
If Bitcoin Core had logic like:
```cpp
if (onion_bind_specified && !regular_bind_specified) {
    // ERROR: can't have Tor without clearnet, ignore the bind
    return;
}
```
Node 0 would fail to bind to any ports, and the test would catch it.

### Node 1: Both clearnet and Tor

**Configuration:**
```
-bind=127.0.0.1:18446
-bind=127.0.0.1:18447=onion
```

**Expected behavior:**
- Two bindings: `127.0.0.1:18446` (clearnet) and `127.0.0.1:18447` (Tor)
- Clearnet peers connect to `:18446`
- Tor peers connect to `:18447`

**What this tests:**
- Multiple `-bind` arguments are correctly processed
- The `=onion` suffix doesn't interfere with the standard binding
- Both bindings can coexist on the same interface (just different ports)

**Potential bug this catches:**
If Bitcoin Core's bind logic had a bug like:
```cpp
if (bind_addr_has_onion_flag) {
    // Replace the previous binding instead of adding a new one
    bindings[0] = onion_binding;
}
```
Node 1 would only bind to one port instead of two, and the test would fail.

### Node 2: Clearnet-only

**Configuration:**
```
-bind=127.0.0.1:18448
```

**Expected behavior:**
- One binding: `127.0.0.1:18448`
- No Tor target (since no `=onion` flag was used)

**What this tests:**
- Standard `-bind` behavior works as expected
- No unexpected extra bindings are created

**Potential bug this catches:**
If Bitcoin Core automatically added a Tor binding even when not requested:
```cpp
// Bug: always add a Tor binding on port+1
if (bind_specified) {
    bind(addr, port);
    bind(addr, port + 1);  // Oops, this is wrong
}
```
Node 2 would have two bindings instead of one, and the test would fail.

---

## The C++ Implementation

### How Bitcoin Core Processes `-bind`

In `src/init.cpp`, Bitcoin Core parses command-line arguments during startup:

```cpp
// src/init.cpp - AppInitParameterInteraction()
if (gArgs.IsArgSet("-bind")) {
    for (const std::string& strBind : gArgs.GetArgs("-bind")) {
        // Parse the bind string: "addr:port" or "addr:port=onion"
        std::string addr_str = strBind;
        ConnectionDirection bind_type = ConnectionDirection::None;

        // Check for "=onion" suffix
        size_t onion_pos = strBind.find("=onion");
        if (onion_pos != std::string::npos) {
            bind_type = ConnectionDirection::In; // Tor incoming
            addr_str = strBind.substr(0, onion_pos);
        }

        // Parse the address and port
        CService bind_addr;
        if (!Lookup(addr_str, bind_addr, /*default_port=*/0, /*allow_lookup=*/false)) {
            return InitError(strprintf("Cannot resolve -bind address: '%s'", addr_str));
        }

        // Add to the bind list
        if (bind_type == ConnectionDirection::In) {
            // This is the Tor target
            m_onion_binds.push_back(bind_addr);
        } else {
            // Standard clearnet binding
            m_binds.push_back(bind_addr);
        }
    }
}
```

The key distinction:
- **Without `=onion`:** Added to `m_binds` (standard P2P)
- **With `=onion`:** Added to `m_onion_binds` (Tor target)

### Actually Binding the Sockets

Later in initialization, Bitcoin Core calls `Bind()` for each address:

```cpp
// src/net.cpp - CConnman::Start()
bool CConnman::Start(/* ... */) {
    // Bind to standard P2P addresses
    for (const CService& addr : m_binds) {
        if (!BindListenPort(addr, /* is_onion= */ false)) {
            return false;
        }
    }

    // Bind to Tor target addresses
    for (const CService& addr : m_onion_binds) {
        if (!BindListenPort(addr, /* is_onion= */ true)) {
            return false;
        }
    }

    return true;
}
```

`BindListenPort()` calls the OS-level `bind()` syscall:

```cpp
bool CConnman::BindListenPort(const CService& addr, bool is_onion) {
    int sockfd = socket(addr.IsIPv4() ? AF_INET : AF_INET6, SOCK_STREAM, IPPROTO_TCP);

    // Set socket options (SO_REUSEADDR, etc.)
    // ...

    // Bind to the address
    if (bind(sockfd, (struct sockaddr*)&sockaddr, len) < 0) {
        LogPrintf("ERROR: Unable to bind to %s on this computer (bind returned error %s)\n",
                  addr.ToString(), NetworkErrorString(WSAGetLastError()));
        return false;
    }

    // Listen for connections
    if (listen(sockfd, SOMAXCONN) < 0) {
        LogPrintf("ERROR: Listening for incoming connections failed (listen returned error %s)\n",
                  NetworkErrorString(WSAGetLastError()));
        return false;
    }

    // Store the socket for later use
    if (is_onion) {
        m_onion_listen_sockets.push_back(sockfd);
    } else {
        m_listen_sockets.push_back(sockfd);
    }

    return true;
}
```

After this, the OS routes incoming TCP connections on those addresses/ports to Bitcoin Core.

### How Tor Connections Use the Onion Binding

When Bitcoin Core wants to make an outbound Tor connection, it uses a SOCKS5 proxy (the local Tor daemon). But for **incoming** Tor connections (hidden services), the Tor daemon forwards connections to the local address/port specified by `-bind=...=onion`.

Example Tor hidden service configuration (`/var/lib/tor/hidden_service/hostname`):
```
HiddenServiceDir /var/lib/tor/bitcoin-service/
HiddenServicePort 8333 127.0.0.1:18447
```

This tells Tor: "Forward connections to `<your-onion-address>.onion:8333` to `127.0.0.1:18447`."

Bitcoin Core's `-bind=127.0.0.1:18447=onion` ensures that port is open and listening.

---

## Why This Test Only Runs on Linux

The test uses Linux-specific `/proc` filesystem queries:

```python
def get_bind_addrs(pid):
    with open(f'/proc/{pid}/net/tcp', 'r') as f:
        # Parse the file...
```

**macOS equivalent:** `lsof -Pan -p <pid> -i`
**Windows equivalent:** `netstat -ano | findstr <pid>`

Each platform has different output formats, making cross-platform support complex. Rather than maintaining three implementations, the test restricts itself to Linux.

In CI, this test runs on Linux builders but is skipped on macOS/Windows.

---

## Why This Test Matters

### 1. Network Configuration Correctness

If Bitcoin Core claims to bind to a port but doesn't actually do so, connections will fail silently. Users will report "my node isn't receiving connections," and debugging will be difficult because the configuration **looks** correct.

This test verifies: **what you configure is what you get**.

### 2. Tor Integration

Tor is a critical privacy feature for Bitcoin users. If the `=onion` binding is broken:
- Hidden services won't work
- Users lose privacy (forced to use clearnet)
- Tor-only setups fail entirely

Testing the `=onion` flag separately from standard bindings ensures both paths work.

### 3. Avoiding Port Conflicts

If Bitcoin Core incorrectly binds to extra ports (e.g., both `:8333` and `:8334` when only `:8333` was requested), it could:
- Conflict with other software on the same machine
- Expose unexpected network services
- Create security vulnerabilities

The test ensures **only** the requested ports are bound.

### 4. Regression Prevention

Network binding logic is low-level code that rarely changes, but when it does, regressions can be subtle. A refactor that accidentally breaks multi-bind support might not be caught without this test.

---

## Comparison with Related Tests

| Test | What it tests | Focus |
|------|--------------|-------|
| `feature_bind_extra.py` | **Multiple bindings + Tor flag** | Port binding correctness |
| `feature_bind_port_discover.py` | **Port discovery** | `-port` vs. `-bind` interaction |
| `feature_bind_port_externalip.py` | **External IP announcement** | What IP is advertised to peers |
| `feature_proxy.py` | **SOCKS5 proxy for outbound Tor** | Tor **outbound** connections |
| `p2p_i2p_*.py` | **I2P network integration** | Alternative anonymity network |

`feature_bind_extra.py` is narrowly focused: **does Bitcoin Core bind to the ports you tell it to, especially when Tor is involved?**

---

## Key Takeaways

1. **`-bind` controls which network interfaces/ports Bitcoin Core listens on** — not just a suggestion, but a binding commitment to the OS

2. **The `=onion` suffix designates a binding as the Tor target** — incoming Tor connections (via hidden service) route through that port

3. **Multiple bindings can coexist** — clearnet and Tor can use separate ports on the same interface

4. **The test uses `/proc` to verify actual bindings** — not just trusting Bitcoin Core's logs, but querying the OS kernel directly

5. **Three test cases cover the configuration space** — Tor-only, clearnet+Tor, clearnet-only

6. **Platform-specific testing is sometimes necessary** — the `/proc` approach is Linux-only, and that's acceptable for coverage

7. **Network configuration correctness is critical** — mismatched expectations (config says port X, but bound to port Y) cause hard-to-debug failures

---

## Conclusion

`feature_bind_extra.py` tests a fundamental promise of any network service: **when you configure it to listen on specific ports, it actually does so**.

By querying the operating system directly (via `/proc` on Linux), the test bypasses Bitcoin Core's logging and verifies the ground truth: which TCP sockets are actually open, which ports are bound, and whether the configuration matches expectations.

The three-node design cleanly covers the configuration space: Tor-only (node 0), dual clearnet+Tor (node 1), and clearnet-only (node 2). Each case tests a different code path in Bitcoin Core's bind logic, ensuring that both standard and Tor-specific bindings work correctly, independently and together.

For developers, this test demonstrates an important testing principle: **don't just test the API, test the observable outcome**. It's not enough to call `bind()` and assume it worked — query the system afterward to confirm the socket actually exists. In distributed systems, networked applications, and anywhere configuration meets implementation, this verification pattern prevents silent failures and ensures your software does what users expect.

---

## References

- **Test file:** `test/functional/feature_bind_extra.py`
- **Network utilities:** `test_framework/netutil.py` — `get_bind_addrs()`, `addr_to_hex()`
- **Bitcoin Core init:** `src/init.cpp` — `-bind` argument parsing
- **Network manager:** `src/net.cpp` — `CConnman::BindListenPort()`
- **Tor documentation:** https://2019.www.torproject.org/docs/tor-manual.html.en#HiddenServicePort
