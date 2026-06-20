# Monitoring Stack

InfluxDB v2 + Telegraf + Grafana, monitoring the firewall, the hypervisor host,
and the DNS sinkhole.

---

## Components

| Component | Notes |
|---|---|
| InfluxDB v2 | Time-series database, dedicated LXC container |
| Telegraf | systemd service on the metrics container and the hypervisor host; runs natively on the firewall via its built-in Telegraf plugin |
| Grafana | Dashboards, Flux query language |

Both InfluxDB and Grafana were deployed via community-maintained install
scripts using the **default** install mode. The advanced installer's VLAN tag
setting failed to propagate to the container config and broke DHCP for the
container — default mode avoided the issue.

> The metrics database org name is case-sensitive and must match exactly in
> every agent's output config, or writes silently fail. (A port typo in the
> database URL caused the same silent-failure symptom — caught via the agent
> log.)

---

## Firewall metrics

The firewall (OPNsense) has a native Telegraf plugin available through its
plugin system — GUI-configured, no SNMP or MIB files required. Significantly
simpler than the manual SNMP approach originally attempted, which was
abandoned in favor of the native plugin. Provides packet-filter stats and
system metrics. Dashboard queries distinguish data sources by host tag.

---

## DNS sinkhole integration (custom)

The DNS sinkhole (Pi-hole) has no native Telegraf plugin, and its newest major
version moved to session-based API authentication (POST a password, receive a
short-lived session token, use that token on subsequent requests) instead of
the previous version's static API key. That ruled out a simple HTTP-input
config with a static token.

**Solution:** a small Python script that performs the full
authenticate → fetch → format cycle on every invocation, wired in via the
monitoring agent's `exec` input type on a 60-second interval.

The script:
- Authenticates against the sinkhole's API to obtain a session token
- Fetches summary stats (total/blocked queries, percent blocked, unique
  domains, client counts, blocklist size)
- Fetches the current top-10 blocked domains
- Emits both as InfluxDB line protocol on stdout, which the agent ingests
  directly
- Logs out the session as a best-effort cleanup step

The script lives outside any user home directory and is owned by the
monitoring service account (see [troubleshooting.md](troubleshooting.md) for
why that matters), with the API password passed via environment variable
rather than hardcoded.

Full debugging path — a vendor-side auth bug, a PATH issue, and a permissions
issue — is documented in [troubleshooting.md](troubleshooting.md).

### Known limitation

The top-blocked-domains metric is a snapshot per collection interval, not an
accumulating history — useful for "what's being blocked right now," not for
trend analysis over time. Would require a script change or a windowed query
to support that.

---

## Dashboards

Built with Flux. Panels bind to the dashboard time range via template
variables. Panel set:

- Packet-filter stats and intrusion-detection alerts (firewall)
- Query volume and block rate (time series)
- Active / total DNS clients (stat)
- Blocklist size (stat)
- Forwarded vs. cached query ratio (time series)
- Unique domains queried (stat)
- Top blocked domains (table, sorted by count descending)

Point-in-time values use a `last()` aggregation rather than being plotted as
trends.

---

## Skills demonstrated

- Time-series monitoring pipeline design (collection → storage → visualization)
- Building a custom integration for a service with no native agent plugin
- Working with session-based REST API authentication
- Service-account permission and PATH debugging on Linux
- Dashboard design with a time-series query language
