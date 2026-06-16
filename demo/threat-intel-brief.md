# Threat Intelligence Brief — GhostLoader Ransomware

**Threat Actor:** GhostLoader (tracked since Q1 2026)  
**Target Sector:** Pharmaceutical, Healthcare, Life Sciences  
**Severity:** Critical  
**Last Observed:** 2026-06-14

---

## Summary

GhostLoader is a ransomware-as-a-service (RaaS) group that has conducted at least nine confirmed intrusions against pharmaceutical manufacturers in North America and Western Europe since January 2026. The group gained initial access via spearphishing and exploited unpatched VPN appliances.

**Little Pharma** was added to the GhostLoader victim leak site on 2026-06-12. Incident responders confirmed the pre-encryption staging sequence described below was observed on seven endpoints before the encryption payload executed.

---

## Key TTP — Pre-Encryption Staging Chain

Before deploying the encryptor, GhostLoader consistently executes a two-step recovery-disabling sequence on the victim host:

**Step 1 — Disable Windows Recovery**
```
bcdedit.exe /set {default} recoveryenabled No
bcdedit.exe /set {default} bootstatuspolicy ignoreallfailures
```

**Step 2 — Delete Volume Shadow Copies**
```
vssadmin.exe delete shadows /all /quiet
wmic.exe shadowcopy delete
```

This sequence executes within a 2-minute window and is unique enough to be a high-fidelity detection signal — GhostLoader uses it in >95% of observed intrusions.

---

## Recommended Response

1. Create a detection rule for the bcdedit → VSS deletion sequence (see `act1-ghostloader-ransomware-staging.toml`)
2. Scope to Windows endpoints; correlate by `host.id` within a 2-minute window
3. Severity: **critical** — if this fires, ransomware execution is likely seconds away
