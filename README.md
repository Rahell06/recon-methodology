# Recon & Asset Discovery Methodology

My working methodology for **subdomain enumeration and asset discovery** on
authorized bug bounty targets. This repo documents *how I work* — the order I do
things in, the decisions behind each step, and why — rather than being a
drop-in scanner.

Everything here is written to run **only against assets that are explicitly in
scope** for a bug bounty programme. Scope enforcement is the first step, not an
afterthought (see [Scope discipline](#scope-discipline)).

---

## Principles

- **Scope is enforced in code, not by eye.** Before any tool touches a host, the
  target is checked against the programme's allowlist/exclusion list with
  anchored suffix matching — so `*.target.com` matches `sub.target.com` but never
  `evil-target.com`.
- **Passive before active.** I exhaust passive/CT sources before generating any
  active footprint against a host I've decided matters.
- **Non-200s are leads, not dead ends.** `401`/`403` responses feed a bypass
  workflow instead of being discarded.
- **Automation triages; humans find bugs.** Scanner output is ranked *below*
  tech-stack and logic signals, because scanners mostly surface duplicates.

---

## The workflow at a glance

```
scope intake  ──►  passive enum  ──►  resolve + probe  ──►  triage  ──►  manual testing
(allowlist +       (subfinder,        (httpx, wildcard     (tech-detect,  (Burp: params,
 code-enforced      amass -passive,    guard, status         nmap, nuclei)  APIs, JS
 scope check)       crt.sh, wayback)   bucketing)                           endpoints)
```

Full detail, flags, and decision rules: **[docs/METHODOLOGY.md](docs/METHODOLOGY.md)**

---

## Scope discipline

The part of this workflow I care about most. I don't judge scope by reading the
policy page — I turn it into an allowlist and enforce it with a deterministic
check that every tool input passes through.

[`tools/scope_checker.py`](tools/scope_checker.py) does anchored suffix matching
(not `fnmatch`), handles an exclusion list, can filter a whole URL file into
in-scope / out-of-scope, and refuses to guess on IPs/CIDRs (returns a warning
rather than a false positive). Out-of-scope assets I discover get logged but
never probed — the checker strips them from every downstream tool's input.

```bash
# Single asset
python3 tools/scope_checker.py sub.target.com -d "target.com,*.target.com"
# → IN SCOPE: sub.target.com

python3 tools/scope_checker.py evil-target.com -d "*.target.com"
# → OUT OF SCOPE: evil-target.com

# Filter a whole list, keeping only in-scope hosts
python3 tools/scope_checker.py --input-file live/all.txt -d "*.target.com" -x "blog.target.com" --output live/in_scope.txt
```

See [docs/scope-discipline.md](docs/scope-discipline.md) for the matching rules
and the reasoning.

---

## Toolchain

These are installed separately; this repo orchestrates and documents how I chain
them.

| Stage | Tools |
|-------|-------|
| Passive enum | subfinder (`-all`), amass (`-passive`), crt.sh, Wayback CDX |
| Resolve / probe | ProjectDiscovery httpx, dnsx |
| Triage | nmap, nuclei |
| URL / JS mining | gau, katana, LinkFinder |
| Content discovery | ffuf |
| Manual testing | Burp Suite |
| Wordlists | SecLists, n0kovo_subdomains, PayloadsAllTheThings (see [wordlists/REFERENCES.md](wordlists/REFERENCES.md)) |

> Deliberately **not** using Nikto — `httpx -tech-detect` plus targeted nuclei
> templates cover the same ground with less noise. See the methodology doc for why.

---

## Repository layout

```
.
├── README.md                 # this file
├── docs/
│   ├── METHODOLOGY.md         # the full step-by-step process
│   └── scope-discipline.md    # scope matching rules + rationale
├── tools/
│   └── scope_checker.py       # deterministic, code-enforced scope validator
├── examples/
│   └── scope.md               # per-target scope template
└── wordlists/
    └── REFERENCES.md          # upstream wordlist sources I pull from
```

---

## Authorization & ethics

This methodology is for **authorized** security testing only — bug bounty
programmes I'm enrolled in, or systems I own or have explicit permission to test.
Nothing here should be run against a target you are not authorized to assess.
Staying inside programme scope is the entire point of the first step.
