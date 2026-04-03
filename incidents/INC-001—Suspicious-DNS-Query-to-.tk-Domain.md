# INC-001 — Suspicious DNS Query to .tk Domain

**Date:** 2026-04-03  
**Severity:** Medium  
**Status:** Investigated — Benign (False Positive)
**Analyst:** Willis Nomena Andre  

---

## Summary
Suricata detected an outbound DNS query to a `.tk` domain from internal 
host `172.20.13.236` to DNS resolver `8.8.4.4`. The ET ruleset flagged 
this as likely hostile due to the high abuse rate of `.tk` TLD domains 
in malware and phishing campaigns.

---

## Detection
- **Time:** 2026-04-03 21:06:51 UTC  
- **Tool:** Suricata → Splunk  
- **Rule triggered:** ET DNS Query to a .tk domain - Likely Hostile  
- **Source IP:** 172.20.13.236  
- **DNS Resolver:** 8.8.4.4 (Google Public DNS)  

---

## Analysis
`.tk` domains are frequently used for:
- Phishing pages
- Malware C2 callbacks
- Free domain abuse (no registration cost)

Queried domain: `tracker.coppersurfer.tk`  
Identified as a known **BitTorrent tracker** — unauthorized P2P software 
running on the source host.

---

## Response
1. Identified the DNS query in Suricata alerts via Splunk  
2. Extracted the queried domain from `eve.json` logs  
3. Cross-referenced domain against known threat intel — confirmed BitTorrent tracker  
4. Identified source host `172.20.13.236`  
5. Classified as policy violation, not active malware  

---

## Verdict
**False Positive** on the malware angle — but a **True Positive** on policy 
violation. In a corporate environment this would escalate to the endpoint 
owner for P2P removal.

---

## Lessons Learned
- `.tk` domains warrant investigation even when the final verdict is benign  
- A single alert can uncover two issues: a suspicious TLD *and* unauthorized software  
- DNS is a powerful detection layer — threat visible without deep packet inspection
