# 04 - Zeek Network Analysis Setup

> **Zeek and Suricata are complementary, not redundant.** Suricata fires alerts when traffic matches a bad signature. Zeek creates a *behavioral record* of every network connection - who talked to whom, for how long, how many bytes, using which protocols. When Suricata says "something bad happened," Zeek tells you *exactly what the full conversation looked like.*

---

## What Zeek Does That Suricata Doesn't

| Capability | Suricata | Zeek |
|---|---|---|
| Signature-based alerts | ✅ | ❌ (by default) |
| Full connection logs | Partial | ✅ (conn.log) |
| DNS query history | Alert only | ✅ Complete history |
| HTTP full transaction logs | Partial | ✅ Every request/response |
| File extraction | ✅ | ✅ With hashing |
| SSL/TLS certificate logging | Partial | ✅ Complete cert details |
| Protocol detection | ✅ | ✅ (more flexible) |
| Scripting / custom analytics | Limited | ✅ Full scripting language |
| Threat hunting pivot data | Limited | ✅ Ideal |

Think of it this way: Suricata is your burglar alarm; Zeek is your security camera system.

---
