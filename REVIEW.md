# Code Review File Convention

Each service repo may carry one or more review files at `<repo>/review/CODE_REVIEW_<date>.md`, produced by a review pass over that service. These files are git-tracked (not gitignored) so the history of what was found and what was fixed is preserved over time, not just held locally.

## Marking a finding as fixed

When a finding described in a review file is fixed, update it in place — don't delete it, and don't move it elsewhere. Two things change:

1. **At the title level**: strike through the original finding title and append `— FIXED <date>`.
2. **Immediately after**: a short description of the fix employed — what changed, which file(s) were touched, any tests added, and the branch/commit it landed on.

An unfixed finding keeps a plain (non-struck) title, so open items stay scannable at a glance — skim the file and anything not struck through is still outstanding.

### Example

```markdown
### 14. ~~Argon2id cost is below current OWASP guidance~~ — FIXED 2026-08-17

Bumped `rsaKeySize` to 3072 in `swlib/crypto/keypair.go`. Keys rotate ~every 27 days
(checked hourly), so the extra keygen cost is negligible. Updated the
`TestCreateKeypair_PrivateKeyParseable` bit-length assertion to match. Committed on
`swlib` branch `fix/crypto-hardening`.
```

vs. an item still open:

```markdown
### 9. IP binding is fragile and can break refresh behind multi-hop proxies
```

## Why this format

- The strikethrough + `FIXED <date>` at the title level makes fixed vs. open status visible from a table of contents or a quick scroll, without reading every section body.
- The fix description is scoped to "what actually shipped," not a restatement of the original finding — the original title (struck through) already says what the problem was.
- Keeping fixed findings in place (rather than deleting them) preserves the review's history: what was found, when, and how it was resolved.
