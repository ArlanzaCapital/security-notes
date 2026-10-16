# Remediating a vulnerable dependency in a development toolchain

**English** | [Español](braces-dependencia-vulnerable-herramientas-dev.md)

## Case status

The remediation has been **merged into the main branch**. Before the merge, it:

- passed validation on Windows and Linux;
- underwent an independent review, with documented reservations;
- passed the full CI pipeline.

The main branch CI also passed after the merge.

This article describes neither an original discovery nor a fix to the upstream package. The vulnerable dependency was **removed from the dependency tree**, not repaired.

## Known vulnerability

[GHSA-vfj7-8cjw-p6xm / CVE-2026-93687](https://github.com/advisories/GHSA-vfj7-8cjw-p6xm) documents stack exhaustion in `braces` through deeply nested brace patterns. It was published on September 18, 2026, and reviewed on October 2. When checked on October 17, 2026, the advisory still listed versions through 3.0.3 as affected and listed no patched version.

## How the problem surfaced

A dependency audit blocked CI because of this development toolchain:

```text
eslint-config-next → @next/eslint-plugin-next → fast-glob → micromatch → braces
```

That block was the expected behavior of the security policy. The dependency's presence alone did not demonstrate a remotely exploitable path in the application.

## Remediation strategy

The Next plugin's specific use of `fast-glob` was replaced locally with an adapter built on `tinyglobby`. The adapter applies only to that plugin dependency and is pinned to the plugin version that was validated. The vulnerable chain disappears from the installed dependency tree, while the CI security control remains in place.

The replacement has a limited scope: it does not implement the entire `fast-glob` API or syntax. Its contract supports an explicit subset, rejects unsupported inputs, and bounds pattern length, nesting depth, and expansion work.

## Evidence

- **Functional equivalence:** results were compared with the original implementation on representative cases, and the behavior of the lint rule that uses the adapter was checked.
- **Bounds:** limits and invalid patterns were tested; cases involving excessive work run in isolated processes with time limits.
- **Installation:** reproducible, with adapter resolution verified on Windows and Linux.
- **Quality:** lint, type checking, unit tests, and integration tests passed in both the proposed change's CI and the main branch CI after the merge.
- **Audit and build:** the dependency audit reported no advisories, and the Linux build passed. The build was validated locally because the workflow does not run a build.
- **Independent review:** concluded that the change was ready to merge, with reservations that did not require code changes: the limited contract is explicitly accepted, and the adapter remains subject to maintenance oversight.

These checks support the tested use. They demonstrate neither universal equivalence nor the absence of other defects.

## A related finding: secret scanning

While reviewing CI after the merge, we observed that the secret scanner had reported “0 commits” scanned, even though the control passed. The scope problem and its correction are explained in [Gitleaks passed, zero commits scanned](gitleaks-zero-commits-false-green.en.md). That article distinguishes our experience from a related public report about coverage of merge commits.

## Maintenance and lessons

The adapter is custom code that needs maintenance, with an assigned owner. Whenever the plugin, glob engine, or Node is updated, three things need to be done:

- check whether the adapter is still being used;
- check the resulting dependency chain;
- repeat the relevant tests.

The outcome must not be taken for granted.

The adapter should be removed when a compatible official alternative becomes available. Checks that still protect behavior should be retained; only those that test the adapter itself should be removed.

The main lesson is that replacing a dependency requires checking behavior, installation, provenance, and future maintenance costs. A clean audit result is not enough on its own, nor is a passing security control unless we verify what it actually examined.
