# INC-002 — Attack Response: Privileged Command Execution Detected

**Date:** 2026-04-03  
**Severity:** High  
**Status:** Investigated — Simulated (Lab Environment)  
**Analyst:** Willis Nomena Andre  

---

## Summary
Suricata triggered a high-severity alert indicating a remote system returned 
a root-level command execution response. The signature `GPL ATTACK_RESPONSE 
id check returned root` typically indicates a successful exploitation attempt 
where the attacker ran `id` and received `uid=0(root)` in response — a strong 
indicator of post-exploitation activity.

---

## Detection
- **Tool:** Suricata → Splunk  
- **Rule triggered:** GPL ATTACK_RESPONSE id check returned root  
- **Severity:** 2 (High)  
- **Source IP:** 18.239.6.71 (testmynids.org)  
- **Destination IP:** 172.20.13.236  

---

## Analysis
The `id` command is commonly run by attackers immediately after gaining 
shell access to confirm privilege level. A response containing `uid=0(root)` 
means the attacker has full control of the system.

**Why this signature fires:**
- The HTTP response body contained a string matching `uid=0(root)`
- Suricata's GPL ruleset flags this pattern as post-exploitation activity
- Source IP `18.239.6.71` resolves to `testmynids.org` — a known IDS testing service

---

## Response
1. Alert identified in Splunk via Suricata sourcetype  
2. Source IP investigated — resolved to testmynids.org (IDS test service)  
3. Confirmed trigger was intentional lab test using `curl testmynids.org/uid0alrt`  
4. No lateral movement or further suspicious activity observed  
5. Classified as intentional simulation — no escalation required  

---

## Verdict
**True Positive** on the detection — Suricata correctly identified the 
pattern. **Benign** in context — traffic was intentionally generated to 
validate the detection pipeline.

---

## Lessons Learned
- `GPL ATTACK_RESPONSE id check returned root` is a critical signature 
  to monitor — in production this would be an immediate P1 incident  
- Validating your detection pipeline with known-good test traffic 
  (testmynids.org) is a best practice before going live  
- Even in a lab, documenting test-triggered alerts builds analyst discipline
