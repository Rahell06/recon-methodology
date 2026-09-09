# Wordlist references

The upstream lists I pull from for real hunts. These are **not** vendored here —
clone them yourself and point the workflow at them (e.g. `WORDLIST_BASE=$HOME/wordlists`).

| List | Use | Source |
|------|-----|--------|
| SecLists | General discovery, params, dirs, files | https://github.com/danielmiessler/SecLists |
| OneListForAll | Aggregated content-discovery list | https://github.com/six2dez/OneListForAll |
| fuzz4bounty | Bug-bounty-oriented fuzzing lists | https://github.com/Bo0oM/fuzz.txt |
| n0kovo_subdomains | ~3M subdomain candidates from CT logs | https://github.com/n0kovo/n0kovo_subdomains |
| PayloadsAllTheThings | Exploitation payloads by class | https://github.com/swisskyrepo/PayloadsAllTheThings |
| nuclei-templates | Community detection templates | https://github.com/projectdiscovery/nuclei-templates |

Compact defaults for a fast first pass (small, checked-in wordlists such as
`common.txt`, `api-endpoints.txt`, `params.txt`, `raft-medium-dirs.txt`,
`sensitive-files.txt`) can live alongside this file if you choose to vendor them.
