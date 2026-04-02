# SOC Dashboard — SPL Queries

All queries used in the Splunk SOC dashboard. Set all panels to **Real-time 30 minute window** unless noted.

---

## Panel 1 — Active Alerts

Shows real Suricata alerts, filtered to remove known noise.

```
sourcetype=suricata event_type=alert NOT dest_port=1900 
NOT alert.signature="SURICATA Ethertype unknown"
| table timestamp src_ip dest_ip dest_port alert.signature alert.severity in_iface
| sort -timestamp
```

---

## Panel 2 — Top Threat Signatures

Most frequently triggered detection rules — shows what Suricata is seeing most.

```
sourcetype=suricata event_type=alert NOT dest_port=1900
| stats count by alert.signature alert.category
| sort -count
```

---

## Panel 3 — Top Source IPs

External IPs generating the most alerts — useful for identifying scanners or attackers.

```
sourcetype=suricata event_type=alert NOT dest_port=1900
| stats count by src_ip
| sort -count
```

---

## Panel 4 — SSH Failed Logins

Detects brute force attempts from auth.log. Any IP with repeated failures is a potential attacker.

```
sourcetype=linux_secure ("Failed password" OR "Invalid user")
| rex field=_raw "from (?P<src_ip>\d+\.\d+\.\d+\.\d+)"
| stats count by src_ip
| sort -count
```

---

## Panel 5 — Alert Severity Over Time

Tracks alert volume by severity over time. Spikes indicate active attack activity.
Set to **Real-time 60 minute window**. Visualization: **Line Chart**.

```
sourcetype=suricata event_type=alert NOT dest_port=1900
| timechart count by alert.severity
```

---

## Panel 6 — DNS Feed (Live)

Real-time feed of DNS queries ordered by most recent. Useful for spotting C2 beaconing or policy violations.

```
sourcetype=suricata event_type=dns
| spath input=_raw path="dns.queries{0}.rrname" output=queried_domain
| table timestamp queried_domain src_ip dest_ip
| sort -timestamp
```

**Suspicious DNS variant** (rare domains only — more likely to be malicious):

```
sourcetype=suricata event_type=dns
| spath input=_raw path="dns.queries{0}.rrname" output=queried_domain
| stats count by queried_domain
| where count < 5
| sort -count
```

---

## Panel 7 — Inbound vs Outbound Alerts

Shows whether threats are coming in or going out. Visualization: **Pie Chart**.

```
sourcetype=suricata event_type=alert NOT dest_port=1900
| stats count by direction
```

---

## Panel 8 — External Threat Sources (GeoIP)

Enriches source IPs with geographic data to identify where attacks are originating.

```
sourcetype=suricata event_type=alert NOT dest_port=1900
| iplocation src_ip
| stats count by src_ip Country City
| sort -count
```

---

## Bonus — Validate Full Pipeline

Run this after any simulation to confirm end-to-end detection is working:

```
sourcetype=suricata event_type=alert
| table timestamp src_ip dest_ip alert.signature alert.severity
| sort -timestamp
```
