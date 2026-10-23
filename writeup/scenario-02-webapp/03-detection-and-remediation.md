[← Previous: Exploitation](./02-exploitation.md) | [Back to Scenario 2 Index](./README.md)

# Phase 3 — Detection, Root Cause Analysis & Remediation

## 3.1 Initial Detection Failure — SQL Injection

Neither SQLi payload produced an alert on first attempt, despite Suricata's Emerging Threats ruleset containing multiple signatures explicitly designed for this purpose (SIDs in the 2020506–2020508 range, `msg:"ET ATTACK_RESPONSE MySQL error in HTTP response..."`, matching on literal content such as `"SQL syntax"` and `"mysql_"` in HTTP responses).

**Root cause investigation:**
- Verified HTTP traffic to the target was being captured correctly at the network layer (confirmed via raw `eve.json` HTTP events) — ruling out a capture problem.
- Inspected the matching Suricata rule's header: `$HTTP_SERVERS any -> $EXTERNAL_NET any`. This is a *response*-direction rule — it looks for the server's reply to the attacker, not the attacker's request.
- Checked the relevant `suricata.yaml` variables:
  ```yaml
  HOME_NET: "[192.168.0.0/16,10.0.0.0/8,172.16.0.0/12]"
  EXTERNAL_NET: "!$HOME_NET"
  ```
  Since the entire lab (including the attacker, `192.168.56.102`) falls within `192.168.0.0/16`, **no address in the lab was ever classified as "external."** As a result, rules depending on `$EXTERNAL_NET` — both request-direction and response-direction SQLi signatures — could never match, regardless of payload.

**Fix:**
```bash
sudo sed -i 's/EXTERNAL_NET: "!\$HOME_NET"/EXTERNAL_NET: "any"/' /etc/suricata/suricata.yaml
sudo systemctl restart suricata
```

**Re-test result:** the `'` (error-based) payload, resubmitted after the fix, triggered the response-based alert correctly, confirmed via `src_ip`/`dest_ip` matching the attacker and victim respectively.

**Secondary finding:** the boolean-bypass payload (`' OR '1'='1`) — even after the fix — did **not** trigger any SQLi-related alert. This is expected, not a bug: the rule family in use detects *error signatures* in the HTTP response body. A successful bypass produces a normal, error-free response, leaving no textual trace for this detection approach to catch. This is documented as a detection-coverage limitation in Lessons Learned below.

## 3.2 Initial Detection Failure — Webshell Access

No alert of any kind was generated for the webshell upload or its subsequent command execution, despite the HTTP request itself being fully captured in `eve.json`.

**Root cause:** the Emerging Threats Open ruleset contains no generic signature for "PHP script accessed from an upload directory" — a pattern specific to this application's vulnerability, not a widely-known exploit signature. The request (`GET /dvwa/hackable/uploads/shell.php?cmd=id`) is, from a purely network-signature perspective, indistinguishable from a normal HTTP request.

**Remediation — custom rule:**
```
alert http any any -> $HOME_NET any (msg:"CUSTOM Possible Webshell Access in Upload Directory"; flow:established,to_server; http.uri; content:"/uploads/"; content:".php"; distance:0; classtype:web-application-attack; sid:9000001; rev:1;)
```
The rule flags any HTTP request whose URI contains both `/uploads/` and `.php` — a simple but effective heuristic for this specific vulnerability class.

**Deployment issue encountered:** the rule file was initially placed at `/etc/suricata/rules/local.rules` and referenced by filename only in `rule-files:`. Suricata resolves filenames in that list relative to `default-rule-path` (confirmed as `/var/lib/suricata/rules/` via `suricata.yaml`), not the location the file was actually created in — resulting in a silent "No rule files match the pattern" warning on every restart, with no syntax error to indicate the real cause. Moving the file to the correct default rule path resolved the issue immediately, with no changes to the rule's syntax required.

**Verification:** after the fix, re-accessing the webshell triggered an alert with `signature_id: 9000001`, correctly identifying source (`192.168.56.102`) and destination (`192.168.56.112`).

## 3.3 Containment & Eradication

**Eradication** — the webshell was removed using legitimate administrative access (SSH as `msfadmin`), not through the webshell itself:
```bash
ssh -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedAlgorithms=+ssh-rsa msfadmin@192.168.56.112
sudo rm /var/www/dvwa/hackable/uploads/shell.php
```
(The `HostKeyAlgorithms`/`PubkeyAcceptedAlgorithms` overrides were required because Metasploitable2's outdated SSH server only supports algorithms deprecated by modern SSH clients.)

Verified via a subsequent request to the webshell's URL, which returned `404 Not Found`.

**Containment** — the attacker's IP was blocked at the firewall level on the victim host:
```bash
sudo iptables -A INPUT -s 192.168.56.102 -j DROP
```
Verified via a failed ping test from the attacker (100% packet loss).

**Note on execution order:** eradication was performed before containment in this engagement, the reverse of the generally recommended order (block first, to prevent the attacker from re-establishing access during cleanup; remove second). This did not materially impact the lab exercise, but is flagged as a process deviation below.

## 3.4 Impact Assessment

- **Confidentiality:** Full disclosure of the `users` table contents via SQLi bypass; arbitrary file read/system information disclosure possible via the webshell (not exhaustively tested beyond `id`).
- **Integrity:** Demonstrated arbitrary command execution as `www-data` — sufficient to modify web-accessible files, though no destructive changes were made beyond the webshell itself.
- **Availability:** No service disruption observed.
- **Scope:** Contained to the isolated lab network; no lateral movement attempted or observed.

## 3.5 Remediation & Recommendations

**Immediate:**
- Webshell removed; attacker IP blocked (both completed).

**Short-term:**
- Review `HOME_NET`/`EXTERNAL_NET` definitions in any Suricata deployment before relying on perimeter-oriented signatures, especially in flat/single-segment network labs.
- Add file-upload content-type and extension validation to any custom or test web applications retained for future exercises (where feasible without breaking the intentional vulnerability).

**Long-term:**
- Expand custom rule coverage for other upload-based web attack patterns (e.g., double extensions, null-byte tricks) as future scenarios are developed.
- Build a small internal library of validated custom Suricata rules, version-controlled alongside this repository, as organizational detection content matures.

---

## Lessons Learned

- Response-based (error-signature) SQLi detection has a structural blind spot: the *more successful* an injection is (i.e., the less it disrupts normal query execution), the *less likely* it is to be detected by this method. A fully successful boolean-based bypass can be invisible to this detection approach entirely.
- Default signature sets (Emerging Threats Open) do not automatically cover application-specific vulnerability patterns such as unrestricted file upload — targeted custom rules remain necessary for adequate coverage of bespoke or lesser-known applications.
- An overly broad `HOME_NET` definition can silently disable every rule that depends on `EXTERNAL_NET`, with no error or warning at startup — this should be validated explicitly during any new Suricata deployment, not assumed correct from defaults.
- Suricata rule file placement must match `default-rule-path` (or use an absolute path) — registering a file under `rule-files` alone is not sufficient if the file isn't where Suricata expects to find it; the resulting failure mode (a startup warning) is easy to miss.
- VirtualBox's "Allow All" promiscuous mode setting at the hypervisor level does not automatically propagate into the guest OS network stack — it must be explicitly enabled at the OS level and made persistent (e.g., via a dedicated systemd unit), or it silently reverts after reboot.
- The ideal containment order is to block attacker access *before* eradicating artifacts, to minimize the window in which an attacker could re-establish persistence; this order was reversed in this engagement without material consequence in the lab context, but is noted as a process improvement for future exercises.

---

[← Previous: Exploitation](./02-exploitation.md) | [Back to Scenario 2 Index](./README.md)
