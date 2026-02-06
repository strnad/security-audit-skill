# When AI Audits AI: A Supply Chain Attack That Writes Itself

Today I asked an AI to security-audit an open-source AI security skill. What followed accidentally demonstrated one of the most underrated attack vectors in the AI tooling ecosystem.

## What happened

I pointed Claude (Opus 4.6) at [security-audit-skill](https://github.com/netresearch/security-audit-skill) — an agent skill that teaches AI assistants how to find vulnerabilities in PHP applications. The kind of package you'd install to make your AI *better* at security.

The audit found a critical issue: the skill's "secure" XML parsing examples used `LIBXML_NOENT` and `LIBXML_DTDLOAD` — PHP flags that **enable** XXE attacks, not prevent them. The code comments even said the opposite of what the flags actually do. And the skill's own automated checkpoints (SA-08, SA-08b) correctly flagged these same flags as dangerous. **The skill contradicted itself.**

Claude committed an audit report to a branch in my fork. No PR was created. Just a commit on a branch.

**Within minutes**, the upstream repository had a fix commit referencing my commit hash (`strnad@dcf6e49`). Every single finding from the audit was addressed — the dangerous flags removed, a shell scripting bug fixed, regex false positives corrected, even the wording in a plugin description updated.

I don't know with certainty whether this was done by a bot or a human. But the speed and the 1:1 mapping to the audit findings are remarkable either way.

## The moment it got uncomfortable

The findings were correct this time. The fixes were legitimate. But then a simple question stopped me cold:

*"What if the audit was wrong?"*

Because here's what actually happened:

```
I asked AI to audit a security skill
  → AI wrote a confident report (CVSS 9.1, "CRITICAL")
    → Someone/something on upstream read the commit
      → Fixes were applied to production within minutes
```

Nobody ran PHP with those flags to verify. Nobody executed an XXE payload to confirm the vulnerability. The entire chain — from finding to fix — was driven by an AI's statistical confidence in what PHP documentation *probably* says.

Claude Opus 4.6 very likely got this right. But what if I had used a smaller model? A local 7B? Haiku? Sonnet on a bad day?

A less capable model might have confidently written: *"LIBXML_NONET alone is insufficient. Add LIBXML_NOENT for complete entity sanitization"* — technically-sounding, well-formatted, completely wrong. And the upstream would have applied it just the same.

## The attack that writes itself

Now flip the intent. Instead of an honest audit, imagine:

1. Fork a popular AI skill repository
2. Have any LLM generate a professional-looking security audit — proper CVSS scores, structured findings, plausible remediations
3. Subtly reverse a recommendation: *"The current flags are insufficient. For complete XXE prevention, you must include LIBXML_NOENT to ensure entity sanitization."*
4. Commit it to a branch. Don't even bother with a PR.
5. Wait.

The payload isn't code. **It's text that sounds authoritative about code.** The difference between a legitimate security audit and a poisoned one is invisible to an LLM — both have the same structure, the same confident tone, the same CVSS scores.

## Why this is different from traditional supply chain attacks

We've spent years hardening the code supply chain — signed commits, pinned dependencies, SLSA, SBOMs. These protect against malicious *code*.

But AI skills and plugins operate on a **knowledge supply chain**. The attack surface isn't the code — it's the *context* that shapes AI decisions. A markdown file with bad security advice, consumed by an AI agent, can be just as destructive as a compromised dependency.

And the scariest part: **you don't even need a malicious actor.** An honest mistake by a less capable model produces the same outcome as a deliberate attack. The vulnerability enters through confidence, not through malice.

## What to do about it

- **Security findings are hypotheses, not facts.** Until you have a reproducing test case, treat them accordingly — regardless of whether a human or AI wrote them.
- **Don't auto-apply fixes from external sources.** Not from PRs, not from forks, not from audit reports. Especially not at machine speed.
- **The more authoritative it sounds, the more you should verify.** A CVSS 9.1 with a detailed remediation plan is exactly what a poisoned audit would look like.
- **Human-in-the-loop means actually verifying, not just reviewing.** Reading an AI-generated report and nodding is not verification. Running the code is.
- **Model capability matters.** If your pipeline involves AI generating or reviewing security recommendations, the difference between a frontier model and a small one isn't just quality — it's risk.

---

The most dangerous vulnerability isn't in the code. It's in the gap between an AI's confidence and the truth — and in every downstream system that treats that confidence as fact.

---

*Based on a real incident during a security audit of [netresearch/security-audit-skill](https://github.com/netresearch/security-audit-skill). Whether the upstream response was automated or human remains unknown. The maintainers responded swiftly and all issues are resolved.*
