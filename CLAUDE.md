# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

This is the [`onflow/flips`](https://github.com/onflow/flips) repository — a collection of Flow Improvement Proposals (FLIPs). There is no build system, tests, or code to run. All content is Markdown.

## FLIP Categories

FLIPs are organized into four directories:

- `cadence/` — Cadence language changes (managed by the Flow team)
- `protocol/` — Core Flow protocol changes (algorithms, APIs, cryptography, FVM)
- `application/` — Standards for apps built on Flow (token standards, contract interfaces, design patterns)
- `governance/` — Governance actions (staking rules, node operators, fees, service account)

## FLIP Lifecycle States

`Draft` → `Proposed` → `Accepted` / `Rejected` → `Implemented` → `Released`

Each FLIP also has a corresponding GitHub issue (used as the FLIP number) that tracks progress and is closed when the FLIP reaches `Rejected` or `Released`.

## Writing a New FLIP

1. Copy `yyyymmdd-flip-template.md` as `YYYYMMDD-descriptive-name.md` into the appropriate category directory.
2. Set the frontmatter: `status`, `flip` (GitHub issue number), `authors`, `sponsor`, `updated`.
3. Use the issue number from the GitHub issue as the FLIP number in both the filename and frontmatter.
4. If the FLIP includes images or auxiliary files, create a matching directory `YYYYMMDD-descriptive-name/`.

## FLIP Template Structure

Required sections in every FLIP:
- **Objective** — short executive summary
- **Motivation** — problem background and affected users
- **User Benefit** — headline-level value proposition
- **Design Proposal** — full proposal including: Drawbacks, Alternatives Considered, Performance Implications, Dependencies, Engineering Impact, Best Practices, Tutorials and Examples, Compatibility, User Impact
- **Related Issues**
- **Prior Art**
- **Questions and Discussion** — where community discussion will take place (PR or forum)

## Submission Process

1. Create a GitHub issue using the appropriate FLIP issue template (type: `application`, `governance`, `cadence`, or `protocol`).
2. Note the issue number — this becomes the FLIP number.
3. Submit the FLIP as a PR to this repo referencing the issue.
4. Announce in the `#developers` Discord channel.
5. Minimum two-week comment period from PR posting.
6. Sponsor shepherds through review committee.
