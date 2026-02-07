When AI Audits AI: A Supply Chain Attack That Writes Itself

Today I asked Claude to security-audit an open-source AI security skill — a package that teaches AI agents how to find vulnerabilities in PHP code.

The audit found a critical issue: the skill's "secure" XML parsing examples used PHP flags that actually enable XXE attacks, not prevent them. The comments said the opposite of what the flags do.

This means the first attack vector is trivial: if you install this skill and ask your AI to "secure my XML parsing against XXE," the AI follows the skill's examples and introduces the exact vulnerability you asked it to prevent. The skill is a backdoor delivery mechanism — and nobody involved has bad intentions.

But it got worse. Claude committed the audit report to a branch in my fork. No PR was created. Within minutes, the upstream repository had a fix commit referencing my commit hash. Every finding addressed.

I don't know if this was a bot or a human. But the speed and 1:1 mapping to the findings are remarkable.

Then I asked myself: what if the audit was wrong?

Nobody ran PHP with those flags to verify. Nobody executed an XXE payload. The whole chain — from finding to fix — ran on AI's statistical confidence in what PHP docs probably say.

Opus 4.6 very likely got this right. But what if I used a smaller model? A local 7B? It might have confidently written "LIBXML_NONET alone is insufficient, add LIBXML_NOENT for complete entity sanitization" — technically-sounding, well-formatted, completely wrong. And upstream would have applied it just the same.

Now flip the intent. Fork a popular AI skill repo. Have any LLM generate a professional-looking security audit with CVSS scores. Subtly reverse a recommendation. Commit it. Don't even make a PR. Wait.

The payload isn't code. It's text that sounds authoritative about code. An LLM cannot tell a legitimate audit from a poisoned one — both have the same structure, tone, and scores.

So there are two attack surfaces here. First: AI skills with wrong patterns silently inject vulnerabilities into every project that uses them. Second: AI-generated audit reports can poison upstream repos that consume them without verification.

We've spent years hardening the code supply chain — signed commits, pinned deps, SLSA. But AI skills operate on a knowledge supply chain. A markdown file with bad advice can be as destructive as a compromised dependency. And you don't even need a malicious actor. An honest mistake by a weaker model produces the same outcome.

Security findings are hypotheses until you have a reproducing test case. Human-in-the-loop means running the code, not reading the report and nodding. Model capability isn't just a quality difference — it's a risk difference.

The most dangerous vulnerability isn't in the code. It's in the gap between AI's confidence and truth.

Based on a real incident with netresearch/security-audit-skill.
