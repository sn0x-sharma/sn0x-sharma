---
icon: code
---

# CODING

## HTB Operation Blackout -Coding Challenges

**Challenges Covered:** Threat Index (Very Easy) | Triple Knock (Easy) | Honeypot (Easy) | Blackwire (Medium)

***

> #### These four challenges are all algorithmic  no web app to poke, no binary to reverse. The challenge is purely: read the problem, design an efficient solution, implement it cleanly. Each one is themed around a cybersecurity scenario (threat scoring, brute-force detection, network containment, logic bomb path counting) but the actual work is data structures and algorithms. The writeups below explain not just _what_ the solution is but _why_ the algorithm works and _why_ naive approaches would blow up on large inputs.

***

## Threat Index

**Difficulty:** Very Easy\
**Flag:** `HTB{thr34t_L3v3L_m1dn1ght_9b1ba6245ccc2a5aabd9166ee2ca6fa9}`

***

### The Problem

You get a single continuous string of lowercase letters and digits  simulating traffic pulled off a TOR exit node. Your job is to scan it for 18 predefined threat keywords, count how many times each appears, multiply by the keyword's weight, and output the total threat score. No spaces between words, no delimiters, just a raw character stream.

The 18 keywords and their weights:

```
scan(1)  response(2)  control(3)  callback(4)  implant(5)    zombie(6)
trigger(7)  infected(8)  compromise(9)  inject(10)  execute(11)  deploy(12)
malware(13)  exploit(14)  payload(15)  backdoor(16)  zeroday(17)  botnet(18)
```

***

### Why This Is Trivial

Input length goes up to 10⁶ characters, 18 keywords. Python's built-in `str.count()` is implemented in C under the hood and does a linear scan in O(N) per keyword. So total work is 18 × N roughly 18 million operations max. That's nothing. Runs in milliseconds.

```
┌──(sn0x㉿sn0x)-[~/HTB/ThreatIndex]
└─$ cat solve.py
```

```python
#!/usr/bin/env python3
import sys

def main():
    stream = sys.stdin.read().strip()

    weights = {
        "scan": 1,       "response": 2,   "control": 3,
        "callback": 4,   "implant": 5,    "zombie": 6,
        "trigger": 7,    "infected": 8,   "compromise": 9,
        "inject": 10,    "execute": 11,   "deploy": 12,
        "malware": 13,   "exploit": 14,   "payload": 15,
        "backdoor": 16,  "zeroday": 17,   "botnet": 18,
    }

    score = 0
    for keyword, weight in weights.items():
        count = stream.count(keyword)
        score += count * weight

    print(score)

if __name__ == "__main__":
    main()
```

```
┌──(sn0x㉿sn0x)-[~/HTB/ThreatIndex]
└─$ echo "payloadrandompayloadhtbzerodayrandombytesmalware" | python3 solve.py
```

```
60
```

Manual check: `payload` ×2 = 30, `zeroday` ×1 = 17, `malware` ×1 = 13. Total = 60. Correct.

***

### Why Not Regex? Why Not Aho-Corasick?

The source write-up mentions both as alternatives so let's actually talk about when you'd use them.

**Regex** (`re.findall` with `pattern1|pattern2|...`)  looks clean but has a trap: overlapping matches. If the stream contains `exploitexploit` and you're searching for `exploit`, `re.findall` by default won't double-count because matches are non-overlapping and consumed sequentially. For this specific problem that's fine. But regex carries overhead from the engine and for 18 static keywords it's genuinely overkill.

**Aho-Corasick**  this is the _correct_ industrial-strength solution for multi-pattern string search. It builds a finite automaton from all your keywords in O(sum of keyword lengths) time, then scans the input in O(N + total\_matches) time  one pass, all patterns simultaneously. For 18 short keywords and N=10⁶, it's faster in theory but the constant factor from building the automaton and the added dependency (`pyahocorasick` library) aren't worth it when `str.count()` × 18 already comfortably fits in the time limit. If keywords numbered in the thousands or N was 10⁸+, Aho-Corasick is the right call.

For this problem: `str.count()` loop wins on simplicity.

***

## Triple Knock

**Difficulty:** Easy\
**Flag:** `HTB{kn0ck_kN0cK_Kn0Ck_a5c56af1ef7b195bf92d7d39dfc8e63b}`

***

#### The Problem

You get S shuffled login log entries. Each entry has: user ID, timestamp (DD/MM HH:MM), and status (`[success]` or `[failure]`). Find every user who had **3 or more failed logins within any rolling 10-minute window**. Output flagged users in lexicographical order.

Constraints: S up to 10⁵, N (users) up to 200.

***

### The Core Algorithm - Sliding Window with Two Pointers

The moment you read "rolling window" in an algorithm problem, you should immediately think **two pointers**. Here's the mental model:

You have a sorted list of failure timestamps for a user. Put a left pointer `i` and a right pointer `j` both starting at index 0. Move `j` forward one step at a time. Every time you move `j`, check if `timestamps[j] - timestamps[i] > 10`. If it is, slide `i` forward until the window is ≤ 10 minutes again. At each position of `j`, the window `[i..j]` contains all failures within 10 minutes ending at `j`. If `j - i + 1 >= 3`, that's 3 or more failures in a 10-minute window flag the user.

This is O(N) per user after sorting, because both `i` and `j` only ever move forward they never backtrack. Total per user: O(F log F) for the sort + O(F) for the scan, where F is their failure count.

***

### Timestamp Conversion

Before the sliding window can work, timestamps need to be numbers. "23/07 15:41" has to become "minutes since some reference point". The formula:

```
minutes = ((month - 1) × 30 + (day - 1)) × 1440 + hours × 60 + minutes
```

Month is 1-indexed so subtract 1. Day same. Each day is 1440 minutes (24×60). This gives a single integer representing absolute minute position in the year. Now "is the gap ≤ 10 minutes?" is just `timestamps[j] - timestamps[i] <= 10`.

The challenge says all months = 30 days, which simplifies the calendar math — no leap year handling needed.

***

### Complete Solution

```
┌──(sn0x㉿sn0x)-[~/HTB/TripleKnock]
└─$ cat solve.py
```

```python
#!/usr/bin/env python3
import sys
from collections import defaultdict

def parse_ts(date, time):
    # "DD/MM" and "HH:MM" → absolute minutes
    d, m = map(int, date.split('/'))
    h, mi = map(int, time.split(':'))
    return ((m - 1) * 30 + (d - 1)) * 1440 + h * 60 + mi

def main():
    data = sys.stdin.read().split()
    it = iter(data)
    S, N = int(next(it)), int(next(it))

    # Collect all failure timestamps per user
    fails = defaultdict(list)
    for _ in range(S):
        user   = next(it)
        date   = next(it)
        time   = next(it)
        status = next(it).strip('[').strip(']')   # "[failure]" → "failure"
        if status == 'failure':
            fails[user].append(parse_ts(date, time))

    flagged = []
    for user, times in fails.items():
        times.sort()
        i = 0
        for j in range(len(times)):
            # Shrink window from left until it's ≤ 10 minutes
            while times[j] - times[i] > 10:
                i += 1
            # Window [i..j] is valid — check if it has 3+ failures
            if j - i + 1 >= 3:
                flagged.append(user)
                break   # No need to keep checking this user

    print(' '.join(sorted(flagged)))

if __name__ == "__main__":
    main()
```

```
┌──(sn0x㉿sn0x)-[~/HTB/TripleKnock]
└─$ python3 solve.py < sample.txt
```

```
user_1 user_3
```

***

### Tracing Through the Sample

For `user_1`, failures at: `10/06 05:17`, `10/06 05:19`, `10/06 05:25`. In minutes these are some values X, X+2, X+8. The window from X to X+8 is 8 minutes, which is ≤ 10. Three failures in that window. Flagged.

For `user_3`, failures at `20/04 13:53`, `20/04 13:54`, `20/04 13:59` (plus one more on a different day). The 13:53 to 13:59 window is 6 minutes, three failures inside it. Flagged.

`user_2` and `user_4` don't have 3 failures in any 10-minute window, so they're clean.

***

### Why Not Just Nest Loops?

Naive approach: for each failure, look at every subsequent failure and count how many are within 10 minutes. That's O(F²) per user. With S=10⁵ failures all on one user that's 10¹⁰ operations  dead. The two-pointer approach is O(F) per user after sorting. That's the difference between timing out and finishing in milliseconds.

***

## Honeypot

**Difficulty:** Easy\
**Flag:** `HTB{l34D_th3M_t0_th3_H0nEyP0t_52e33c525d28ba772f6e555d9bae2dce}`

***

### The Problem

You have a network modeled as a tree with N nodes (N up to 10⁶). Node 1 is the root (the compromised router). One node H is your honeypot. You can cut exactly one edge. After cutting, you want to minimize the number of nodes still in the same connected component as node 1, **but the honeypot H must stay connected to node 1**. Output the minimum reachable node count after the optimal cut.

***

### Why Trees Make This Easy

In a general graph, cutting one edge might not disconnect anything  there could be multiple paths between nodes. In a tree, there is **exactly one path** between any two nodes. Cut an edge and you always split the tree into exactly two components. No exceptions. This means:

* Every edge `(parent, child)` defines a potential cut that isolates the entire subtree rooted at `child`
* The two components after any cut are: subtree(child) and everything else
* We want to maximize the size of the subtree we cut away, subject to: the subtree must **not** contain the honeypot H

Why maximize what we cut? Because the question asks for the minimum remaining nodes = N - (maximum removed). So minimizing remaining = maximizing removed.

***

### The DFS Plan

Root the tree at node 1. For each node `u`, compute two things:

* `subtree_size[u]`  how many nodes are in the subtree rooted at `u` (including `u` itself)
* `contains_honeypot[u]`  does this subtree contain node H?

Then the answer is: find the node `u` (excluding root) where `contains_honeypot[u] == False` and `subtree_size[u]` is maximum. Cut the edge between `u` and its parent. Remaining nodes = N - subtree\_size\[u].

Both values can be computed in a single DFS pass (post-order: process children before parent).

***

### Iterative DFS - Why Recursive Won't Work Here

N goes up to 10⁶. A recursive DFS on a degenerate tree (basically a linked list: 1→2→3→...→10⁶) would hit 10⁶ stack frames. Python's default recursion limit is 1000. Even with `sys.setrecursionlimit(10**7)` you'd blow the process stack (not Python's soft limit, the actual OS thread stack). Iterative DFS using an explicit stack on the heap is the only safe approach for large N.

The trick for iterative post-order DFS is the "visited flag" pattern: push a node with `visited=False` first. When you pop it unvisited, push it again with `visited=True` and then push all its children. When you pop it visited, it means all children have been processed  now you can safely compute the parent's values from the children's already-computed values.

```
┌──(sn0x㉿sn0x)-[~/HTB/Honeypot]
└─$ cat solve.py
```

```python
#!/usr/bin/env python3
import sys

def main():
    data = sys.stdin.read().strip().split()
    it = iter(data)
    N = int(next(it))

    adj = [[] for _ in range(N + 1)]
    for _ in range(N - 1):
        a = int(next(it))
        next(it)          # skip the '-' separator
        b = int(next(it))
        adj[a].append(b)
        adj[b].append(a)

    H = int(next(it))     # honeypot node

    parent           = [0] * (N + 1)
    subtree_size     = [0] * (N + 1)
    contains_honey   = [False] * (N + 1)

    # Iterative post-order DFS from root = 1
    stack = [(1, 0, False)]   # (node, parent, visited)

    while stack:
        u, p, visited = stack.pop()

        if not visited:
            parent[u] = p
            stack.append((u, p, True))          # re-push as "to process after children"
            for v in adj[u]:
                if v != p:
                    stack.append((v, u, False))
        else:
            # Post-order: all children already processed
            sz  = 1
            has = (u == H)
            for v in adj[u]:
                if v != parent[u]:
                    sz  += subtree_size[v]
                    has  = has or contains_honey[v]
            subtree_size[u]   = sz
            contains_honey[u] = has

    # Find the largest subtree that doesn't contain the honeypot
    max_cut = 0
    for u in range(2, N + 1):        # skip root (can't cut above root)
        if not contains_honey[u]:
            if subtree_size[u] > max_cut:
                max_cut = subtree_size[u]

    print(N - max_cut)

if __name__ == "__main__":
    main()
```

```
┌──(sn0x㉿sn0x)-[~/HTB/Honeypot]
└─$ echo "7
1 - 2
1 - 3
3 - 4
3 - 5
1 - 6
6 - 7
5" | python3 solve.py
```

```
5
```

***

### Tracing the Sample

Tree rooted at 1:

```
        1
       /|\
      2  3  6
        / \   \
       4   5   7
           ^
        (honeypot H=5)
```

Subtree sizes: node 2 → size 1, node 4 → size 1, node 5 → size 1, node 3 → size 3 (nodes 3,4,5), node 6 → size 2 (nodes 6,7), node 7 → size 1.

`contains_honey`: node 5 → yes (it IS H), node 3 → yes (subtree contains 5), node 4 → no, node 2 → no, node 6 → no, node 7 → no.

Valid cuts (subtrees not containing honeypot): node 2 (size 1), node 4 (size 1), node 6 (size 2), node 7 (size 1).

Maximum valid cut: subtree rooted at node 6, size 2. Cut edge (1-6).

Remaining: 7 - 2 = 5. Correct.

***

## Blackwire

**Difficulty:** Medium\
**Flag:** `HTB{f1n4l_st4t3_th3_b0mB_1s_4rm3d_7bdbf012642329cf8397ffc212f35e13}`

***

### The Problem

A firmware update contains a logic bomb implemented as a Finite State Machine. The FSM has states 0 through T. You're given:

* A **transition table**: T entries, each saying "from state S, opcode X advances you to state S+1"
* An **opcode stream**: a sequence of L/8 opcodes

When the FSM sees an opcode that matches its current transition rule, it has the **option** to advance — not a requirement. You need to count the total number of distinct ways to advance from state 0 all the way to state T by choosing a subset of matching opcodes in order.

Input is raw bits, not characters. T up to 4000, opcode stream up to 80000 bits (10000 opcodes).

***

### What's Actually Being Asked

Let me rephrase this more concretely because the problem statement is dense.

Imagine the opcode stream is a list of numbers. Each transition rule says "to go from state S to state S+1, you need to see opcode X\[S]". You scan through the opcode stream left to right. Every time you see the right opcode for your current state, you _can_ choose to take that transition. The question is: in how many different ways can you go 0→1→2→...→T using the opcodes in order?

This is a classic "count the number of ordered subsequences" problem. The word "ordered" is key you pick opcodes from left to right, and each chosen opcode for transition S has to come _after_ the opcode you chose for transition S-1.

***

### Dynamic Programming - Why It Works

Define `dp[s]` = number of distinct ways to reach state `s` given the opcodes processed so far.

Start: `dp[0] = 1` (one way to be at the start), everything else 0.

For each opcode in the stream, check if it triggers any transition. If opcode `op` triggers the transition from state `s` to `s+1` (i.e., `transitions[s] == op`), then every way we previously reached state `s` just became an additional way to reach state `s+1`. So: `dp[s+1] += dp[s]`.

Process this update **backwards** (from `s = T-1` down to `s = 0`). Why backwards? To avoid using a single opcode for multiple transitions in the same step. If you go forward, you might update `dp[1]` and then immediately use that updated value to update `dp[2]`  that would be using the same opcode to advance two states, which isn't allowed. Going backwards, by the time you process state `s`, you haven't yet touched `dp[s]` in this iteration, so you're always using the value from before the current opcode was processed.

This is the same pattern as the classic 0/1 knapsack DP where you process the weight array backwards to prevent "using the same item twice".

***

### Parsing the Binary Input

The transition table entries are 20 bits each: first 12 bits = current state S, last 8 bits = required opcode. The opcode stream is 8 bits per opcode. All input arrives as a bit string.

```
┌──(sn0x㉿sn0x)-[~/HTB/Blackwire]
└─$ cat solve.py
```

```python
#!/usr/bin/env python3
import sys

def main():
    data = sys.stdin.read().split()
    T, L = int(data[0]), int(data[1])

    # Concatenate all bit chunks into one string
    bits = "".join(data[2:])

    # Split: first T*20 bits are the transition table, next L bits are opcode stream
    table_bits  = bits[:20 * T]
    stream_bits = bits[20 * T : 20 * T + L]

    # Parse transition table
    # Entry i: bits [20i .. 20i+12) = state, bits [20i+12 .. 20i+20) = opcode
    transitions = [None] * T
    for i in range(T):
        entry  = table_bits[20*i : 20*(i+1)]
        state  = int(entry[:12], 2)
        opcode = int(entry[12:], 2)
        transitions[state] = opcode

    # Decode opcode stream: every 8 bits is one opcode
    opcodes = [int(stream_bits[i:i+8], 2) for i in range(0, L, 8)]

    # DP: dp[s] = number of distinct ways to reach state s
    dp = [0] * (T + 1)
    dp[0] = 1

    for op in opcodes:
        # Backwards to avoid using same opcode for multiple transitions in one step
        for s in range(T - 1, -1, -1):
            if transitions[s] == op:
                dp[s + 1] += dp[s]

    print(dp[T])

if __name__ == "__main__":
    main()
```

```
┌──(sn0x㉿sn0x)-[~/HTB/Blackwire]
└─$ python3 solve.py < sample.txt
```

```
4
```

***

### Tracing a Mini Example

Say T=2 (states 0,1,2). Transition table: state 0 needs opcode A, state 1 needs opcode B.\
Opcode stream: `[A, B, A, B]`.

Process opcode A (position 0): `dp[1] += dp[0]` → `dp = [1, 1, 0]`\
Process opcode B (position 1): `dp[2] += dp[1]` → `dp = [1, 1, 1]`\
Process opcode A (position 2): `dp[1] += dp[0]` → `dp = [1, 2, 1]`\
Process opcode B (position 3): `dp[2] += dp[1]` → `dp = [1, 2, 3]`

So `dp[2] = 3`  three distinct paths to reach the final state. The three paths correspond to which A and which B you chose: (A₁,B₂), (A₁,B₄), (A₃,B₄). Makes sense.

***

### Complexity

* Parsing: O(T + L/8)
* Main DP loop: O((L/8) × T) = O(10000 × 4000) = O(40 million) at worst

40 million simple array operations in Python is on the edge  might need PyPy or optimizing the inner loop with numpy if the judge is strict. The key optimization: skip the inner state loop entirely when no state maps to the current opcode (most opcodes). In practice, with T=4000 distinct possible states but only 256 possible opcode values, there are lots of opcodes that don't trigger any transition at all. Can precompute a dict from opcode → list of states that care about it to skip irrelevant iterations.

```python
# Optimized: precompute which states each opcode triggers
from collections import defaultdict
opcode_to_states = defaultdict(list)
for s, op in enumerate(transitions):
    if op is not None:
        opcode_to_states[op].append(s)

for op in opcodes:
    for s in reversed(opcode_to_states[op]):   # only states that care about this opcode
        dp[s + 1] += dp[s]
```

This prunes the inner loop significantly when opcode coverage is sparse and with 8-bit opcodes (256 values) and T up to 4000 transitions, collisions are likely but the optimization still helps on sparse streams.

***

### Attack Chain

```
Threat Index
═════════════
Read stream → loop 18 keywords → str.count() each → weighted sum → score


Triple Knock  
═════════════
Parse logs → filter failures per user → sort timestamps
        ↓
Two-pointer sliding window over sorted failure times
        ↓
window[i..j] ≤ 10 min AND size ≥ 3 → flag user
        ↓
Lexicographic sort → output


Honeypot
═════════
Parse tree → iterative post-order DFS from root
        ↓
Compute subtree_size[u] and contains_honeypot[u] for all nodes
        ↓
Find max subtree_size[u] where contains_honeypot[u] == False (u ≠ root)
        ↓
Answer = N - max_cut


Blackwire
═════════
Parse bits → transition table (12-bit state, 8-bit opcode) + opcode stream
        ↓
dp[0] = 1, all others = 0
        ↓
For each opcode: backwards update dp[s+1] += dp[s] for matching states
        ↓
dp[T] = number of distinct activation paths → FLAG
```

***

### Techniques I Used

| Technique                                          | Where Used     |
| -------------------------------------------------- | -------------- |
| Weighted substring frequency counting              | Threat Index   |
| `str.count()` O(N) per keyword                     | Threat Index   |
| Aho-Corasick multi-pattern matching (alternate)    | Threat Index   |
| Timestamp normalization to absolute minutes        | Triple Knock   |
| Sliding window two-pointer on sorted timestamps    | Triple Knock   |
| Lexicographic output sorting                       | Triple Knock   |
| Tree rooting and post-order DFS                    | Honeypot       |
| Iterative DFS with visited-flag pattern            | Honeypot       |
| Subtree size + honeypot containment propagation    | Honeypot       |
| Maximum subproblem with constraint (no honeypot)   | Honeypot       |
| Binary bit string parsing (12-bit + 8-bit fields)  | Blackwire      |
| Dynamic programming — ordered subsequence counting | Blackwire      |
| Backwards DP update (0/1 knapsack pattern)         | Blackwire      |
| Opcode-to-state precomputation for loop pruning    | Blackwire      |
| rustscan port enumeration                          | All challenges |

***
