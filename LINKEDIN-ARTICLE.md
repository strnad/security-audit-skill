# When AI Audits AI: A Supply Chain Attack That Writes Itself

Today I asked Claude to security-audit an open-source AI security skill — a package that teaches AI agents how to find vulnerabilities in PHP code.

The audit found a critical issue: the skill's "secure" XML parsing examples used PHP flags that actually enable XXE attacks, not prevent them. The comments in the code said the opposite of what the flags do. The skill's own checkpoints correctly flagged these flags as dangerous. It contradicted itself.

Claude committed an audit report to a branch in my fork. No PR was created.

Within minutes, the upstream repository had a fix commit referencing my commit hash. Every finding addressed — dangerous flags removed, shell bugs fixed, regex false positives corrected.

I don't know if this was a bot or a human. But the speed and 1:1 mapping to the findings are remarkable.

Then I asked myself: What if the audit was wrong?

The entire chain was:

AI writes confident report (CVSS 9.1, "CRITICAL") → upstream reads the commit → fixes applied to production within minutes

Nobody ran PHP with those flags to verify. Nobody executed an XXE payload. The whole chain ran on statistical confidence in what PHP docs probably say.

Opus 4.6 very likely got this right. But what if I had used a smaller model? A local 7B? It might have confidently written "LIBXML_NONET alone is insufficient, add LIBXML_NOENT for complete entity sanitization" — technically-sounding, well-formatted, completely wrong. And upstream would have applied it just the same.

Now flip the intent:

1. Fork a popular AI skill repo
2. Have any LLM generate a professional-looking security audit with CVSS scores
3. Subtly reverse a recommendation
4. Commit it. Don't even make a PR.
5. Wait.

The payload isn't code. It's text that sounds authoritative about code. An LLM cannot distinguish a legitimate audit from a poisoned one — both have the same structure, tone, and scores.

We've spent years hardening the code supply chain — signed commits, pinned deps, SLSA. But AI skills operate on a knowledge supply chain. The attack surface isn't code — it's context that shapes AI decisions. A markdown file with bad advice can be as destructive as a compromised dependency.

The scariest part: you don't need a malicious actor. An honest mistake by a less capable model produces the same outcome. The vulnerability enters through confidence, not malice.

What to do:

- Security findings are hypotheses until you have a reproducing test case
- Don't auto-apply fixes from external sources at machine speed
- Human-in-the-loop means running the code, not reading the report and nodding
- Model capability isn't just a quality difference — it's a risk difference

The most dangerous vulnerability isn't in the code. It's in the gap between AI's confidence and truth — and in every system that treats that confidence as fact.

Based on a real incident with netresearch/security-audit-skill.
