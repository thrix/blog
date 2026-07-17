---
title: "The Dumb Git Protocol That Flooded Our Git Server"
date: 2026-07-17
draft: false
tags: ["git", "testing-farm", "tmt", "http", "debugging"]
summary: "A CI clone spike pulled terabytes off an internal git server and caused an outage. The cause was one URL pointing at git's dumb HTTP protocol, which makes every fresh clone re-fetch a repo object by object."
---

Our internal git server started timing out one afternoon. `git clone` and
`git fetch` began failing with connection timeouts, and twelve package builds
failed over the day because of it. The host was saturated: a few hundred running
tasks, CPU pinned in user space, and the web server process grown to tens of
gigabytes of RAM with every worker busy.

The repos are served by [cgit](https://git.zx2c4.com/cgit/), a common web
frontend for git. Nothing about the server had changed. What changed was the
load [Testing Farm](https://docs.testing-farm.io) put on it: the largest clone
spike we'd produced, around 3x a normal day, while scaling workers to a new
maximum.

## Where the traffic came from

The access log named the source. Thousands of requests, all with a `git/2.55.0`
user agent, all for objects under one repository:

```text
GET /cgit/tests/openssl/objects/pack/pack-....idx        200
GET /cgit/tests/openssl/objects/3f/2a...                 404
GET /cgit/tests/openssl/objects/9c/71...                 200
GET /cgit/tests/openssl/objects/a1/0b...                 404
```

The requests came from Testing Farm workers, which had scaled up hard that week.
Every worker starting a run cloned this repo, and each clone made thousands of
per-object requests, a large share of them 404.

The repo (an OpenSSL test suite) is large with deep history, but repo size
doesn't produce terabytes. A normal clone transfers one packfile and stops. Thousands
of tiny requests per clone, most fetching individual objects by SHA, many
404ing, is the signature of git's **dumb HTTP protocol**.

## Git's two HTTP protocols

Git serves over HTTP two ways with different cost profiles.

The **smart** protocol runs `git-upload-pack` on the server. The client requests
`info/refs?service=git-upload-pack`, sends what it has and wants, and the server
computes one packfile containing exactly those objects and streams it back. One
negotiation, one packfile, minimal bytes.

The **dumb** protocol applies when the server treats `.git/` as static files with
no git process behind it. Without negotiation, the client walks the object graph
itself: read a ref, GET that commit object, parse it, GET the objects it
references, repeat. One HTTP GET per object.

Packing makes it worse. A dumb client first assumes an object is loose and
requests it by path. If the object has been packed, the server returns 404, and
the client falls back to downloading whole pack files and their indexes to
extract the object locally. One dumb clone of a packed repo becomes thousands of
GETs, a wave of 404s for every packed object, and full-pack downloads. Nothing is
shared between clones, so every worker repeats the whole sequence from empty.

That accounts for the volume. Not one large download times N workers, but N
workers each issuing thousands of small requests and transferring more than the
repo contains.

## Confirming the protocol

The dumb path lives under `/cgit/` and the smart path under `/git/`. The
`Content-Type` of the `info/refs` response tells them apart.

Smart endpoint:

```console
$ curl -sI "https://distgit.example.com/git/tests/openssl/info/refs?service=git-upload-pack" \
    | grep -i content-type
content-type: application/x-git-upload-pack-advertisement
```

Dumb endpoint:

```console
$ curl -sI "https://distgit.example.com/cgit/tests/openssl/info/refs" \
    | grep -i content-type
content-type: text/plain; charset=utf-8
```

`application/x-git-upload-pack-advertisement` means a `git-upload-pack` process
will negotiate a packfile. `text/plain` means git receives a static file and
falls back to walking objects by hand. Both endpoints advertised the same
several-thousand refs for the same repo.

The timing gap is large. Cloning the repo through `/cgit/` took about seven
minutes, grinding through object requests and 404s. Cloning through `/git/`
finished in seconds with one packfile. The plan ran this clone twice per test
run, from every worker, during the biggest scale-up we'd done.

I timed every clone in a two-hour window to rule out a few unlucky runs. The
dumb-protocol clones of this one repo averaged about seven minutes each and
accounted for roughly 89% of all git-clone time measured in that window. One
misrouted URL was most of the load.

## The fix

The clone URL pointed at `/cgit/`. Changing it to `/git/` was the fix:

```text
before:  https://distgit.example.com/cgit/tests/openssl/
after:   https://distgit.example.com/git/tests/openssl
```

The URL lived in [tmt](https://github.com/teemtee/tmt) plan metadata, a
`plan import` that fetched the OpenSSL test plan by cloning the repo. Whoever
wrote the plan pasted the `cgit` web-UI URL, the one the browser shows when you
view the repo. It renders in a browser and clones once without trouble. At a few
hundred concurrent workers, the `/cgit/` fallback turned each clone into an
object-by-object flood.

After the change deployed, the object storm in the access log stopped and the
server load returned to baseline.

## Takeaways

A git server buried in per-object requests and `objects/xx/...` 404s is serving
the dumb protocol, and something is cloning the slow way. Adding capacity doesn't
address it; moving the client to the smart protocol does, and that can be a
one-line URL change.

Check where clone URLs come from. A `cgit` or web-UI link targets a human reading
code in a browser. It clones, which is the trap: it works in a manual test, passes
review, and serves the dumb protocol to every automated client after that. The
browse URL and the clone URL differ by a path segment and diverge by orders of
magnitude under load.

Nothing here was broken in the usual sense. The repo, the server, and the workers
all worked. A `/cgit/` where a `/git/` belonged stayed invisible until the worker
count scaled far enough for the protocol difference to reach terabytes.
