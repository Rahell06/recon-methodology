# Scope discipline

Staying inside programme scope is the first rule of bug bounty work. Getting it
wrong — probing an asset that only *looks* in scope — is how testers end up out
of scope and in trouble. So I don't rely on reading the policy carefully; I turn
scope into an allowlist and enforce it with a deterministic check.

## The matching rule

[`tools/scope_checker.py`](../tools/scope_checker.py) uses **anchored suffix
matching**, not shell-style `fnmatch`:

| Pattern | Matches | Does **not** match |
|---------|---------|--------------------|
| `*.target.com` | `sub.target.com`, `a.b.target.com` | `target.com`, `evil-target.com` |
| `target.com` | `target.com` (exactly) | `sub.target.com`, `nottarget.com` |

The dangerous case a naive check gets wrong is `evil-target.com` against
`*.target.com`. A substring or loose glob match would wrongly call it in scope.
Anchoring on the leading dot (`.target.com`) prevents that.

## How it's used

- **Exclusions are checked first.** If a host matches the exclusion list it's out,
  even if it also matches the allowlist.
- **IPs and CIDRs are not guessed.** The checker returns `False` plus a warning
  for anything that looks like an IP, so it never produces a false positive on an
  address — those are gated manually.
- **Every tool input is filtered.** Host and URL lists are passed through
  `--input-file` so no downstream tool receives an out-of-scope target.

## Examples

```bash
# Single host
python3 tools/scope_checker.py sub.target.com -d "target.com,*.target.com"
# → IN SCOPE

python3 tools/scope_checker.py evil-target.com -d "*.target.com"
# → OUT OF SCOPE

# Respect an exclusion
python3 tools/scope_checker.py blog.target.com -d "*.target.com" -x "blog.target.com"
# → OUT OF SCOPE

# Filter a whole list into an in-scope file
python3 tools/scope_checker.py --input-file recon/all.txt \
        -d "*.target.com" -x "blog.target.com" \
        --output recon/in_scope.txt

# Machine-readable output for pipelines
python3 tools/scope_checker.py api.target.com -d "*.target.com" --json
```

Exit code is `2` when the asset is out of scope (or the vuln class is excluded),
so it composes cleanly in shell pipelines.
