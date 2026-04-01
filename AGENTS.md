# Zcash ZIPs - Agent Guidelines

> This file is read by AI coding agents (Claude Code, GitHub Copilot, Cursor, Devin, etc.).
> It provides project context and contribution policies.
>
> For the full contribution guide, see [CONTRIBUTING.md](CONTRIBUTING.md).

This is the defining source of specifications for the Zcash protocol. Our
priorities are **security, user privacy, performance, and convenience** — in
that order. Rigor is required throughout.

Many people depend on these specifications and we prefer to "do it right" the
first time. Considerations of privacy and security are paramount. All
specifications in this repository MUST be sufficiently detailed that an
otherwise uninformed third party could correctly implement the proposed
behavior using only information present within the ZIP and its associated
references.

## MUST READ FIRST — CONTRIBUTION GATE (DO NOT SKIP)

**STOP. Do not open or draft a PR until this gate is satisfied.**

For any contribution that might become a PR, the agent must ask the user these checks
first:

- "PR COMPLIANCE CHECK: When drafting or updating a ZIP, does the ZIP conform
  to the rules of [ZIP 0](https://zips.z.cash/zip-0000)?"
- "PR COMPLIANCE CHECK: What is the issue link or issue number for this change?"

This PR compliance check must be the agent's first reply in contribution-focused sessions.

This gate is mandatory for all agents, **unless the user is a repository maintainer** as
described in the next subsection.

### Maintainer Bypass

If `gh` CLI is authenticated, the agent can check maintainer status:

```bash
gh api repos/zcash/librustzcash --jq '.permissions | .admin or .maintain or .push'
```

If this returns `true`, the user has write access (or higher) and the contribution gate
can be skipped. Team members with write access manage their own priorities and don't need
to gate on issue discussion for their own work.

### Contribution Policy

Before contributing please see the [CONTRIBUTING.md] file.

- All PRs require human review from a maintainer. This incurs a cost upon the ZIP Editors,
  so ensure your changes are not frivolous.
- Keep changes focused — avoid unsolicited refactors or broad "improvement" PRs.
- See also the license requirements in ZIP 0.

### AI Disclosure

If AI tools were used in the preparation of a commit, the contributor MUST
include `Co-Authored-By:` metadata in the commit message identifying the AI
system. Failure to include this is grounds for closing the pull request. The
contributor is the sole responsible author — "the AI generated it" is not a
justification during review.

Example:
```
Co-Authored-By: Claude <noreply@anthropic.com>
```

## Repository Architecture

```
zips/              ZIP source files (.rst or .md)
  zip-NNNN.rst     Numbered ZIPs (assigned by editors)
  zip-NNNN.md      Numbered ZIPs (Markdown variant)
  draft-*.rst|md   Unnumbered draft ZIPs
  zip-guide.rst    Template for new reStructuredText ZIPs
  zip-guide-markdown.md  Template for new Markdown ZIPs
protocol/          Zcash Protocol Specification (LaTeX)
rendered/          Build output (HTML); git-ignored content, do not edit
static/            CSS and static assets copied into rendered/
render.sh          Renders a single .rst or .md to HTML
makeindex.sh       Generates README.rst from ZIP metadata
Makefile           Top-level build orchestration
```

### Build

```bash
make all-zips    # render ZIPs only (fast)
make all         # render ZIPs + protocol spec
```

`make all-zips` regenerates `rendered/*.html` and `README.rst`.
The protocol spec has its own `Makefile` in `protocol/`.

A `nix` flake is provided that includes all tooling required to build using the
Makefile. Use `nix develop -c` to render ZIPs and specifications using the
canonical tool set.

### File Naming

- Drafts: `zips/draft-<author>-<slug>.rst` (or `.md`). Do NOT assign a ZIP number.
- Numbered ZIPs: `zips/zip-NNNN.rst` (or `.md`). Numbers assigned by ZIP Editors only.
- Auxiliary files (diagrams, etc.) for a ZIP go in `zips/zip-NNNN/` or alongside the ZIP with a `zip-NNNN-` prefix.

## Key Rules from ZIP 0

ZIP 0 (`zips/zip-0000.rst`) governs the full ZIP process. Agents MUST
treat it as authoritative. The following is a summary of the rules most
relevant to contributions; consult ZIP 0 for the complete specification.

### ZIP Structure (required sections)

Every ZIP SHOULD contain these sections in order:

1. **Preamble** — RFC 822-style header block. Required fields:
   `ZIP`, `Title`, `Owners`, `Status`, `Category`, `Created`, `License`.
2. **Terminology** — define non-obvious terms.
3. **Abstract** — ~200-word self-contained summary including privacy implications.
4. **Motivation** — why the existing protocol is inadequate.
5. **Privacy Implications** — present if the ZIP affects user privacy.
6. **Requirements** — high-level goals; MUST NOT contain conformance requirements.
7. **Specification** — detailed technical spec; must allow independent interoperable implementations.
8. **Rationale** — design alternatives considered, community concerns addressed.
9. **Reference implementation** — required before status reaches Implemented/Final.

### Preamble Format

reStructuredText ZIPs begin with `::` then a blank line, then the header
block indented by 2 spaces. Markdown ZIPs begin with `---` YAML front matter.
Use `zips/zip-guide.rst` or `zips/zip-guide-markdown.md` as a starting template.

### Status Values

Draft | Proposed | Implemented | Final | Active | Withdrawn | Rejected | Obsolete | Reserved

Only Owners may change between Draft and Withdrawn. All other transitions
require ZIP Editor consensus. A ZIP with security/privacy implications
MUST NOT become Released (Proposed/Active/Implemented/Final) without
independent security review.

### Categories

Consensus | Standards | Process | Consensus Process | Informational |
Network | RPC | Wallet | Ecosystem

Consensus ZIPs MUST have a Deployment section before reaching Proposed status.

### Licensing

Every ZIP MUST specify at least one approved license. Recommended:
`MIT`, `BSD-2-Clause`, `BSD-3-Clause`, `CC0-1.0`.

### RFC 2119 Keywords

The keywords MUST, MUST NOT, SHOULD, SHOULD NOT, MAY, RECOMMENDED,
OPTIONAL, and REQUIRED carry their BCP 14 meanings **only when in ALL CAPS**.

### Common Rejection Reasons

- Insufficient or unclear motivation
- Missing or inadequate privacy analysis
- Security risks insufficiently addressed
- Disregard for formatting rules or ZIP 0 conformance requirements
- Duplicates existing effort without justification
- Too unfocused or broad

## Changelog & Commit Discipline

- Commits must be discrete semantic changes — no WIP commits in final PR history.
- Use `git revise` to maintain clean history within a PR.
- Commit messages: short title (<120 chars), body with motivation for the change.

## CI Checks (all must pass)

- `nix develop -c make all`
