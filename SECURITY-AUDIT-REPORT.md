# Security Audit Report: security-audit-skill

**Date:** 2026-02-06
**Repository:** netresearch/security-audit-skill
**Version:** 1.3.1
**Auditor:** Automated security audit (Claude)
**Scope:** Full repository — all source code, scripts, configurations, CI/CD, and documentation

---

## Executive Summary

This repository is an AI agent skill that provides security audit patterns, reference documentation, and tooling for PHP/OWASP security assessments. It contains 2 executable scripts (Python, Bash), configuration files (hooks, composer, plugin manifest), a CI/CD workflow, and extensive reference documentation.

**The repository does not contain malware or intentionally malicious code.** It is a legitimate knowledge package. However, the audit identified **1 critical**, **2 high**, **4 medium**, and **3 low** severity findings. The critical finding is particularly concerning because the repository's stated purpose is to teach secure coding, but its own reference documentation contains dangerously incorrect XXE prevention examples that would actually *enable* XXE attacks.

| Severity | Count |
|----------|-------|
| Critical | 1 |
| High | 2 |
| Medium | 4 |
| Low | 3 |

---

## Findings

### FINDING-01: XXE "Secure" Code Examples Actually Enable XXE Attacks

**Severity:** CRITICAL
**CVSS:** 9.1 (CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N)
**Files affected:**
- `skills/security-audit/SKILL.md:29`
- `skills/security-audit/references/xxe-prevention.md:70-74`
- `skills/security-audit/references/xxe-prevention.md:136`
- `skills/security-audit/references/xxe-prevention.md:170`
- `skills/security-audit/references/xxe-prevention.md:208`
- `skills/security-audit/references/xxe-prevention.md:220`
- `skills/security-audit/references/xxe-prevention.md:236`

**Description:**

Multiple code examples labeled as "secure" use the flags `LIBXML_NOENT` and `LIBXML_DTDLOAD`, which **enable** XXE rather than prevent it:

- `LIBXML_NOENT` (value 2): **Enables** entity substitution. When external entities are defined, this flag causes them to be resolved and expanded, which is the core mechanism of XXE attacks.
- `LIBXML_DTDLOAD` (value 4): **Enables** loading of external DTD subsets, allowing attackers to define external entities via remote or local DTDs.

Example from `SKILL.md:29` (labeled "XML parsing (prevent XXE)"):
```php
$doc->loadXML($input, LIBXML_NONET | LIBXML_NOENT | LIBXML_DTDLOAD);
```

Example from `xxe-prevention.md:70-74` (labeled "Secure flags"):
```php
$flags = LIBXML_NONET          // Disable network access
       | LIBXML_NOENT          // Substitute entities       <-- DANGEROUS
       | LIBXML_DTDLOAD        // Don't load external DTD   <-- WRONG COMMENT & DANGEROUS
       | LIBXML_DTDATTR
       | LIBXML_PARSEHUGE;
```

The comment on `LIBXML_DTDLOAD` says "Don't load external DTD" but the flag does the **exact opposite** — it enables external DTD loading.

**Internal contradiction:** The repository's own `checkpoints.yaml` correctly identifies these flags as dangerous:
- SA-08: "PHP files must not use LIBXML_NOENT flag that enables XXE"
- SA-08b: "PHP files must not use LIBXML_DTDLOAD flag that enables XXE"

This means the checkpoints would flag the repository's own reference documentation as vulnerable.

**Impact:** Any developer following these "secure" examples will introduce XXE vulnerabilities into their application. Given that this skill is specifically designed to teach XXE prevention, the impact on downstream consumers is severe.

**Remediation:** Remove `LIBXML_NOENT` and `LIBXML_DTDLOAD` from all "secure" code examples. The correct secure flags are:
```php
$flags = LIBXML_NONET;   // Disable network access - this is the key safe flag
```
For PHP < 8.0, also call `libxml_disable_entity_loader(true)`.

---

### FINDING-02: security-audit.sh Treats LIBXML_NOENT as a Mitigation

**Severity:** HIGH
**File:** `skills/security-audit/scripts/security-audit.sh:50`

**Description:**

The XXE check in the audit script considers the presence of `LIBXML_NOENT` as evidence that XML parsing is secured:

```bash
SECURED=$(grep -rn "LIBXML_NOENT\|LIBXML_NONET\|libxml_disable_entity_loader" \
  "$PROJECT_DIR/src" --include="*.php" 2>/dev/null | wc -l || echo "0")
if [[ "$SECURED" -eq 0 ]]; then
    echo "XML parsing found without obvious XXE protection:"
```

Since `LIBXML_NOENT` actually **enables** entity substitution (a prerequisite for XXE), its presence should be treated as a **vulnerability indicator**, not a mitigation. Code using `LIBXML_NOENT` would pass the audit as "secured" when it is actually vulnerable.

**Remediation:** Remove `LIBXML_NOENT` from the secured-patterns grep. Only `LIBXML_NONET` and `libxml_disable_entity_loader(true)` should be treated as mitigations.

---

### FINDING-03: `set -e` Combined with Arithmetic Causes Silent Abort

**Severity:** HIGH
**File:** `skills/security-audit/scripts/security-audit.sh:6,23`

**Description:**

The script uses `set -e` (exit on error) at line 6 and `((WARNINGS++))` at multiple lines (23, 34, 53, 71, 85, 99, 115, 121, 125, 138, 150). When `WARNINGS` is 0, the post-increment expression `((0++))` evaluates to 0 (falsy), which gives a non-zero exit code. Under `set -e`, this terminates the script immediately.

This means the audit will **silently abort after detecting the first warning**, producing an incomplete audit report. Users would see only one warning and a truncated output, potentially believing their code passed checks that were never actually executed.

**Impact:** False sense of security — vulnerabilities may be missed because the audit didn't run to completion.

**Remediation:** Use `WARNINGS=$((WARNINGS + 1))` instead of `((WARNINGS++))`, or use `((WARNINGS++)) || true` to suppress the non-zero exit code.

---

### FINDING-04: Wildcard Composer Dependency Version Constraint

**Severity:** MEDIUM
**File:** `composer.json:16`

**Description:**

```json
"require": {
    "netresearch/composer-agent-skill-plugin": "*"
}
```

The `*` constraint accepts any version of the dependency, including potentially compromised or incompatible versions. This is a supply chain risk — if the `netresearch/composer-agent-skill-plugin` package on Packagist were compromised, any installation of this skill would pull the malicious version.

**Remediation:** Pin to a specific version range, e.g., `"^1.0"`.

---

### FINDING-05: PreToolUse Hook Is Informational Only (No Enforcement)

**Severity:** MEDIUM
**Files:** `scripts/check_risky_command.py:3`, `hooks/hooks.json`

**Description:**

The hook script's docstring explicitly states it is "informational only" and will not block dangerous commands. The script outputs warnings via `<system-reminder>` tags but always exits with code 0, meaning dangerous commands (like `rm -rf /`, `curl | sh`, etc.) will proceed after the warning is displayed.

Users installing this skill may expect that the "PreToolUse guards for risky commands" (as described in `plugin.json:4`) actually prevent execution of dangerous commands.

**Impact:** Users may over-rely on the hook's protection. The gap between the description ("guards") and the behavior ("informational warnings") could lead to a false sense of security.

**Remediation:** Either update the description to clearly state "informational warnings" instead of "guards", or implement actual blocking by returning a non-zero exit code for high-severity matches.

---

### FINDING-06: Regex Pattern Bypass Opportunities in Hook Script

**Severity:** MEDIUM
**File:** `scripts/check_risky_command.py:12-86`

**Description:**

Several detection patterns can be trivially bypassed:

| Pattern | Bypass |
|---------|--------|
| `rm -rf /` | `rm --recursive --force /`, `find / -delete` |
| `curl\|sh` | `curl ... \| python`, `curl ... \| perl`, `wget -qO- ... \| bash` (different from detected pattern) |
| `chmod 777` | `chmod a=rwx`, `chmod u=rwx,g=rwx,o=rwx` |
| Hardcoded creds | Using backticks or `$()` substitution instead of quotes |
| `git push --force` | `git push --force-with-lease` triggers a false positive (it's actually safer) |

**Impact:** Reduced effectiveness of the warning system. Since the hook is informational-only (FINDING-05), the practical impact is limited.

**Remediation:** Expand patterns to cover alternative syntaxes, or document the limitations. Also exclude `--force-with-lease` from the force-push pattern since it's a safer alternative.

---

### FINDING-07: CI Egress Policy Is Audit-Only

**Severity:** MEDIUM
**File:** `.github/workflows/release.yml:20`

**Description:**

```yaml
- name: Harden Runner
  uses: step-security/harden-runner@0634a267... # v2.12.0
  with:
    egress-policy: audit
```

The `egress-policy: audit` only logs outbound network connections during CI but does not block unauthorized ones. If the CI pipeline were compromised (e.g., through a dependency), data could be exfiltrated. Using `egress-policy: block` with an explicit allowlist would be more secure.

**Remediation:** Consider upgrading to `egress-policy: block` with allowed endpoints listed.

---

### FINDING-08: Missing SECURITY.md Vulnerability Disclosure Policy

**Severity:** LOW
**File:** (missing)

**Description:**

The repository's own checkpoints (SA-01) require a `SECURITY.md` file for vulnerability reporting, but the repository itself does not have one. A security audit skill should practice what it preaches.

**Remediation:** Add a `SECURITY.md` with responsible disclosure instructions.

---

### FINDING-09: Referenced Documentation Files Do Not Exist

**Severity:** LOW
**Files:** `skills/security-audit/SKILL.md:24-25`, `README.md:74-75`

**Description:**

Both `SKILL.md` and `README.md` reference two files that do not exist in the repository:
- `references/secure-php.md`
- `references/secure-config.md`

This causes broken references for users of the skill.

**Remediation:** Either create the missing files or remove the references.

---

### FINDING-10: Force Push Pattern False Positive

**Severity:** LOW
**File:** `scripts/check_risky_command.py:82`

**Description:**

The pattern `r"git\s+push\s+(-f|--force).*\b(main|master)\b"` also matches `git push --force-with-lease main`, which is a **safer** alternative to `--force` that prevents overwriting commits others have pushed. Flagging it as "high severity" is a false positive that could train users to ignore warnings.

**Remediation:** Adjust the regex to exclude `--force-with-lease`, e.g., use a negative lookahead: `(-f|--force)(?!-with-lease)`.

---

## Positive Findings

The audit also identified several good security practices:

1. **Pinned CI action versions** — All GitHub Actions in `release.yml` use full SHA commit hashes instead of mutable tags, preventing tag-hijacking attacks.
2. **Hardened CI runner** — The `step-security/harden-runner` action is used for CI security monitoring.
3. **No hardcoded secrets** — No API keys, passwords, tokens, or credentials were found anywhere in the codebase.
4. **Properly quoted shell variables** — `$PROJECT_DIR` is consistently double-quoted in `security-audit.sh`, preventing word splitting and glob expansion issues.
5. **Minimal attack surface** — The repository has no runtime server components, no network listeners, no database connections, and no user authentication. It's a passive knowledge package with two standalone scripts.
6. **Defensive input handling in Python hook** — `check_risky_command.py` safely handles malformed JSON input and empty stdin.
7. **2-second timeout on hooks** — The hook configuration prevents hanging if the script stalls.
8. **MIT license** — Clear, permissive licensing with no ambiguity.

---

## Trust Assessment

**Can you trust this repository?**

The repository is **not malicious** — it contains no backdoors, data exfiltration, obfuscated code, or intentionally harmful behavior. It is a well-intentioned security knowledge package.

However, **the XXE prevention examples are actively dangerous** (FINDING-01). Anyone copying the "secure" XML parsing patterns from this skill into production code would actually be introducing XXE vulnerabilities. This is the most critical concern and should be fixed before the skill is used as a security reference.

The `security-audit.sh` script also has two bugs (FINDING-02, FINDING-03) that could produce false negatives and incomplete results.

**Recommendation:** The skill is safe to install (it won't harm your system), but **do not follow the XXE code examples** until they are corrected. The remaining findings are lower severity and represent typical areas for improvement in an open-source project.

---

## Methodology

This audit examined:
- All executable code (`scripts/check_risky_command.py`, `skills/security-audit/scripts/security-audit.sh`)
- All configuration files (`hooks/hooks.json`, `composer.json`, `.claude-plugin/plugin.json`)
- CI/CD pipeline (`.github/workflows/release.yml`)
- Skill definition and checkpoints (`SKILL.md`, `checkpoints.yaml`)
- All reference documentation (`xxe-prevention.md`, `owasp-top10.md`, `cvss-scoring.md`, `api-key-encryption.md`)
- `README.md` and `LICENSE`
- Git history (20 most recent commits)

Checks performed: command injection, path traversal, hardcoded secrets, supply chain risks, CI/CD security, XXE/XSS/SQLi in documentation correctness, regex bypass analysis, error handling, and internal consistency.
