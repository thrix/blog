---
title: "A DNS Record That Existed Everywhere Except CI"
date: 2026-06-30
draft: false
tags: ["dns", "testing-farm", "kubernetes", "coredns", "aws", "debugging"]
summary: "A flaky CI failure where a freshly-created DNS record was unresolvable from inside the cluster, but resolved fine from my laptop a few minutes later. The culprit was negative DNS caching, driven by the zone's SOA, poisoning lookups for our ephemeral per-job records."
---

We had a [Testing Farm](https://docs.testing-farm.io) CI job that kept failing,
and the failure made no sense to me for a while. The job spins up a throwaway
environment for each pipeline, including an
[Artemis](https://gitlab.com/testing-farm/artemis) instance you reach at a
per-job hostname like:

```text
artemis.gitlab-ci-12345678.testing-farm.io
```

The job died because the GitLab runner couldn't resolve that hostname. Fine,
DNS problem. Except when I went to check it myself, it resolved instantly from
my laptop:

```console
❯ host artemis.gitlab-ci-12345678.testing-farm.io
artemis.gitlab-ci-12345678.testing-farm.io has address 203.0.113.10
artemis.gitlab-ci-12345678.testing-farm.io has address 203.0.113.11
```

So the record was right there. The runner just couldn't see it. That kind of "it
works for me but not in CI" is always either a timing thing or a caching thing,
and this one turned out to be a particularly sneaky bit of DNS that I think a lot
of people have hit without realizing what it was. I poked at it with
[Claude Code](https://docs.anthropic.com/en/docs/claude-code/overview) driving
the `kubectl` and `dig`, which kept the loop quick.

## How the records get there

A bit of background first. Those per-job records aren't static. They're created
on the fly by
[external-dns](https://github.com/kubernetes-sigs/external-dns): when the
ephemeral environment comes up, external-dns writes an `A` record into our public
Route 53 zone (`testing-farm.io`) pointing at the freshly-created load balancer,
and it all gets torn down again when the job finishes.

The runners are pods in an EKS cluster, so a lookup from a runner travels a
little chain:

```text
build pod → CoreDNS (cluster) → /etc/resolv.conf upstream → AWS VPC resolver → Route 53
```

The important detail: the hostname simply does not exist until external-dns
publishes it, and that only happens once the load balancer is up.

## Reproducing it where it actually breaks

My laptop was useless for this. It uses a totally different resolver, so of
course it was happy. I needed to ask DNS the same way a runner would, so I threw
a pod into the cluster and queried the cluster DNS directly:

```console
$ kubectl run dnstest --rm -it --image=busybox:1.36 -- \
    nslookup artemis.gitlab-ci-12345678.testing-farm.io
Server:    10.96.0.10
Address:   10.96.0.10:53

Non-authoritative answer:

Non-authoritative answer:
```

Nothing. No address. But a stable name in the very same zone resolved fine from
that same pod:

```console
$ nslookup api.staging.testing-farm.io
...
api.staging.testing-farm.io  canonical name = ec2-198-51-100-20.us-east-2...
Address: 198.51.100.20
```

So DNS in the cluster wasn't broken in general. It was broken for this one name.
Same zone, same resolver, one works and one doesn't. That's the moment it stopped
being "DNS is flaky" and started being "something is specifically wrong with this
name."

## It's NODATA, not NXDOMAIN

`nslookup` glosses over the bit that actually matters here, so I switched to
`dig` and asked two things: what does the cluster resolver say, and what does the
authoritative server say?

The cluster resolver (CoreDNS):

```console
$ dig +noall +comments +answer @10.96.0.10 artemis.gitlab-ci-12345678.testing-farm.io
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 22678
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1
```

Look closely: `status: NOERROR` but `ANSWER: 0`, plus one `AUTHORITY` record.
That's not NXDOMAIN, which would mean "this name doesn't exist." It's NODATA, a
*negative* answer that basically says "nothing of that type here," and it comes
back carrying the zone's `SOA` record in the authority section. Hold that
thought.

Now the authoritative Route 53 nameserver:

```console
$ dig +noall +comments +answer @ns-xxx.awsdns-xx.org. artemis.gitlab-ci-12345678.testing-farm.io
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 62164
;; flags: qr aa rd; QUERY: 1, ANSWER: 2, AUTHORITY: 4, ADDITIONAL: 1
;; ANSWER SECTION:
artemis.gitlab-ci-12345678.testing-farm.io. 60 IN A 203.0.113.11
artemis.gitlab-ci-12345678.testing-farm.io. 60 IN A 203.0.113.10
```

The authoritative server has the record. The cluster's resolver is handing back
a stale "no it doesn't" instead. So the record exists, and the resolvers in the
middle just don't believe it yet.

## Quick detour: what negative caching actually is

There are two flavors of DNS caching, and most people only ever think about the
first one.

Positive caching is the obvious one: "name X is 1.2.3.4", remembered for that
record's TTL. Negative caching is the other half: a resolver also needs to
remember that a name *doesn't* exist, so it doesn't hammer the authoritative
servers every time something asks for a typo. But there's no record for a name
that doesn't exist, so where does the TTL for "it's not there" come from?

[RFC 2308](https://datatracker.ietf.org/doc/html/rfc2308) answers that: the
negative answer carries the zone's `SOA`, and the resolver caches the "doesn't
exist" for

```text
negative TTL = min(SOA MINIMUM field, TTL of the SOA record)
```

Here's our SOA (nameserver and mailbox redacted):

```console
$ dig +noall +answer SOA testing-farm.io
testing-farm.io. 900 IN SOA ns-xxx.awsdns-xx.net. hostmaster.example. 1 7200 900 1209600 86400
                 ^TTL                                                 │ │    │   │       └ MINIMUM = 86400
                 (of the SOA record)                          serial refresh retry expire
```

That last number, `MINIMUM = 86400`, is the negative-cache knob. The confusing
part is the name: way back in BIND, `MINIMUM` meant the default TTL for every
record in the zone, but RFC 2308 quietly redefined it to mean the negative
caching TTL and never renamed the field. So it reads like one thing and means
another. And these were just the AWS Route 53 defaults. I'd never touched them.

Do the math and the effective negative TTL was `min(900, 86400) = 900` seconds.
Up to fifteen minutes of a resolver insisting a name doesn't exist.

## The actual root cause

Once you see the negative-caching angle, the whole thing falls out:

1. A job starts and something resolves
   `artemis.gitlab-ci-<jobid>.testing-farm.io` a beat too early, before
   external-dns has published it.
2. Route 53 answers, correctly, "not here yet", and hands back the SOA.
3. Everything in the resolution path is now allowed to cache that "not here" for
   the negative TTL. CoreDNS caps its own cache at 30s (`cache 30` in the
   Corefile), but it forwards upstream to the AWS VPC resolver, which honors the
   zone's negative TTL of up to fifteen minutes.
4. A few seconds later external-dns publishes the record. Too late. The cluster
   keeps serving the cached miss until it expires.
5. The job's wait-for-Artemis loop times out inside that window, and the job
   fails.

That also explains the maddening "works on my laptop" part. By the time I checked
by hand, minutes had passed, the negative entry had aged out, and my laptop's
resolver had probably never cached the miss to begin with. The bug hides itself
by healing, just slower than the job can wait.

I actually watched it heal. Same `dig` against the cluster resolver, twice, a
couple of minutes apart:

```text
Run 1:  status: NOERROR, ANSWER: 0   (negative, cached miss)
Run 2:  status: NOERROR, ANSWER: 2   (the two A records)
```

Nothing changed but the clock.

## The fix

The annoying thing is you can't fix this on the record. There is no record, the
whole problem is the *absence* of one, and negative caching is a property of the
zone's SOA by design. So you go to the SOA and shorten how long the internet is
allowed to remember a miss.

In Route 53 I edited the `SOA` for `testing-farm.io` and dropped the record's
TTL from `900` to `60`:

```text
before:  TTL 900   …  1 7200 900 1209600 86400
after:   TTL 60    …  1 7200 900 1209600 60
```

Because the negative TTL is `min(SOA TTL, MINIMUM)`, taking the TTL down to `60`
already caps it at a minute for any well-behaved resolver. I lowered the
`MINIMUM` field too, since a few resolvers look at that one directly. None of
this touches real records: everything that exists still resolves on its own TTL
exactly like before. The only thing that changed is how long a resolver clings to
"that name isn't there", and that went from fifteen minutes to one.

The flaky job went green on the next run, and the per-job hostname resolved from
inside the cluster right away.

## What I'd take away from this

If a lookup "doesn't resolve", check whether you're actually looking at a cached
*negative* answer rather than a missing record. `NOERROR` with `ANSWER: 0`
(NODATA) and `NXDOMAIN` both get cached against the zone's SOA, and neither of
them means "the record was never created."

Test from where it breaks, not from your laptop. Mine lied to me the entire time
because it sits on a different resolver. A throwaway pod plus a few `dig`s, one at
the cluster resolver and one at the authoritative servers, is what finally showed
which layer was wrong.

And if you create DNS records dynamically at all, external-dns, per-PR
environments, anything ephemeral, go lower your zone's SOA negative-cache TTL.
The Route 53 default of `900 / 86400` is fine for a stable zone and quietly
hostile to ephemeral names: one lookup that lands a few seconds early can poison
resolution for a quarter of an hour. The default just isn't built for names that
blink in and out of existence all day.
