+++
date = '2026-03-08T16:42:18+05:30'
title = 'Barbwire: an eBPF based behavioral correlator'
description = 'Technical writeup on how Barbwire was developed, the issues faced and lessons learned'
categories = ['eBPF', 'Experiment', 'Go']
tags = ['Learning', 'Linux', 'eBPF']
keywords = ['eBPF', 'Linux']
summary = 'Technical writeup on how Barbwire was developed, the issues faced and lessons learned'
+++
## Introduction
After the eBPF learning phase I wrote about in the [last post](https://itsmonish.pages.dev/blog/learning-ebpf), I wanted to build something that looked like a real security tool. Not because I thought I'd ship it, but because the programs I tried before where stupid. At some point you need a problem with actual constraints.

Barbwire is that project. It's a lightweight behavioral correlator for Linux. It watches for processes that open sensitive files and then make network connections within a short time window. This is classic exfil behavior and rough fingerprint for credential harvesting and so on. It's not a production EDR. It's an experiment. It's on my [Github](https://github.com/ItsMonish/barbwire) if anyone's interested.

## The core design decision
The first instinct when building something with eBPF is to put logic in the BPF side. It's running in the kernel, it's fast, why not do the correlation there?

I thought about it and decided against it early. The BPF verifier gets unhappy with complex logic. It's harder to debug, less portable across kernel versions, and for this use case, unnecessary. Userspace is fast enough. The design I landed on: BPF is a dumb data collector, userspace does the thinking.

The BPF side attaches tracepoints to three syscalls. `sys_enter_openat` for file opens, `sys_enter_connect` for network connections, and `sys_enter_execve` for process execution to track lineage. All three write into a single shared ring buffer with an event type field. Userspace reads and routes.

I used a ring buffer over a perf event array deliberately. Perf event arrays are per-CPU, so you have to poll multiple buffers and deal with out-of-order events. A ring buffer is a single shared buffer, better throughput, simpler consumer. So obviously I wanted the path of least resistance.

## How correlation and scoring works
On the userspace side, three in-memory maps hold state. Recent file opens per PID, process lineage from exec events, and a deduplication map to avoid alerting on the same PID repeatedly. A cleanup goroutine evicts stale entries every 30 seconds.

The correlation window is configurable. I have it set to 5 seconds. When a connect event comes in, Barbwire looks back at that PID's recent file opens and checks if any fall within the window. If they do, it scores the pair.

Scoring is based on file and connect pairs, not individual signals. A process connecting to the network is normal. Opening `/etc/passwd` on its own is normal. Both within 5 seconds is suspicious. The score comes from which file category was accessed:

```yaml
suspicious_files:
  - category: "credential access"
    base_score: 3
    patterns:
      - "/etc/passwd"
      - "/etc/shadow"
      - "/.aws/credentials"
  - category: "ssh key exfiltration"
    base_score: 3
    patterns:
      - ".ssh/id_rsa"
      - ".ssh/id_ed25519"
```

Process lineage modifies the score up or down. A shell spawning a process that reads credentials and connects out is more suspicious than the same behavior happening under systemd:

```yaml
suspicious_parents:
  - comm: "bash"
    modifier: 2
  - comm: "sshd"
    modifier: 3

legit_parents:
  - comm: "systemd"
    modifier: -2
  - comm: "dockerd"
    modifier: -2
```

Alert threshold and severity thresholds are decoupled. The threshold decides if the alert is an actual alert, severity is a classification of the alert. When a connect event comes in, Barbwire picks the highest scoring file match for that PID and emits one alert. One alert per connect event per process, no duplicates (hopefully).

A fired alert looks like this:

```
┌─ barbwire alert — PID 41890  ─────────────
│  command  : python
│  file     : /etc/passwd
│  connect  : 2404:6800:4007:821::200e:80
│  severity : HIGH
│  reasons  : credential access, suspicious parent: fish
│  parent   : fish (pid 13387)
│  gparent  : tmux: server (pid 9483)
└─────────────────────────────────────────────
```

## My Screwups while building it
A few things broke in ways that were actually instructive. The first one was actually two bugs hiding behind each other, which made it genuinely annoying to debug.

There was a constant mismatch between the C and Go sides. `EVENT_EXEC`, the event type for `sys_enter_execve` tracepoint, was 2 and `EVENT_CONNECT`, the event type for `sys_enter_connect`, was 3 in the BPF code, but I had them swapped in Go. At the same time, I had attached the exec tracepoint and called Close() on it immediately without a defer, so it detached right after attaching (Yeah this is embrassing).

The symptom was simple, in terms of events I had none. Thing was execve events were being routed to the connect handler. Since those events carry no IP address family, they got dropped as invalid. Meanwhile connect events were being routed to the exec handler, which was already closed. Everything was failing quietly in a different place than where I was looking.

I spent a while staring at the correlation logic before going back to basics and checking whether the tracepoints were even active. Once I caught the missing defer, connect events started showing up, but in the wrong handler. That's when I found the swapped constants. Two separate bugs, one symptom, neither pointing at the other. Really annoying

The most useful bug involved stale memory in the event struct. `bpf_ringbuf_reserve` gives you a pointer to uninitialized memory. If you skip the `__builtin_memset` before writing your fields, whatever was in memory from the previous event bleeds into the fields you didn't set. I had filenames with garbage characters appended, leftovers from longer filenames earlier in the buffer. The fix is one line. But you have to know it's needed.

## Limitations and takeaways
The biggest problem with Barbwire is the one that can't be tuned away.

For example, `curl` reads `/etc/passwd` on every invocation. Here is it making the call: 
![curl opening passwd](/images/barbwire/1.png)

Not because it's malicious, but because that's how it works. I believe, basically any program using libc networking does this. Barbwire flags it every time. No matter how hard I tuned the knobs, I couldn't get rid of such alerts. Then it hit me, single-process correlation without broader context has a ceiling.

Evidently, production security tools handle this differently. Binary hash whitelisting instead of process name matching, which is spoofable. Process ancestry graphs that track behavioral chains across multiple processes. Behavioral baselines built up over time. A shell that reads `/etc/passwd` and then forks a child that connects out would be caught by a graph-based correlator. Barbwire misses it because the file open and the connect are in different PIDs.

Building this made that limitation concrete in a way I don't think I'd have gotten from just reading about EDR design. Honestly, I never even thought of these before. You also understand why process ancestry graphs exist when your tool fails exactly where they'd succeed.

The other thing I took away: the BPF to userspace boundary is unforgiving. Memory layout, pointer semantics, which helper functions apply where, what the verifier accepts. The documentation covers the what, but the why only shows up when things break. 

All things considered, I had fun with all this. That's the whole point if you think about it.
