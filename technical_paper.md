# Breaking Tests on Purpose: A Learning Journey Through Bitcoin Core's Architecture

**Author:** Elizabeth Wandia Mugo
**Date:** February 2026

## Abstract

Understanding large codebases can feel like being handed a map of an unfamiliar city with no street names. This paper documents my journey through Bitcoin Core's 500,000+ lines of C++ code with a deceptively simple task: make exactly one functional test fail. What started as an intimidating assignment became a masterclass in surgical code navigation, test-driven development, and the power of minimal changes. I share not just the technical solution, but the false starts, the breakthrough moments, and the lessons learned about confidence in tackling systems that seem too big to comprehend. This is a story about learning to break things intentionally—a skill that's often harder for women developers who've been conditioned to seek perfection.

## 1. Introduction

The email with the assignment was short: "Make one functional test fail. Don't modify test files. Good luck."

I stared at my screen. I'd heard of Bitcoin Core. I knew it was important, complex, production-grade code used by thousands of nodes worldwide. What I didn't know was where to even start. Opening the repository felt like walking into a library where all the books were in a language I only half-spoke.

There's a specific kind of imposter syndrome that hits when you're facing a massive codebase. It's the voice that says, "Real developers would know exactly where to look." It's the pressure to appear confident when you're actually googling "what is RPC" for the third time. And if you're a woman in tech, there's an extra layer: the fear that asking for help will confirm someone's bias that you don't belong here.

But here's what I learned: you don't need to understand everything. You just need to find one thread and pull.

This paper documents how I went from overwhelmed to making a precise, two-line change that broke exactly one test out of hundreds. More importantly, it's about the process of learning to navigate complexity without letting perfectionism paralyze you.

### 1.1 The Challenge

The assignment had elegant constraints:
- Modify only `.cpp` or `.h` files in the `src/` directory
- Make exactly ONE functional test fail
- Keep all other tests passing
- Build and verify the solution

These constraints weren't arbitrary. They forced me to:
1. Understand how Bitcoin Core's test framework works
2. Trace backwards from a test to its implementation
3. Make a surgical change, not a sledgehammer fix
4. Prove my solution worked

### 1.2 Why This Matters

In my experience, women developers are often excellent at careful, precise work—but we're sometimes hesitant to break things. We've been socialized to color inside the lines, to not make mistakes, to be *certain* before we act. This exercise flipped that script. The goal was destruction. Controlled, intentional, minimal destruction.

There's something liberating about permission to break things. It removes the pressure of perfection and replaces it with curiosity: "What happens if I change this?"

## 2. Methodology: The Detective Work

### 2.1 First Steps (Or: Staring at Code for Two Hours)

My first instinct was to run the tests. I needed to see what "passing" looked like before I could break anything.

```bash
cd bitcoin
cmake -B build
cmake --build build -j$(sysctl -n hw.ncpu)
```

And then... I waited. The first build took 47 minutes on my laptop. I made tea. I answered emails. I questioned my life choices.

**Lesson learned:** Build time is thinking time. Instead of feeling guilty about "not being productive," I used it to read the test framework documentation. Those 47 minutes turned into research time.

When the build finally completed, I ran all the tests:

```bash
build/test/functional/test_runner.py
```

Watching hundreds of tests pass felt surreal. Each one represented some aspect of Bitcoin functionality that developers had carefully implemented and verified. And I was about to break one of them.

### 2.2 Choosing a Target (The Analysis Paralysis Phase)

With over 300 functional tests, how do you choose? I spent an embarrassing amount of time overthinking this.

My first approach was to pick something "impressive." Maybe break a complex consensus test? Something with cryptography? Then I caught myself: I was performing for an imaginary audience instead of solving the problem strategically.

I needed a test that:
- Was simple enough to understand quickly
- Had a clear, isolated implementation
- Wouldn't cascade into breaking other tests

I listed the tests and looked for simple names. `rpc_uptime.py` caught my eye. Uptime—how long a node has been running. That seemed... achievable?

**The moment of truth:** I opened `test/functional/rpc_uptime.py`. It was 42 lines. I almost cried with relief.

### 2.3 Reading the Test (The Aha Moment)

```python
def _test_uptime(self):
    wait_time = 10
    self.nodes[0].setmocktime(int(time.time() + wait_time))
    assert self.nodes[0].uptime() >= wait_time
```

This test:
1. Sets mock time 10 seconds in the future
2. Calls `uptime()` RPC method
3. Asserts uptime is at least 10 seconds

The assertion was my target: `assert self.nodes[0].uptime() >= wait_time`

If I could make `uptime()` return something less than 10, the test would fail.

**Here's where the impostor syndrome kicked in:** I thought, "This seems too simple. Surely I'm missing something?" I spent another hour looking for hidden complexity before realizing that no, sometimes the simple answer is the right answer.

Women developers: we do this to ourselves. We doubt the simple solution because we assume we must be missing something sophisticated that "real" developers would see. Sometimes the simple answer is correct.

### 2.4 The Hunt (Grep Is Your Friend)

Now I needed to find where `uptime()` was implemented. I started with grep:

```bash
grep -r "uptime" src/ --include="*.cpp" --include="*.h"
```

Dozens of results. Most were comments or unrelated. But one stood out:

`src/rpc/server.cpp`: A file that registered RPC commands.

I opened it. Line 187. There it was:

```cpp
static RPCHelpMan uptime()
{
    return RPCHelpMan{"uptime",
        "\nReturns the total uptime of the server.\n",
        {},
        RPCResult{
            RPCResult::Type::NUM, "", "The number of seconds that the server has been running"},
        RPCExamples{
            HelpExampleCli("uptime", "")
            + HelpExampleRpc("uptime", "")
        },
        [&](const RPCHelpMan& self, const JSONRPCRequest& request) -> UniValue
        {
            return GetTime() - GetStartupTime();
        },
    };
}
```

**The lambda function at the bottom was the actual implementation.** It calculated uptime as `GetTime() - GetStartupTime()`. Simple, elegant, correct.

And I was about to break it.

## 3. The Technical Solution

### 3.1 Understanding the Implementation Chain

The code flow looked like this:

1. **Test Layer** (`test/functional/rpc_uptime.py`): Python test calls `self.nodes[0].uptime()`
2. **RPC Interface** (JSON-RPC over HTTP): Routes the call to the handler
3. **RPC Handler** (`src/rpc/server.cpp:189-191`): Lambda function that calculates uptime
4. **Time Functions** (`src/util/time.cpp`, `src/common/system.cpp`): `GetTime()` and `GetStartupTime()`

To break the test surgically, I needed to intervene at the RPC handler level. Modifying `GetTime()` or `GetStartupTime()` would be too broad—those functions were used everywhere.

### 3.2 The Modification

**File:** `src/rpc/server.cpp`
**Lines:** 189-191

**Original code:**
```cpp
[&](const RPCHelpMan& self, const JSONRPCRequest& request) -> UniValue
{
    return GetTime() - GetStartupTime();
}
```

**Modified code:**
```cpp
[&](const RPCHelpMan& self, const JSONRPCRequest& request) -> UniValue
{
    // return GetTime() - GetStartupTime();
    return 0;
}
```

That's it. Two lines. Comment out the correct calculation, return 0 instead.

### 3.3 Why This Works

The test expects: `uptime() >= 10` (at least 10 seconds)
My modified code returns: `0`
The assertion fails: `0 >= 10` is false

**Why only this test fails:**
- The `uptime` RPC command is only tested in `rpc_uptime.py`
- No other tests depend on the actual uptime value
- The change is isolated to this single RPC endpoint

### 3.4 The Rebuild and Verification

I modified the code, took a breath, and rebuilt:

```bash
cmake --build build -j$(sysctl -n hw.ncpu)
```

Only took 3 minutes this time—incremental builds are beautiful.

Then the moment of truth:

```bash
build/test/functional/test_runner.py rpc_uptime.py
```

```
FAILED: rpc_uptime.py
```

Success! (The weird kind where failure is success.)

Now for the real test—did I break ONLY that one test?

```bash
build/test/functional/test_runner.py
```

I watched the test runner scroll through hundreds of tests. Green checkmarks. Green checkmarks. Then:

```
rpc_uptime.py ✗
```

And then... more green checkmarks.

I let out a breath I didn't know I was holding.

## 4. Results and Discussion

### 4.1 The Solution in Context

My solution worked, but was it the *best* solution? I considered alternatives:

**Alternative 1:** Return -1
- **Pro:** Also fails the test
- **Con:** Negative uptime is logically weird

**Alternative 2:** Return `wait_time - 1` (9 seconds)
- **Pro:** Closer to correct, still fails
- **Con:** Too clever; harder to understand why it's wrong

**Alternative 3:** Return a very large number
- **Pro:** Obviously wrong
- **Con:** Fails the test in a confusing way

**My choice (return 0):**
- **Pro:** Dead simple; obviously incorrect; clear intent
- **Con:** None

I chose the simplest solution. In the past, I might have chosen something more "clever" to prove I was smart enough. But I've learned that simplicity is a feature, not a bug. The best code is code that the next person (or future you) can understand at 2am.

### 4.2 What I Learned About Bitcoin Core

**Architecture insights:**
- Bitcoin Core uses the `RPCHelpMan` pattern for defining RPC commands
- RPC handlers are lambda functions that receive `JSONRPCRequest` and return `UniValue`
- Time functions are mockable for deterministic testing (`setmocktime`)
- The codebase has clear separation between layers (RPC, implementation, tests)

**Testing philosophy:**
- Each test is independent and isolated
- Tests use real `bitcoind` instances (not mocks)
- The framework supports hundreds of tests running in parallel
- Tests are written in Python for accessibility

### 4.3 What I Learned About Myself

**On perfectionism:** I wasted hours second-guessing simple solutions. The pressure to be "impressive" almost led me to over-complicate the problem. Learning to trust the simple answer is an ongoing process.

**On asking for help:** I didn't ask for help on this assignment—I wanted to prove I could do it alone. In retrospect, discussing approaches with peers would have been faster and more educational. Lone wolf syndrome is real, and it's not serving me.

**On confidence with large codebases:** The intimidation factor of 500k+ lines of code is real, but it's also... a lie? Once I stopped trying to understand *everything* and focused on tracing *one path*, the problem became manageable. I don't need to know the entire Bitcoin protocol to modify one RPC handler.

**On breaking things:** There was a moment when I hesitated before changing the code. "What if I break something important?" But that's exactly the point—in a controlled environment with tests, breaking things is how you learn. The fear of making mistakes is learned behavior, and it can be unlearned.

### 4.4 Broader Implications for Learning

This exercise exemplifies what I call **"intentional destruction as pedagogy."** Instead of building something new (where you might cargo-cult patterns without understanding them), breaking something forces you to:

1. **Understand what something does** (read the test)
2. **Trace the implementation** (find the code)
3. **Understand dependencies** (why won't this break other things?)
4. **Verify your mental model** (did it break in the way you expected?)

It's a form of reverse engineering that's deeply educational.

## 5. Conclusion

When I started this assignment, I thought success would mean finding the right line of code to change. But the real success was learning to navigate complexity without fear.

To the women developers reading this: the impostor syndrome never fully goes away, but you can learn to recognize it for what it is—a liar that wants to keep you small. That voice that says "you're not good enough" is wrong. You are good enough to read complex code. You are good enough to break things and fix them. You are good enough to contribute to systems you don't fully understand yet.

The key insights from this exercise:

1. **You don't need to understand everything.** Find one thread and pull.
2. **Simple solutions are valid.** Don't over-engineer to prove yourself.
3. **Breaking things is learning.** Give yourself permission to fail in controlled environments.
4. **Build time is thinking time.** Don't guilt yourself about "waiting."
5. **The intimidation is learned.** Large codebases are just many small pieces.

### 5.1 Final Thoughts

I made a two-line change that broke one test out of hundreds. On the surface, that seems trivial. But the journey to those two lines taught me more about Bitcoin Core, test-driven development, and my own capabilities than any tutorial could have.

The assignment was about breaking a test. The real lesson was about breaking through self-doubt.

If you're facing a large codebase that feels overwhelming, remember: someone else felt that way too. The developers who built Bitcoin Core didn't have perfect knowledge when they started. They pulled threads, made mistakes, asked questions, and learned.

You can too.

---

## Acknowledgments

Thank you to the Bitcoin Core developers for creating such a well-structured codebase with comprehensive tests. The clarity of the RPC layer made this exercise possible.

Thank you to the women developers who've shared their own stories of impostor syndrome and persistence. Seeing your journeys gave me permission to share mine.

## References

1. Bitcoin Core repository: https://github.com/bitcoin/bitcoin
2. Bitcoin Core functional test framework documentation
3. Bitcoin Core RPC documentation
4. My commit: `c9e9d2f` - "make rpc_uptime.py fail"

---

## Appendix A: The Commit

```
commit c9e9d2fee433f6055fb510792eea91a449aeb6a2
Author: Elizabeth Wandia Mugo <wwandia173@gmail.com>
Date:   Tue Dec 30 16:25:52 2025 +0300

    make rpc_uptime.py fail
```

**Files changed:**
- `src/rpc/server.cpp`: Modified uptime RPC handler to return 0

**Test result:**
- `rpc_uptime.py`: ✗ FAILED
- All other tests: ✓ PASSED

## Appendix B: For Future Learners

If you're attempting this exercise, here are some tips:

1. **Start with simple tests.** Look for short test files with clear names.
2. **Use grep liberally.** Don't be afraid to search the entire codebase.
3. **Read the test first.** Understand what it's checking before hunting for code.
4. **Trace backwards.** Test → RPC call → RPC handler → Implementation.
5. **Make minimal changes.** The smaller the change, the less likely you'll break other things.
6. **Rebuild incrementally.** You don't need a full rebuild for small changes.
7. **Run the specific test first.** Verify it fails before running all tests.
8. **Document your process.** Future you will thank present you.

And most importantly: **Give yourself permission to not know.** Not knowing is the starting point of learning, not a character flaw.
