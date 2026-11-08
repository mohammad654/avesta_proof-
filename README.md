# Avesta Proof

**Turn code scans into audit evidence.** Avesta Proof scans a repository, maps what it finds to
ISO/IEC 27001, IEC 62443-4-1 and the EU Cyber Resilience Act (CRA), and produces a signed,
tamper-evident report with a fix for every finding. Your code never leaves your machine.

Built by Avesta Tech, written from scratch. Design ideas came from studying a C/C++ quality tool
(see `docs/ccqat-review.md`); no code was copied.

## Why it is different

| | Typical scanner | Avesta Proof |
|---|---|---|
| Output | A list of findings | Findings **and** a per-control verdict for 3 standards |
| Priority | Severity only | Actively exploited first (CISA KEV), then likelihood (EPSS), then severity |
| Trust | A PDF anyone could edit | Ed25519-signed report + hash-chained scan history an auditor can verify offline |
| Manual evidence | Out of scope | Attestations (pen test, threat model) that **expire**, tracked like tests |
| Exceptions | Hidden suppressions | Accepted risks with reason, approver and expiry; shown in the report |
| CRA | A blog post | Technical-file draft (Annex VII), SBOM, key-date clock, support-period check |
| Report | A table of findings | A designed document: score dial, risk matrix, risk treemap, control grid, trend line, language mix - all offline SVG, no JavaScript needed |
| Re-use | Re-scan to change anything | `render` rebuilds every report from the signed JSON, re-branded, in under a second |
| Sales | n/a | One-page trust statement and badge for customers |
| Honesty | Green when a tool is missing | A skipped tool makes its controls **partial**, never a silent pass |

## Install

```bash
pip install -e ".[scanners,signing]"     # Python 3.11+
# optional external tools, each adds evidence:
#   trivy (dependency CVEs, misconfiguration), gitleaks (secrets), cppcheck (C/C++)
avesta-proof doctor                       # shows what is installed
```

## Five-minute tour

```bash
avesta-proof init  ./my-product                  # writes avesta-proof.toml to fill in
avesta-proof keygen --dir ~/.avesta              # signing key pair (keep the .key secret)
avesta-proof scan ./my-product -o proof --format all --sign ~/.avesta/avesta-proof.key
open proof/report.html
avesta-proof verify proof/report.json --pubkey ~/.avesta/avesta-proof.pub
avesta-proof verify-ledger proof
```

`--format all` writes `report.html`, `report.json`, `report.sarif` (GitHub/GitLab code
scanning), `sbom.cdx.json` (CycloneDX 1.5), `trust-statement.html`, `badge.svg` and
`cra-technical-file.md`. Add `pdf` if you installed `weasyprint`.

Run it again later: the report shows what is **new** and what got **fixed** since the last scan,
and the ledger grows by one linked entry.

## Commands

| Command | What it does |
|---|---|
| `scan PATH` | Run scanners, write reports. Key flags: `--format`, `--only`, `--fail-on high`, `--new-only`, `--baseline auto\|none\|FILE`, `--sign KEY`, `--offline`, `--kev-file FILE`, `--run-tests`, `--semgrep-config bundled` |
| `render REPORT.json` | Rebuild `report.html`, `trust-statement.html`, `badge.svg`, `report.sarif` or `pdf` from an existing report, with an optional `--theme FILE`. No scanners, no network |
| `init [PATH]` | Create `avesta-proof.toml` |
| `keygen` | Create an Ed25519 key pair |
| `verify REPORT [--pubkey PUB]` | Check hash and signature (exit 3 if invalid) |
| `verify-ledger [DIR]` | Check the scan-history chain and snapshots (exit 3 if broken) |
| `doctor` | Show which tools are installed |

Exit codes: `0` ok, `1` gate failed, `2` usage error, `3` verification failed.

## What runs

| Scanner | Finds | Needs |
|---|---|---|
| `semgrep` | code vulnerabilities (many languages); falls back to bundled offline rules if the registry is unreachable | `pip install semgrep` |
| `bandit` | Python security issues (Python repos only) | `pip install bandit` |
| `cppcheck` | C/C++ memory and logic errors (C/C++ repos only) | cppcheck |
| `lizard` | complexity, function length, parameter count (limits in `[quality]`) | `pip install lizard` |
| `codebase` | size and language mix, comment density, copy-paste duplication, test-code share, oversized files | built in |
| `trivy` | dependency CVEs, container/IaC misconfiguration | trivy |
| `gitleaks` | hard-coded secrets | gitleaks |
| `coverage` | test presence and coverage (reads `coverage.xml/json`, `lcov.info`, or `--run-tests`) | built in |
| `hygiene` | SECURITY.md, SBOM, CI, lock files, dependency updates, LICENSE, CODEOWNERS, `.env` | built in |

A scanner that does not apply (no C files, no Python files) is marked **not-applicable** and
does not count against you. A scanner that applies but is missing is marked **skipped**.

## How a report is made

```
scanners (run in the repo, JSON out) -> parsers -> Finding objects
   -> dedupe -> classify (crypto) -> enrich (KEV/EPSS) -> SBOM -> risk acceptances -> baseline diff
   -> tag with controls -> evaluate controls (auto / manual / derived) -> score
   -> ScanResult -> report.json (+ fingerprint + signature) -> history snapshot + ledger entry
                 '-> ReportModel -> insights -> sections -> HTML | trust page | badge | SARIF | PDF
                                 '-> SBOM and CRA technical file (written during the scan)
```

**Analysis happens once; presentation happens as often as you like.** Everything below the
`ReportModel` line is rebuilt by `avesta-proof render report.json`, so a new logo, a shorter
executive edition or a customer-facing trust page costs a second and cannot change a number -
the model can only show what the signed document contains.

### What the report shows

| Section | What it answers |
|---|---|
| Executive summary | Score dial, the six numbers that matter, remediation effort in engineer-days, readiness per framework |
| Progress over time | Score trend from the signed snapshot history, new versus fixed |
| Risk and exploitability | Likelihood/impact matrix, what is on CISA's actively-exploited list, highest EPSS next |
| Fix these first | Ranked cards with the control each finding maps to and an effort estimate |
| Where the risk sits | Treemap of weighted risk per folder - concentration, not just counts |
| Control-by-control | Status square per control, counts per framework, the gaps that block a pass, then full tables |
| Codebase profile | Size, language mix, comment density, duplication, test-code share, peak complexity |
| CRA, evidence, findings, method, integrity, limits | Deadlines, attestations and accepted risks, the full list, what ran and what did not, how to verify |

Charts are inline SVG generated by `avesta_proof.viz`: no JavaScript required, no web fonts, no
network, identical on screen and in print. The only script in the page is a small progressive
enhancement (scroll-spy and a findings filter); with it disabled every row stays visible.

### Control status
- **pass**: every tool that gives evidence ran and found nothing.
- **partial**: nothing found, but some of those tools did not run.
- **warn**: only low-severity findings.
- **fail**: a medium or worse finding, or manual evidence has expired.
- **not-assessed**: no evidence yet (tool not run, or no attestation).

Control kinds: *auto* (scanner evidence), *manual* (needs an `[[attestation]]`),
*derived* (computed from `[product]`, for example the CRA support period).

### Score
Starts at 100. Each severity subtracts points, capped per severity so 500 low findings cannot
outweigh one critical (critical 25, high 10, medium 3, low 1). Accepted risks do not count.
Grades: A 90+, B 80+, C 65+, D 50+, F below.

## Configuration: `avesta-proof.toml`

See `examples/avesta-proof.toml`. Sections: `[product]` (name, manufacturer, contact, support
end, CRA class), `[gate]` (`fail_on`, `new_only`), `[quality]` (complexity, file size,
duplication and comment limits), `[report]` (branding and content, below), `[[attestation]]`
(manual evidence with `valid_days`), `[[accept]]` (accepted risk with `expires`).
`.avestaproof-ignore` removes a finding completely; prefer `[[accept]]`, which keeps it visible.

### Branding the report: `[report]`

```toml
[report]
organisation = "Your Company B.V."
navy = "#0E1A33"          # page furniture
gold = "#C9A461"          # accents
top_findings = 20         # how many cards "Fix these first" shows
show_all_findings = false # shorter report for an executive audience
sections = ["summary", "risk", "compliance"]   # omit to render everything
```

Every value is type-checked and colours must be hex, because the config comes from the scanned
repository. The table is stored inside the report, so `render` reproduces the same look later;
`render --theme other.toml` overrides it without touching the evidence.

## Evidence you can trust

- **Fingerprint:** SHA-256 of the canonical report JSON.
- **Signature:** Ed25519 over the same bytes; the public key is embedded and can be pinned with `--pubkey`.
- **Ledger:** `ledger.jsonl` links each scan to the previous entry's hash and stores a snapshot in `history/`. Deleting or editing an old scan breaks the chain.
- Private keys are written with mode 0600. For production, keep them in a KMS or HSM (roadmap).

## CI

`examples/github-action.yml` shows a pull-request gate with weekly re-scans (new CVEs appear
without code changes), SARIF upload, and cached history for new-vs-fixed tracking.
This repo's own `.github/workflows/ci.yml` runs Avesta Proof on itself.

## Tests

```bash
pip install -e ".[dev,scanners,signing]" && pytest -q     # 113 tests, ~2s, 91% coverage
```

Parsers are tested on saved tool output; signing, ledger tampering, baselines, attestations,
KEV/EPSS (with a fake network) and the CLI are tested end to end. Charts are pure functions
tested on three numbers; the view model is tested against deliberately malformed documents; and
one test feeds a finding title full of HTML tags through the renderer to prove a hostile
repository cannot inject markup into the report.

## What is verified and what is not

Verified here: all unit tests, an end-to-end run with Bandit, Lizard and the built-in checks, and
signature/ledger tamper detection. **Not yet run against real tools in my test environment:**
Trivy, Gitleaks and cppcheck (not installed; their parsers are tested on sample output) and the
live CISA KEV and FIRST EPSS downloads (network was blocked; the code uses a cache and fails
safe). Run `avesta-proof doctor` and a scan on a real project to confirm on your machine.

## Known limits

- Control mappings are indicative and need review by a compliance expert against the current standard text.
- Automated tools miss design flaws and business-logic bugs; pair with threat modelling and pen tests.
- SBOM reads requirements*.txt, poetry.lock, Pipfile.lock, package-lock.json, Cargo.lock. Not yet: Maven, Gradle, Go, yarn, pnpm, NuGet.
- No hosted dashboard yet; `report.json` and the ledger are the data model for it.
- PDF export needs `weasyprint` and has not been verified here; printing `report.html` from a
  browser produces a clean A4 document with the navigation hidden.
- Duplication detection finds exact copies of six or more normalised lines. It does not find
  renamed-variable clones, which need a parser per language.

## Next steps

See `docs/ccqat-review.md` section 6 for the month-by-month plan to May 2027 and
`docs/pitch.md` for the commercial story.
# avesta_proof-
