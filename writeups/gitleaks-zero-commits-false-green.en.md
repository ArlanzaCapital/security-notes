# Gitleaks passed, zero commits scanned: checking the scan's actual scope

**English** | [Español](gitleaks-zero-commits-false-green.md)

A secret-scanning control passed after changes were merged. On reviewing the log, the important number was not the finding count: the scanner had reported zero commits processed.

The result did not demonstrate a leak. In this case, a control had been accepted as successful without establishing that it had processed the expected commits. Git's scope and the scanner's counter needed to be checked separately.

The correction required defining the scope, including merge changes, and rejecting empty or incomplete results. This article explains the integration problem and the checks adopted. It presents neither an original discovery nor a claim about every version of Gitleaks or its GitHub action.

## Downloaded history is not scanned history

Gitleaks can scan patches from Git history. The `--log-opts` options determine what it receives from Git. Having every commit available does not require the scanner to process them. This distinction is documented in [Gitleaks' Git mode](https://github.com/gitleaks/gitleaks#git).

The checkout also needs checking: `actions/checkout` can fetch the full history with `fetch-depth: 0`. The required objects must be available and their references must resolve; the subsequent selection remains the scan's responsibility. [Checkout documentation](https://github.com/actions/checkout#fetch-all-history-for-all-tags-and-branches).

A step named “scan all history” may end up running a limited range selected by its adapter. The step's name does not establish its scope.

## Two different decisions: traversing commits and showing their changes

Git options can restrict history or change the patches produced:

- `--first-parent` follows only the first parent when traversing a merge.
- `--no-merges` excludes merge commits.
- `--diff-merges=first-parent` shows a merge's changes relative to its first parent. This is not the same as restricting traversal with `--first-parent`.

Also, `git log -p` does not show merge patches by default unless options enable that behavior. Enumerating merge commits and obtaining their changes are separate problems. [Git log documentation](https://git-scm.com/docs/git-log).

This creates two risks: omitting commits from merged branches and omitting content introduced while resolving a merge. For certain topologies and ranges, filtering can even leave the scanner with no processable commits. This does not mean that every push of a merge produces an empty scan.

## The related public report that deserves credit

[Issue #236 in gitleaks/gitleaks-action](https://github.com/gitleaks/gitleaks-action/issues/236), opened by `sumanthratna`, already described a related false negative: first-parent traversal could leave out content brought in from another branch while the action reported a clean result.

The [action's code at public revision e0c47f4](https://github.com/gitleaks/gitleaks-action/blob/e0c47f4f8be36e29cdc102c57e68cb5cbf0e8d1e/src/gitleaks.js#L85-L131) shows the options applied to events and the forwarding of the scanner's exit code. This is a specific revision, not a claim about all later versions.

The case documented here had a zero counter. The issue focuses on commits omitted through the second parent: it is a related precedent, not an exact public reproduction of that result. This article's contribution is to explain scope verification and the guards applied, without claiming priority for the defect.

## The correction: an explicit scope for each event

The adapter was replaced with a direct Gitleaks invocation, downloaded from its official distribution, with a pinned version and checksum verification of the downloaded archive. This check is not presented as an independent checksum calculation of the extracted binary on the runner. Two policies were adopted:

| Event | Selected scope |
|---|---|
| Pull request | Commits reachable from the PR HEAD that are not reachable from its base SHA |
| Push to the main branch or manual run | History reachable from the selected HEAD |

A `BASE..HEAD` range expresses that reachability difference; it is not simply a date subtraction or a diff between two trees. The selection can be checked separately with `git rev-list`. History reachable from HEAD is also not the same as every branch, tag, or object in a repository. [Git rev-list documentation](https://git-scm.com/docs/git-rev-list).

The following snippets illustrate scope selection only, not the complete acceptance control. They assume verified SHA references, available objects, and a previously verified installation. The syntax corresponds to Gitleaks 8.24.3, the version examined for this integration.

```bash
# Pull request: BASE and HEAD are the event's verified SHAs.
range="${BASE}..${HEAD}"
git rev-list --count "$range"
gitleaks git --redact --log-opts="--diff-merges=first-parent ${range}" .
```

```bash
# Full-history policy for the selected HEAD.
gitleaks git --redact --log-opts="--full-history --diff-merges=first-parent HEAD" .
```

Choosing a diff relative to the first parent lets us observe what the merge introduced without excluding its other branches from traversal. It can produce repeated findings across commits and merges; it should not be confused with a deduplication mechanism. [Merge diff formatting](https://git-scm.com/docs/git-log#_diff_formatting).

Invocation syntax and options for reporting, redaction, and exit codes are in the [Gitleaks documentation](https://github.com/gitleaks/gitleaks#usage). Event and scope choices are policies of this integration, not universal product guarantees.

Printing the count does not block an empty range. The workflow must check it and apply the guards below; copying only these snippets does not fully fix the false green.

## Zero findings and a valid scan require separate checks

The corrected step rejects:

1. A range with no commits enumerated by Git.
2. Scanner findings.
3. A process error code, including a partial run reported as an error.
4. A missing or zero processed-commit counter.
5. An empty log, a missing or empty SARIF report, or a detected failure while capturing evidence.

The status of Gitleaks and that of the process saving its output are preserved separately. A logging pipeline must not turn a scanner failure into apparent success.

File existence and size are basic guards: on their own, they prove neither report validity nor scope coverage. Validation also inspected the SARIF and its correspondence with the run.

The log counter is interpreted with a parser tied to the pinned version. If the format changes and can no longer be read, the control fails: it does not replace an unknown value with a successful result.

Rejecting zero commits is a conservative policy. It can block legitimate ranges without processable text; those cases need an explicit decision. It is not a universal Gitleaks rule, nor does it turn a positive counter into proof of complete coverage.

## The two counters do not have to match

`git rev-list --count` enumerates commit objects. The scanner's counter comes from its diff processing.

In [Gitleaks 8.24.3's DetectGit code](https://github.com/gitleaks/gitleaks/blob/v8.24.3/detect/git.go#L18-L80), binary files and deleted files are discarded before the commit is added to the counter. Deleting lines from a file that still exists is not the same as deleting the entire file.

The two figures are therefore not required to be equal. Both are recorded to detect suspicious changes and aid diagnosis. Coverage is justified by the scope, options, and tests, not by numerical equality.

## How the change was validated

Earlier validation documented tests with synthetic repositories:

| Case | Documented result |
|---|---|
| History with a clean merge | Successful scan, with commits processed and a report containing no findings |
| Dummy marker in a detectable format introduced only while resolving a merge | Finding attributed to the merge, blocking the step |
| Commit that only deletes lines from a retained file | Valid run; this test does not represent complete file deletion |
| Empty range | Rejected before launching the scanner |

A run over the real history was also documented, with no findings in the selected scope. That result does not establish that there were never any secrets or that formats or references excluded by the policy are covered.

This text summarizes earlier tests and the corrected configuration: the tests were not repeated to write the article. The table's results come from that working documentation and are not presented as an independent public reproduction. The code snippets are generic. Evidence from the source project is not published here.

## What to retain after each run

Useful evidence answers concrete questions:

- What event triggered the scan, and which references were selected?
- What version, options, configuration, and exclusions were used?
- How many commits did Git enumerate, and how many did the scanner report?
- How did the scanner and output capture finish?
- Which log and report correspond to that run?

A scope summary, redacted log, and SARIF were retained, including failed runs when material was available. These copies are reviewed before publication: redacting matches does not guarantee that paths, messages, or metadata contain no private information.

The acceptance criterion stopped being merely “the process exited with code zero.” It now required an identified scope, a nonempty run, a valid result, and available evidence. A passing job is useful only if we can explain what it examined to reach that result.
