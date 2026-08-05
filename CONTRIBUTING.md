# Contributing to DDC Specifications

Thank you for your interest in contributing to the Diamond Data Chain specifications.

---

## RFC Process

Changes to **Stable** specifications follow a Request for Comments (RFC) process:

1. **Open a Discussion** – Start a GitHub Discussion describing the proposed change, its motivation, and any alternatives considered.
2. **Draft an RFC** – Submit a pull request adding or modifying a specification document. Mark the PR as a draft until the RFC is ready for review.
3. **Review Period** – The RFC must remain open for a minimum of **14 days** to allow community feedback.
4. **Governance Vote** – Breaking changes (major version bump) require a governance vote as defined in [`specs/governance.md`](specs/governance.md). Non-breaking changes require approval from two specification maintainers.
5. **Merge** – Once approved, the RFC is merged and the specification version is updated accordingly.

## Non-Breaking Changes

Typo fixes, clarifications, and example improvements may be submitted as regular pull requests without an RFC. These still require review from one maintainer.

## Style Guide

- Write in clear, precise technical English.
- Use [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) keyword conventions (`MUST`, `SHOULD`, `MAY`, etc.) for normative requirements.
- Prefer concrete examples over abstract descriptions.
- Include diagrams (ASCII or Mermaid) where they aid understanding.
- All normative statements should be testable.

## Specification Template

New specification documents should follow the structure in [`specs/_template.md`](specs/_template.md).

## Code of Conduct

This project follows the [Contributor Covenant Code of Conduct](https://www.contributor-covenant.org/version/2/1/code_of_conduct/).
