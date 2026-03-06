---
title: "Debugging Stuck tmt Processes with Claude Code"
date: 2026-03-06
draft: false
tags: ["testing-farm", "tmt", "debugging", "python", "regex", "claude-code"]
summary: "How I used Claude Code to debug stuck Testing Farm jobs in minutes — tracing the problem from process list through Python stack traces to a catastrophic regex backtracking bug in tmt."
---

We received a support request reporting ~30
[Testing Farm](https://docs.testing-farm.io) jobs stuck in "running" state with
no evidence of tests executing. The jobs came from
[Hummingbird](https://gitlab.com/redhat/hummingbird), a project that builds and
tests container images. Guest logs showed VMs were provisioned quickly, but
nothing was happening after that. Here's how I used
[Claude Code](https://docs.anthropic.com/en/docs/claude-code/overview) to debug
the issue — going from "something is stuck" to a confirmed root cause in about
15 minutes.

## The symptoms

The Hummingbird team reported multiple Testing Farm jobs across several merge
requests stuck in "running" state for hours. The artifact logs showed the test
runner had successfully provisioned guest machines and started executing, but
then stopped producing any output. This wasn't a new issue — it had happened
intermittently before — but this time it affected ~30 jobs simultaneously.

Looking at the worker, I could see the container was running, SSH connections to
test hosts were established, but nothing was actually executing on the hosts.

## Connecting to the stuck process

I connected to the worker container running one of the stuck jobs. From there,
all the debugging was done using Claude Code with a series of diagnostic
commands.

## The debugging session

### Step 1: Check process state

```console
# ps aux --sort=-%cpu | head -10
```

This immediately revealed two [tmt](https://github.com/teemtee/tmt) processes
consuming ~75% CPU each, both running for over 130 minutes of CPU time. Both
were executing tmt test runs against different architectures (x86_64 and
aarch64) for the same Hummingbird test plan (`/ci/run_tests_container`).

### Step 2: Check kernel stack

```console
# cat /proc/1061/stack
[<0>] futex_wait_queue+0x63/0x90
[<0>] futex_wait+0x189/0x270
[<0>] do_futex+0xc6/0x190
[<0>] __x64_sys_futex+0x129/0x1e0
[<0>] do_syscall_64+0x60/0x90
[<0>] entry_SYSCALL_64_after_hwframe+0x6e/0xd8
```

Both processes were stuck on a futex wait — a lock in userspace. This pointed
to something inside the Python runtime holding the GIL.

### Step 3: Python-level stack traces

The container had `py-spy` installed, which can dump Python stack traces
without attaching a debugger:

```console
# py-spy dump --pid 1061
```

```text
Thread 1061 (idle): "MainThread"
    wait (threading.py:359)
    as_completed (concurrent/futures/_base.py:243)
    _invoke_in_pool (tmt/queue.py:166)
    go (tmt/steps/execute/__init__.py:1234)
    ...

Thread 1742 (active+gil): "ThreadPoolExecutor-10_0"
    findall (re/__init__.py:278)
    _extract_failures (tmt/frameworks/shell.py:20)
    extract_results (tmt/frameworks/shell.py:153)
    ...
```

The second process (PID 1310) showed an identical stack. The pattern was clear:

- **MainThread** was idle, waiting for a `concurrent.futures` task to complete
- **Worker thread** held the GIL and was stuck inside `re.findall()` at
  `tmt/frameworks/shell.py:20`

### Step 4: Identify the problematic code

The code at `tmt/frameworks/shell.py:20`:

```python
def _extract_failures(invocation, log_path):
    try:
        log = invocation.phase.step.plan.execute.read(log_path)
    except tmt.utils.FileError:
        return []

    return re.findall(r'.*\b(?:error|fail)\b.*', log, re.IGNORECASE | re.MULTILINE)
```

This regex scans an entire test output file for lines containing "error" or
"fail". With `re.MULTILINE`, `.*` matches within each line — but on very long
lines, the regex engine does enormous amounts of work.

### Step 5: Check the input data

```console
# find /var/ARTIFACTS/work-run_tests_container* -name "output.txt" \
    -exec ls -lh {} \;
```

Most output files were small (< 2 KB), but two were **1.6 MB** each — the
`Verify-reproducibility-1/output.txt` files from the Hummingbird container
build reproducibility tests.

```console
# wc -l output.txt
900

# wc -L output.txt
1058495
```

900 lines, but the longest line was **1,058,495 characters**. A second line was
467,425 characters. These were base64-encoded
[in-toto](https://in-toto.io/) attestation payloads and JSON build
configurations — massive single-line outputs from the Hummingbird container
build pipeline dumped via `bash -x` tracing.

### Step 6: Confirm the issue

To confirm the regex was the actual problem, I ran a controlled test inside the
container:

```python
import time, re, signal

class Timeout(Exception):
    pass

def handler(signum, frame):
    raise Timeout()

signal.signal(signal.SIGALRM, handler)

with open('output.txt') as f:
    lines = f.readlines()

long_line = lines[658]  # the 1M-char line

for size in [10000, 50000, 100000, 200000]:
    chunk = long_line[:size]
    signal.alarm(10)
    try:
        start = time.time()
        re.findall(r'.*\b(?:error|fail)\b.*', chunk, re.IGNORECASE | re.MULTILINE)
        print(f'{size:>7} chars: {time.time() - start:.3f}s')
    except Timeout:
        print(f'{size:>7} chars: TIMED OUT after 10s')
```

Results:

```text
  10000 chars: 5.389s, 0 matches
  50000 chars: TIMED OUT after 10s
 100000 chars: TIMED OUT after 10s
 200000 chars: TIMED OUT after 10s
```

Even **10,000 characters took 5.4 seconds**. At 50,000+ characters, the regex
didn't complete within 10 seconds. The full 1M-character line would take hours
— which is exactly what we observed (130+ minutes of CPU time per process).

For comparison, a safe alternative that skips lines over 100k characters
completed the entire file in **1.1 seconds** and found 7 legitimate matches.

## Root cause

The regex `.*\b(?:error|fail)\b.*` with `re.MULTILINE` suffers from
**catastrophic backtracking** on long lines. Here's why:

1. The leading `.*` greedily consumes the entire line
2. It then backtracks character by character looking for a word boundary
   followed by "error" or "fail"
3. When neither word is found (as in a base64-encoded line), the engine must
   backtrack through every position in the line
4. The trailing `.*` adds another layer of backtracking

On a 1M-character line with no matches, this results in the regex engine
exploring an enormous number of positions before giving up.

The worker thread holds Python's GIL while executing `re.findall()`, which
blocks the main thread waiting in `concurrent.futures.as_completed()`. The
process appears stuck — high CPU usage but no progress.

## The fix

This is a [tmt](https://github.com/teemtee/tmt) bug. A proper fix could:

- **Limit line length** before applying the regex (skip or truncate lines over
  a threshold)
- **Use a non-backtracking pattern** like iterating line by line with a simple
  substring check (`"error" in line.lower() or "fail" in line.lower()`)
- **Set a regex timeout** (Python 3.13+ doesn't have built-in regex timeouts,
  but the operation could be wrapped with a signal-based timeout)

The user-side workaround is to avoid dumping massive single-line outputs (like
base64 attestation blobs) into test logs — for example, by disabling `bash -x`
tracing around commands that produce such output.

## Claude Code as a debugging tool

This was an unusual debugging session in that I didn't manually type most of
the diagnostic commands — I described the problem to Claude Code and it
autonomously ran the investigation. The workflow was:

1. I provided the connection command to reach the stuck container
2. Claude Code ran `ps aux`, checked `/proc/*/stack`, used `py-spy` for Python
   stack traces, examined the source code, checked file sizes, found the
   long lines, and ran a controlled reproduction test
3. The entire investigation from "something is stuck" to "confirmed root cause
   with reproduction" took about 15 minutes

What would normally require jumping between terminals, reading source code,
crafting test scripts, and cross-referencing multiple pieces of evidence was
handled as a single conversational debugging session.
