# STINGAR v2.4.4 Release Notes

**Release Date:** TBD

## Overview

STINGAR v2.4.4 is a honeypot patch release that fixes source IP addresses being
logged with an IPv4-mapped IPv6 prefix (`::ffff:`) after the dual-stack IPv6
listener change shipped in v2.4.3. Only the `cowrie` and `conpot` images change.

## Affected Versions

- Introduced in `v2.4.3` by the dual-stack IPv6 listener change (cowrie SSH/Telnet
  and conpot protocols bind `tcp6` on `::`).
- Any customer running `4warned/cowrie:v2.4.3` (or `4warned/conpot:v2.4.3`) that
  receives IPv4 traffic is affected. This is not customer-specific.

## Upgrade Steps

- Update honeypot image tags to `v2.4.4` for `cowrie` and `conpot`
- Pull and recreate the affected honeypot containers
- No configuration changes are required; dual-stack IPv4 + IPv6 listening is retained

## Breaking Changes

- None.

## Bug Fixes

- **IPv4-mapped source IPs on dual-stack listeners** – When a listener binds
  `tcp6` on `::` (dual-stack), the kernel delivers inbound IPv4 connections as
  IPv4-mapped IPv6 addresses (`::ffff:a.b.c.d`). The honeypot Fluentd output
  plugins now unwrap these to a clean dotted-quad (`a.b.c.d`) before emit.
  - Previously, IPv4 Telnet and SSH attackers were logged as
    `::ffff:<ipv4>` (e.g. `::ffff:221.160.73.81`) and classified as IPv6.
  - This broke deduplication, safelist/blocklist matching, CIF sharing, and
    geo/ASN enrichment, because the mapped form is a distinct string from the
    plain IPv4 value.
  - Real IPv6 addresses and plain IPv4 addresses are unchanged.
  - Fixed in both `cowrie` and `conpot` output plugins.

## Defense in Depth (server side)

- The feed engine (mellis) now unwraps IPv4-mapped IPv6 to dotted-quad during
  honeypot source-IP classification, so any emitter that still sends `::ffff:`
  values is normalized centrally before dedup and CIF sharing. This ships on the
  v2.5 line.

## Historical Data

- Events already indexed by a v2.4.3 honeypot retain the `::ffff:` prefix in
  Elasticsearch. New events after upgrading to v2.4.4 are clean. A reindex is
  only needed if historical IOC values must be normalized for correlation.

## Images Changed

- `4warned/cowrie:v2.4.4` (linux/amd64, linux/arm64)
- `4warned/conpot:v2.4.4` (linux/amd64, linux/arm64)
