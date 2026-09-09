# Methodology

The full subdomain-enumeration and asset-discovery process I follow on
authorized bug bounty targets. Written to be reproducible: the steps, the flags
that matter, and the reasoning behind each decision.

Target names are deliberately kept generic (`target.com`) throughout.

---

## 1. Scope intake & enforcement

I pull a programme's assets into a single list rather than eyeballing the policy
page, then enforce that list in code.

- **Aggregate scope.** Read the in-scope domains and wildcards from the platform
  and write them to a per-target `scope.md` (see [examples/scope.md](../examples/scope.md)).
  Public mirrors such as [`bounty-targets-data`](https://github.com/arkadiyt/bounty-targets-data)
  (hourly dumps for HackerOne / Bugcrowd / Intigriti / YesWeHack) are a useful
  fallback for the raw scope.
- **Enforce in code, not by judgment.** [`scope_checker.py`](../tools/scope_checker.py)
  holds the allowlist + exclusion list and does **anchored suffix matching**, not
  `fnmatch`:
  - `*.target.com` matches `sub.target.com`, never `evil-target.com`
  - a bare `target.com` matches exactly
- **Out-of-scope assets I discover** get logged but never probed. The checker
  filters them out of every tool's input, so no downstream tool (httpx, nuclei,
  ffuf) ever touches them.
- **IPs / CIDRs** aren't handled by the checker (it returns `False` + a warning) —
  those I gate manually.

---

## 2. Passive / CT enumeration

Passive first — no active footprint until I've decided a host matters. Order:

1. `subfinder -silent -all -t 50` — aggregates its own passive source set,
   including Certificate Transparency logs.
2. `amass enum -passive` — capped at ~5 minutes; skipped in a quick pass.
3. **crt.sh** direct JSON: `https://crt.sh/?q=%25.target.com&output=json`,
   parsed to drop wildcards and off-domain names.
4. **Wayback CDX**: `http://web.archive.org/cdx/search/cdx?url=*.target.com/*&collapse=urlkey`,
   host labels extracted by regex.

Everything is merged → lowercased → wildcard prefix stripped (`sed 's/^\*\.//'`)
→ deduped into a single `subdomains/all.txt`.

> I don't call CertSpotter as a separate step — subfinder already folds CT
> sources in, so a dedicated call would mostly duplicate results.

---

## 3. Active enumeration

- **subfinder is the workhorse.** API keys live in its provider config so the
  passive sources return more results.
- **No brute-forcing by default.** The wildcard pre-check (see §4) makes
  brute-forcing wasteful on CDN-fronted zones, so it's not in the standard
  pipeline. When a specific zone is interesting enough to brute-force manually, I
  use a large CT-derived list (e.g. `n0kovo_subdomains`) with a fast resolver.
- **Assetfinder** isn't in my standard chain — subfinder + amass + CT already
  cover the same ground.

---

## 4. Resolving & probing live hosts

**Wildcard guard runs first.** Three random labels are queried at the apex; if
2+ resolve, the zone is a wildcard and gets recorded so junk collapsing to the
wildcard IP is filtered before probing.

Then httpx over `subdomains/all.txt`:

```
httpx -silent -status-code -title -tech-detect -content-length \
      -follow-redirects -threads 50 -rate-limit 150
```

- The real **ProjectDiscovery httpx** binary is resolved explicitly (the Python
  `httpx` CLI silently ignores these flags — a common footgun).
- Output is bucketed by status: `status_200`, `status_3xx`, `status_403`,
  `status_401`. The `401`/`403` buckets are **leads for a bypass workflow**, not
  dead ends.
- Live URLs are extracted to `live/urls.txt` for every downstream tool.

---

## 5. Triage before manual testing

This is where I diverge from the generic tutorial — **no Nikto.** The triage
order:

1. **`httpx -tech-detect`** (already collected in §4) — the tech stack is the
   first triage signal. Known software → check CVEs / default creds immediately.
2. **`nmap -sV --top-ports 1000 -T4 --open`** → `open_ports.txt`.
3. **nuclei**, two ways:
   - broad pass, gated by severity (`high,critical` on a quick run; `+medium`
     on a full run) with `-rl 300 -c 50 -bs 50`
   - focused CVE pass: `-update-templates`, then `-tags cve` (optionally a
     specific year), high/critical only.

**Decision rule for what earns manual attention:** tech-stack anomalies +
auth-bearing endpoints + the param/API/JS buckets outrank raw nuclei hits,
because scanner hits are usually duplicates. Signals are tracked so nothing gets
lost between sessions.

---

## 6. Manual testing handoff (Burp)

Into Burp, in priority order:

1. **Parameterized URLs** — pre-bucketed as `with_params.txt`.
2. **API endpoints** — matched on `/api/`, `/v[0-9]/`, `/graphql`, `/rest/`.
3. **JS-derived endpoints** — extracted from JavaScript.

First things I look at:

- ID-bearing params → IDOR / BOLA
- the auth model → cookie / JWT / OAuth / SAML
- business-critical flows → payment, password reset, data export
- hidden JSON params

If I'm hunting auth bugs, I authenticate once and share that session across Burp
and the CLI tools so everything tests as the authenticated user.

---

## 7. Wordlists & data sources

- **Compact defaults** for fast fuzzing: `common.txt`, `api-endpoints.txt`,
  `params.txt`, `raft-medium-dirs.txt`, `sensitive-files.txt`.
- **Upstream, for real hunts** (see [wordlists/REFERENCES.md](../wordlists/REFERENCES.md)):
  SecLists, OneListForAll, fuzz4bounty, `n0kovo_subdomains` (subdomains),
  PayloadsAllTheThings (payloads).
- nuclei uses its own community templates (`-update-templates`), plus fresher
  community PoC template sets.

---

## 8. Note-taking & state

- **Per-target folder** `targets/<name>/` with a `NOTES.md` running log: leads,
  killed items, blockers, and where to start the next session.
- **Structured state** in a per-target JSONL lead ledger that survives re-runs
  and preserves each lead's status (investigating / killed / reported).
- Recon output stays in bucketed subdirectories so notes can reference exact
  `file:line`-style paths.

---

## 9. What I do differently from a generic tutorial

- **DNS wildcard pre-check gates brute-forcing** — wildcard zones are detected
  before wasting time probing hosts that all collapse to one IP. Most tutorials
  brute-force blind.
- **Scope is enforced in code, not by eye** — anchored suffix matching kills the
  classic `evil-target.com` false positive, and every tool input is filtered.
- **One auth session, inherited by all tools** — httpx / katana / ffuf / nuclei
  all run authenticated instead of re-logging-in per tool.
- **Recon output is routed to a lead board, not just dumped** — every signal
  becomes a tracked lead with a status, so nothing is forgotten across sessions.
- **Status buckets are leads** — `401`/`403` feed a bypass workflow; non-200s
  aren't discarded.
- **Scanner hits rank below logic / tech-stack signals** — automation finds
  duplicates; the unique bugs come from manually-triaged param/API/JS buckets.
