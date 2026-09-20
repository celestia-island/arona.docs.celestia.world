# Contributing to arona.docs.celestia.world

Thank you for your interest in contributing. This file is the only place where this
repository's contribution policy is defined; it is worded identically across the
organization apart from the repository name and licence class.

## Contribution policy — read this first

- **High merge bar, not a public roadmap.** Opening a pull request does not imply it
  will be merged. We accept a deliberately small number of changes, and only when
  they fit the architecture and pass review. This is by design, not rudeness.
- **What we welcome:** bug reports, focused fixes, well-scoped improvements to the
  **periphery** (adapters, device profiles, tooling, documentation, translations,
  tests), and a design discussion in an issue or PR comment *before* large code.
- **What we generally will not merge:** large unsolicited rewrites, architectural
  changes without a prior design discussion, bulk machine-generated pull requests,
  anything that lowers a safety or security bar, and changes to security-critical
  surfaces without an explicit invitation and extended review.
- **Core vs. periphery.** Security-critical surfaces — authentication and
  authorization, transport, hardware command paths, billing ledgers, safety gates —
  are held to the strictest bar and are maintained by the core team. Periphery is
  where external contributions are most useful and most likely to be accepted.
- **Design documents and changelogs do not belong in the source tree.** The pull
  request description *is* the design record and the merged history is the changelog.
- **A payment does not buy a tier.** Paying for work (a bounty, a sponsored task, a
  commissioned deployment) buys a deliverable and its evidence — never merge
  authority, review priority, or exclusivity.

## Contribution tiers

| Tier | Surface | Merge authority |
|---|---|---|
| 1 | documentation, translations, tests, CI hygiene | maintainers, low friction |
| 2 | periphery: adapters, profiles, tooling, plugin packages | maintainers after review |
| 3 | product logic | core team only; prior design discussion required |
| 4 | security-critical surfaces (auth/RBAC, transport, hardware command path, billing, safety) | core team only, explicit invitation, extended review |

## CLA

Every accepted contribution requires a signed Contributor License Agreement — see
[`CLA.md`](CLA.md). Signing is a **one-time action per contributor**
(and per employer, when contributing on behalf of a company). Add a
`Signed-off-by: Your Name <you@example.com>` line to each commit (`git commit -s`);
the sign-off is your electronic acceptance of the CLA.

## Security

Do **not** open public issues for security vulnerabilities. Report them privately
through GitHub Security Advisories — see [`SECURITY.md`](SECURITY.md).

## Code of conduct

Be respectful, constructive and inclusive. We follow the
[Contributor Covenant](https://www.contributor-covenant.org/version/2/1/code_of_conduct/);
see [`CODE_OF_CONDUCT.md`](CODE_OF_CONDUCT.md).

## Pull request process

1. Branch from the default branch (`master`; a few repositories use `main`).
   The `dev` branch model is retired — do not open pull requests against it.
2. Discuss large or behaviour-affecting changes in an issue first.
3. Keep commits atomic; the message convention is enforced by
   `celestia-devtools commit-msg-lint`.
4. Make the repository's own gates pass (format, lint, tests, version checks).
5. Sign off every commit (see above).
6. Rebase rather than merge. On your own feature branch a `--force-with-lease` is
   acceptable; a plain force push is not.

## Licence

arona.docs.celestia.world is licensed under the **Business Source License 1.1** (see [`LICENSE`](LICENSE)). The Change Date is **2030-01-01**: from that
date the Licensed Work converts to the **Synthetic Source License 1.0 (SySL-1.0)**
for all users and all uses, with no production-use restriction and no
separate commercial licence required.

For internal operations the licence is already permissive today: the Additional Use
Grant permits production use by any organization regardless of size or revenue
(internal operations of companies, government agencies, public institutions,
education and non-profit organizations), plus modification and contribution
upstream. What requires a separate commercial licence until the Change Date is
offering the Licensed Work to third parties as a hosted, managed or cloud service,
reselling or rebranding it as a product, or making it the substantial source of
value in a commercial offering to third parties.


## AI-generated contributions

This organization develops largely with AI agents, and its licences are written for
that. Disclosure is a licence obligation, not a courtesy:

- **SySL-1.0 repositories** — Section 2.3 requires every contribution distributed as
  source code to state, at the file or commit level, whether it was AI-generated,
  human-written, or a combination, and which models were used.
- **BUSL-1.1 repositories** — the same disclosure is required by the project's
  contribution policy, and modelled on the SySL pattern.

State it in the commit message, and in a file header when the file is new. Never
strip the existing disclosure notice from the root README.
- **Contributor direction.** For AI-generated portions you confirm you directed,
  selected and refined the output; where a jurisdiction does not recognise copyright
  in AI output, the CLA grants operate as contractual covenants.

## Legal

Operator information, the terms of service, the privacy policy and the filing
records for the hosted services live at https://celestia.world/legal/terms. A filing number is deliberately
not duplicated into the source tree.

---
*Canonical file maintained in the organization metadata repository; changes apply to
every repository that adopts it.*
