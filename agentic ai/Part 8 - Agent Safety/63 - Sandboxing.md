

These notes fill in concepts that are commonly expected alongside Sandboxing but were missing from the base chapter. Same style as the original — add this as extra sections, don't replace anything.

---

# Isolation Strength Spectrum

The base chapter lists four Sandbox types but doesn't compare their actual isolation strength or cost.

```text
Weakest Isolation, Lowest Cost

↓

Process Sandbox
 -> shares the same kernel, easiest to escape

↓

Container Sandbox
 -> shares the host kernel, isolated filesystem/network

↓

MicroVM Sandbox
 -> own lightweight kernel, container-like speed
    (e.g. Firecracker, gVisor)

↓

Full Virtual Machine
 -> completely separate kernel and hardware emulation

↓

Strongest Isolation, Highest Cost
```

MicroVMs are worth calling out on their own — they're the tier many AI code-execution products actually use, because they give near-VM isolation without full VM overhead.

---

# Sandbox Escape

The base chapter treats the Sandbox boundary as absolute. In practice, it can be broken.

```text
Sandbox Escape

↓

Malicious or buggy code exploits a flaw
in the isolation layer itself

↓

Code gains access OUTSIDE the sandbox
 (host filesystem, other tenants, network)
```

This is why Sandboxing is described as containing damage, not eliminating risk. Mitigations include:

```text
Keep sandbox runtimes patched and updated

Use stronger isolation (microVM/VM) for higher-risk workloads

Never treat the sandbox as the ONLY safeguard
 -> combine with Guardrails and Least-Privilege
```

---

# Multi-Tenant Isolation

The base chapter frames Sandboxing as protecting the production system from the Agent. There's a second boundary that matters just as much.

```text
Single-Tenant Framing

↓

Sandbox protects: Host <- Agent
```

```text
Multi-Tenant Framing

↓

Sandbox protects: User A's Agent <-> User B's Agent

↓

Two different users' code must never
 be able to see or affect each other
```

Platforms running many users' Agents need isolation **between** sandboxes, not just between each sandbox and the host.

---

# Concrete Resource Limits

The base chapter mentions CPU/Memory as things a Sandbox "may restrict" but doesn't show what enforcing that looks like.

```text
Typical Sandbox Limits

↓

CPU     -> e.g. 1 core max
Memory  -> e.g. 512 MB max
Disk    -> e.g. 100 MB, wiped on destroy
Timeout -> e.g. kill after 30 seconds
Network -> e.g. zero outbound access by default
```

Without explicit limits, a single Agent task (infinite loop, memory leak, runaway recursion) can exhaust shared resources even inside an "isolated" sandbox.

---

# Network Egress Allowlisting

"Restrict network access" deserves a concrete default.

```text
Default

↓

NO outbound network access at all

↓

Explicitly Allow

↓

Only the specific APIs the task needs
 (e.g. flight-search-api.example.com)
```

This prevents two things at once:

```text
Data exfiltration
 -> sandboxed code can't phone home with stolen data

Unauthorized external calls
 -> sandboxed code can't reach unintended services
```

Default-deny network access is the same principle as Least- Privilege, applied at the network layer.

---

# Filesystem Isolation Details

A more concrete version of "limit access to files."

```text
Read-Only Mounts

↓

Sandbox can READ certain reference files,
 but cannot modify them

↓

Ephemeral Storage

↓

Any files the sandbox WRITES exist only
 for the sandbox's lifetime

↓

Destroyed with the sandbox, nothing persists
```

This matters because "destroy the sandbox after execution" (from the base workflow) only actually protects data if writes never leave the ephemeral layer in the first place.

---

# Syscall Filtering

A lower-level technique that container/VM isolation is often paired with.

```text
Full OS Syscall Access

↓

Sandbox code can request ANY operating system operation
 (even if containerized)
```

```text
Syscall Filtering (e.g. seccomp)

↓

Only an explicit allowlist of syscalls is permitted

↓

Dangerous syscalls (mount, ptrace, reboot, etc.)
 are blocked at the kernel level
```

This adds a layer of protection even if the container/process isolation itself has a flaw.

---

# Don't Trust Sandbox Output Blindly

A gap between this chapter and the previous one worth closing explicitly.

```text
Sandbox Isolates EXECUTION

↓

It does NOT validate the RESULT

↓

Sandboxed code can still return:
 incorrect output, hallucinated data,
 or a maliciously crafted response
```

The Sandbox's job is containment — Output Guardrails (previous chapter) still need to inspect whatever the sandbox returns before it's used or shown to a user.

---

# Secrets Injection at Runtime

The base chapter's Mistake 3 says "don't share secrets with the sandbox," but some tasks genuinely need a credential (e.g. calling an approved API). Here's the safer pattern.

```text
Bad

↓

Secret baked into the sandbox image or code
```

```text
Better

↓

Sandbox starts with NO credentials

↓

Task requires an API call

↓

Short-lived, scoped credential injected
 at the moment it's needed (see Least-Privilege, JIT access)

↓

Credential expires when the sandbox is destroyed
```

---

# Warm Pools vs Cold Start

A performance consideration the base chapter's workflow doesn't address.

```text
Cold Start

↓

Create Sandbox -> Boot -> Execute -> Destroy

↓

Slower, but zero idle resource cost
```

```text
Warm Pool

↓

Pre-boot a pool of empty sandboxes ahead of time

↓

Assign one instantly when a request arrives

↓

Faster response, but idle sandboxes cost resources
```

High-traffic Agent platforms often use warm pools for latency- sensitive tasks and cold starts for infrequent ones.

---

# Updated Best Practices (Additions)

Choose isolation strength based on risk — containers for routine tasks, microVMs or full VMs for higher-risk or multi-tenant workloads.

Set explicit CPU, memory, disk, and timeout limits; don't rely on "isolation" alone to prevent resource exhaustion.

Default network access to zero, then allowlist only the specific destinations a task needs.

Keep sandbox writes on ephemeral storage so nothing survives sandbox destruction by accident.

Pair container/VM isolation with syscall filtering for defense in depth.

Still validate sandbox output with Guardrails — isolation doesn't guarantee correctness.

Inject credentials at runtime, scoped and short-lived, instead of baking them into the sandbox image.

---

# Updated Common Beginner Mistakes (Additions)

### Mistake 5

Assuming container isolation alone is unbreakable and skipping syscall filtering or resource limits.

---

### Mistake 6

Isolating the sandbox from the host, but not isolating different users' sandboxes from each other.

---

### Mistake 7

Trusting whatever the sandbox returns without running it through Output Guardrails.

---

### Mistake 8

Baking long-lived credentials into a sandbox image instead of injecting short-lived ones at runtime.

---

# Updated Key Takeaways (Additions)

- Isolation strength is a spectrum — processes, containers, microVMs, and full VMs trade off security against cost.
- Sandbox escapes are possible, so Sandboxing should never be the only safeguard.
- Multi-tenant platforms must isolate sandboxes from each other, not only from the host.
- Concrete limits (CPU, memory, timeout, network egress) are what actually enforce "restricted resources."
- Sandbox output still needs Guardrail validation — containment isn't the same as correctness.

---

