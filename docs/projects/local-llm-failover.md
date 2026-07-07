# Local LLM Failover

Giving a self-hosted AI agent a **local LLM backstop** — automatic failover when the
primary cloud model is unavailable, plus on-demand use to save API costs.

!!! success "Status"
    Control plane, headless autostart, and agent integration **done & verified**.
    One networking item outstanding (a host-local forwarder — see below).

## 1 · The Goal

Give the primary AI agent (a self-hosted **Hermes Agent** instance) a **local LLM** it
can fall back to:

1. **Automatically** when the primary cloud model (Claude) is rate-limited, overloaded,
   or unreachable, and
2. **On demand**, for tasks that don't need a frontier model — saving paid API usage.

The local model is **Qwen3 14B** served by **Ollama** inside **WSL2** on a Windows 11
desktop. Ollama exposes an OpenAI-compatible API, so the agent treats it as just another
provider in its fallback chain.

## 2 · Benefits

| Benefit | Why it matters |
|---|---|
| **Cost savings** | Offload low-stakes tasks to the local GPU instead of paying per-token. |
| **Resilience** | Cloud outage or rate-limit → the agent keeps working on local Qwen. |
| **Privacy** | Local-only tasks never leave the network. |
| **One identity** | Failover is a *provider swap* inside the same agent — not a second agent with divergent memory. |
| **Reuses hardware** | The desktop already runs WSL2 + Ollama; no new box. |

## 3 · Architecture

### Reach-in, not a resident agent

Every agent install is a complete standalone agent (own model, memory, sessions). Rather
than spawn a second agent on the desktop that would drift, the primary agent connects
*into* the desktop over an admin SSH channel and does the work **as itself** — one brain,
one memory.

```
   ┌──────────────┐   ① SSH (admin control)   ┌────────────────────────────┐
   │  NAS          │ ─────────────────────────▶│   Windows 11 desktop        │
   │  (the agent)  │                            │   ┌──────────────────────┐  │
   │               │   ② HTTP (LLM API)         │   │ WSL2 → Ollama (local API)  │  │
   │  ollama       │ ─────────────────────────▶ │   │   qwen3:14b           │  │
   │  provider     │                            │   └──────────────────────┘  │
   └──────────────┘                            └────────────────────────────┘

   Two channels: ① SSH control plane, ② the HTTP LLM data path.
```

### Failover request flow

```
   Agent needs a model call
        │
        ▼  Primary: Claude ──success──▶ response
        │
        │ rate-limit / 5xx / connection error
        ▼  Fallback: qwen3:14b (local Ollama) ──success──▶ response
```

The agent framework's fallback-provider chain tries entries in order when the primary fails.

### Headless cold-boot

A scheduled task boots WSL on startup (no login required); systemd then starts Ollama
automatically, so the local model is available even on a freshly-rebooted, logged-out machine.

## 4 · Key Lessons (the transferable bits)

These are the gotchas worth knowing if you build something similar:

- **Windows OpenSSH for admin accounts** reads authorized keys from a *system* file, not
  the user's home directory — and it must have locked ACLs or it's silently ignored. This
  is the single biggest time-sink.
- **Headless scheduled tasks on a Microsoft-account PC:** you often can't supply the
  account password (PIN/Hello login). The **S4U logon type** registers a "run whether
  logged on or not" task with **no stored password** — it mints a local-only token, which
  is all that's needed to launch WSL.
- **WSL2 networking:** the newer "mirrored" mode was unreliable on this hardware (its port
  relay flapped). Reverting to **NAT + a port forward** made the host-side path rock-solid.
- **Ollama speaks the OpenAI API**, so wiring it into an agent framework is just a custom
  provider entry with a `base_url` — no special integration needed.

## 5 · Outstanding: a host-local forwarder

After moving to NAT networking, the **host itself** reaches the local model perfectly, but
a **remote machine on the LAN** gets a TCP connection that then returns no response. The
cause is a known limitation of the built-in Windows port-forwarder: it serves connections
that *originate on the host* reliably, but drops the response path for **externally-originated**
connections.

The planned fix is a small **user-space forwarder** (e.g. `socat`) running on the desktop:
it accepts the LAN connection, terminates it **host-locally** (the path we've proven works),
then opens a fresh host-local connection to the model. Both halves ride the known-good path,
so responses return correctly to remote clients.

```
   remote client ──LAN──▶  forwarder on desktop (host-local termination)
                                │
                                ▼  host-local  ──▶  WSL Ollama
```

---

!!! info "Public version"
    This write-up omits internal addresses, hostnames, credential paths, and exact
    topology. The full operational runbook lives in the access-controlled internal build.
