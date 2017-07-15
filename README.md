# MeasureDNSrefresh

Some code to try to estimate how effective DNS Opportunistic Refresh
may be for a server.

The basic idea of DNS opportunistic refresh is that you may have lots
of different names and types in a single zone which all share the same
serial number (the SOA serial). If you do a lookup and the SOA serial
has not changed, then you can extend the TTL (time-to-live, which is a
timer for how long you can cache the data) for _all_ of the names &
types in that zone.

There is a draft explaining the idea in more detail:

https://datatracker.ietf.org/doc/draft-muks-dnsop-dns-opportunistic-refresh/

The main outstanding question is whether or not this is actually helps
much in the real world. That question is what this repository tries to
answer.

## Authority Investigation

In order to look for queries that may be saved by opportunistic
refresh from the authoritative side, we can examine query logs for
this pattern:

* We get a query (Query A) for a given QNAME/QTYPE (query name and
  type). We can answer this authoritatively from zone X. We expect the
  TTL for Query A to be at time T1.

* Before the TTL for Query A expires, we get another query (Query B)
  for a different QNAME/QTYPE which we can answer authoritatively from
  zone X. Now we can extend the TTL from Query A to be at T2.

* Later, we see a Query A again, after T1 but before T2. The
  difference T2-T1 is the time that we would have saved by using
  opportunistic refresh.

This works for both negative (NXDOMAIN or NODATA responses) as well as
positive answers. We have to be sure that the SOA has not changed
during this sequence. Finally, we should check queries across all of
the authoritative servers for zone X otherwise we may miss queries
that can be saved (this means that if we do not, the savings are a
sort of worst-case measurement).

## Resolver Investigation

We can also do some investigation on the resolver side. In either a
passive or active mode.

In passive mode, we just look at queries, in a similar manner to the
authoritative investigation above. The limitation of this is that we
cannot know if the SOA has been updated for a response, so we can only
check for TTL extension for negative answers (which include the SOA of
the zone). This is quite limited (and in fact the motivation for the
draft).

In active mode, when we see a response that does not include the SOA,
we can perform a separate SOA query explicitly and use that to
estimate potential benefits. This needs to happen in real time, so
requires either an instrumented resolver or a side program which
watches resolver traffic and does SOA queries alongside it.

Note that if we see an SOA at any time for a zone and it has the same
serial, we can apply the opportunistic refresh logic for all lookups
prior to that (thanks to Stephen Morris for the suggestion).

