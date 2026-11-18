# Comparing four attempts at the plume-works images repo

Written 2026-11-18. Four clones of `plume-works/images` sit under `.nested/`, each on its own
branch with an open PR. Each was built by a coding agent using a different skill set or execution
mode. This note records how they compare on code quality, slop, maintainability, bugs,
readability, YAGNI/DRY/KISS, and documentation a later session can work from.

| # | Clone | Branch | PR | Skills | Execution |
|---|---|---|---|---|---|
| 1 | `.nested/images` | `core-image` | #1 | superpowers | subagent-driven development |
| 2 | `.nested/images-try2` | `core-images-try2` | #2 | superpowers | subagent-driven development |
| 3 | `.nested/images-try3-no-ssd` | `core-images-try3-no-ssd` | #3 | superpowers | `executing-plans`: one session, one final review |
| 4 | `.nested/images-try4-grill` | `spec-4-images` | #18 | mattpocock (`grill-with-docs`, `to-spec`, `to-tickets`, `implement-spec`) | one subagent per ticket, each in a worktree |

## How this was checked

- All tracked code and docs in the four clones were read in full.
- shellcheck and hadolint were run on every shell script and Dockerfile.
- CI status was read from the four PRs: every check is green on all four.
- The saved session transcripts were analysed for process (human turns, corrections, what was
  actually verified, token use).
- The images were **not** built locally for this comparison. "Works" below means CI passed.

## The comparison is not fair, and where

The attempts did not start from the same place, so the headline ranking says less about the skill
sets than it appears to.

- **Attempts 2 and 3 started from attempt 1's plan.** Their branches share attempt 1's first
  design and plan commits, followed by 17 more planning commits that fixed the plan ("Remove
  invented rules from design", "Fix defects in core image plan", "Drop Node from core until
  something needs it", short ADRs). They executed a plan that had already been corrected once
  against a real run.
- **Attempt 4 started fresh.** It had no prior plan, did its own interview, and found its own
  mistakes for the first time.
- **Scope differs by a factor of four.** Attempts 1–3 build one `core` image. Attempt 4 builds
  four images, a Coder template and a lint workflow. More scope means more surface for excess and
  loose ends, whatever the method.
- **The requirements differ.** Attempt 4's core runs as root for CI job containers, pins every
  tool, and adds CMake, Ninja and Python. Those came from grilling answers, not from the method.
- **No planning session for attempts 1–3 was saved.** The superpowers brainstorm and planning
  cannot be compared with the grill session at all; only execution can.

What can be compared fairly:

- **Attempt 2 against attempt 3.** Same plan, same scope, different execution mode. This is the
  cleanest comparison here.
- **Attempt 1 against attempts 2 and 3.** Shows what a second planning pass bought.

What cannot: superpowers against grill as methods. A fair rerun would give both the same written
brief, the same scope (core only), a fresh start, and saved planning transcripts.

## Scorecard

| | 1 | 2 | 3 | 4 |
|---|---|---|---|---|
| Scope | core | core | core | 4 images, Coder template, lint CI |
| Lines added | 748 | 981 | 1,048 | 2,549 |
| Lint findings (excluding unpinned-apt and style notes) | 1 | 0 | 3 | 0 |
| Output tokens, execution | ~133k | ~120k | ~84k | ~665k (includes planning) |
| Subagents | 16 | 12 | 1 | 18 |
| Human corrections during execution | 1 | 0 | 1 | ~6 |
| YAGNI / KISS | fair | good | good | poor |
| In-repo docs for a later session | weak | good | good | mixed |

Lint detail: attempt 1 has one hadolint DL4006 (pipe without pipefail). Attempt 3 has one DL4006
and two SC2155. Attempts 2 and 4 have none. Every attempt leaves apt packages unpinned on purpose.

## Attempt 3: best fit for the stated criteria

- **Code.** `core/start-dockerd` is the cleanest of the three superpowers versions: it fails as
  soon as the daemon dies, stops the daemon on timeout, and appends to the log so a retry keeps
  the earlier one. Failing checks in `core/test.sh` print why they failed.
- **Renovate.** The config is 31 lines with no custom rules, because `uv` is installed from its
  GitHub release, which carries release dates.
- **Process.** All four tasks were implemented in about six minutes. One whole-branch review found
  every real defect. It had the lowest cost of the four.
- **Weak spots.**
  - The argument-rotation trick in `core/test.sh` (`check_run`, lines 45–60) is hard to read, and
    several checks are 200-character one-liners with embedded shell.
  - The Ubuntu digest is deliberately an old one, to exercise Renovate, and is still in the
    branch.
  - The `uv` tarball install began as a misreading of "let Renovate use GitHub releases for uv,
    but still have uv from its image". It was kept after the agent showed Renovate cannot mix the
    two sources.
  - The checksum it verifies comes from the same release, so it proves only that the download was
    intact.
  - The PR body still says amd64 was not verified, though it later was.

## Attempt 2: simplest code, one patched-on Renovate workaround

- **Renovate.** The `uv` handling in `.github/renovate.json` is a regex manager plus two extra
  package rules, 20 of the file's 51 lines. It is a patch made after the full run, not part of
  the plan: the agent reported at the end that GHCR gives no release dates, so `uv` would never
  update under the 3-day wait, and the human chose "read uv's version from GitHub releases" while
  keeping `uv` copied from its image. That combination forces two update PRs, because a GitHub
  release knows the version and not the image digest. Between them `main` shows the new version
  while still building the old one. The agent disclosed this. It should work, but it has not been
  exercised or reviewed. So this is a consequence of a product choice, not a defect of the method.
  Attempt 4 never met the problem: it sets no release wait and installs every tool from its
  release tarball.
- **`start-dockerd`.** It truncates the log on each start and leaves the daemon running on
  timeout, so a retry starts a second one. Reviewers saw both, and neither was shown to the human
  (see "Attempt 2 against attempt 3" below). The script has one commit and was never touched
  again.
- **Leftovers.** An empty `org.opencontainers.image.version` label; a commit-and-revert pair at the
  tip of the branch (a weekly digest schedule, requested and then withdrawn); the spec still lists
  `.dockerignore` at the root after it moved to `core/`.
- **Process waste.** The Task 1 reviewer had `VOLUME` removed as out of scope, and Task 2 added it
  back: a fix commit, a rebuild and a re-review for an identical end state.

## Attempt 1: weakest

- It ships Node 24 from NodeSource, which the later attempts dropped in favour of Bun.
- CI publishes `<sha>-amd64` and `<sha>-arm64` tags that are never cleaned up.
- The "fails unprivileged" test passes even if `start-dockerd` is missing.
- The final-review fix wave added "adopt a daemon that is already starting" logic, which created
  two new defects. The last fix to it has no test and no review.
- There are no decision records, and the spec is the earliest, least corrected version.

## Attempt 4: best engineering, most excess

- **What it gets right.**
  - It is the only attempt that handles the real nested-Docker problems: a check that the engine's
    storage is not on the overlay filesystem, with an error that says what to mount; cgroup v2
    setup; a lock against concurrent starts; a systemd path.
  - It is clean on both linters, and lints itself in CI.
  - The ADRs in `docs/adr/` give reasons and rejected options. ADR 0003 records two things a
    reader would otherwise "fix".
  - A subagent caught a smoke test that had been silently skipping every check after one command
    consumed its input.
- **Over-built.**
  - About 50 lines of tag-rollback bash sit inside `.github/workflows/images.yml`.
  - `.github/scripts/images.sh` uses a jq dependency sort with cycle detection to order four
    images.
  - A weekly Docker prune timer has a 30-line test.
  - `images/devcontainer-base/Dockerfile` is 13 lines; its smoke test is 139.
- **Brittle.**
  - Smoke tests match exact CLI output such as "No MCP servers configured" and an OpenCode log
    line. A wording change in a Renovate bump will fail CI.
  - The devcontainer smoke test extracts its config from the README with awk.
- **Not DRY.** The Coder start-up hand-off is described in at least five places: the README twice,
  ADR 0002, the `main.tf` header, and `agent-services.target`. The README is 360 lines.
- **Loose ends.**
  - Actions are on floating tags (`@v7`, `@v4`), including a third-party cleanup action with
    package write access, while the README says everything is pinned.
  - The Coder template pins an all-zero placeholder digest and was never run on a real Coder
    deployment. Both are stated in the README.
  - ADR 0002 cites a `prototype/startup-sequence` branch that was never pushed.
  - `docs/agents/` is generic skill boilerplate, with examples about `src/ordering` and
    `src/billing`.
  - A comment block in `images/smoke-lib.sh` has a broken sentence left by an edit.
  - PR #18 now conflicts with `main`, which gained a default `renovate.json`.
- **Spec durability.** The spec lives only in GitHub issue #4 and is already stale: it does not
  mention OpenCode 2, the declared volumes, `OPENCODE_PASSWORD`, or the ADR 0003 exceptions. The
  replacement `.devcontainer.json` exists only in issue #16. Tickets were left unticked until the
  human complained.

## Attempt 2 against attempt 3: what reached the human

This is the one fair comparison here (same plan, same scope), and the clearest difference between
the two execution modes is not in the code they first wrote. It is in which review findings the
human got to decide on.

Both runs wrote a `start-dockerd` with the same weaknesses: a daemon left running after a timeout,
and a log overwritten on each start. Both runs' reviewers noticed.

| | Attempt 2 (subagent-driven) | Attempt 3 (one session) |
|---|---|---|
| Who found it | Task 2 reviewer, as Minor | Final whole-branch reviewer, as Minor |
| What happened next | Written to the ledger. The final reviewer triaged the leftover daemon as "can stay" and put the log truncation under "Declined to judge". | The controller listed it under "Still deferred" in its report. |
| What the human saw | Nothing. The final report has 2 decisions and 10 rulings; neither issue is among them. The ledger was then deleted. | The deferred line, then a numbered open-items list with a recommendation for each. |
| Outcome | Never fixed. | Fixed test-first in `6071fc0` after "all recommended". |

Attempt 3's open-items round also produced the subordinate-ID rename, the self-update switch-off
and the `.dockerignore` move. Attempt 2 got the last of these through its fix wave and not the
others.

Why it happened: in the subagent-driven run a finding passes through three filters before the
human (task reviewer, ledger, final reviewer's triage), and each one can close it. The final
reviewer's reason for "can stay" was that the spec bars supervision logic, which is a judgement
the human would likely have made differently, as they did in attempts 1 and 3. In the one-session
run the agent that read the review is the one that reports to the human, so deferred items
survived.

Three cautions.

- **One run of each.** Attempt 1, also subagent-driven, did fix the same issues, because its final
  reviewer rated them Important.
- **Attempts 1 and 2 had different inputs, and the difference points at the spec.** Attempt 1 ran
  from the first, uncorrected plan. Attempt 2's polished spec added two passages that attempt 1's
  lacks: "Must not happen: the core gains an entrypoint, a supervisor, ..." and "Left to the
  builder: layer ordering and the internals of `start-dockerd`". Attempt 2's final reviewer cited
  exactly these to close the findings ("the spec bars supervision logic", "the spec leaves
  internals to the builder"). So the tighter spec gave the reviewer grounds to dismiss real
  defects as out of scope. Attempt 3 ran from the same polished spec, and its reviewer still
  raised them as Minor, so the spec alone does not explain it either.
  This is less "the second planning pass over-specified" than "scope statements were read as
  quality waivers". The second pass mostly removed things (invented rules, Node) and its two
  added passages are boundaries, not detail. "Left to the builder" was meant as freedom to choose
  an implementation, and the reviewer took it as exemption from review; "no supervisor" was
  stretched to cover stopping a daemon the script itself started. A spec that says "the builder
  chooses how, and it is still reviewed for correctness" would close that gap, as would a final
  report that lists every finding a reviewer dismissed.
- **The human asked.** In attempt 3 the deferred line was already on screen, but the full
  open-items list came from an explicit request to go through open items.

## Process findings

- **Subagent-driven development filtered findings away from the human.** Detail in the section
  above. Minor findings were decided by reviewers and the ledger was deleted, so the human could
  not overrule them.

- **Per-task reviewers earned little.** In attempts 1 and 2 they found one or two real issues
  each, were told not to rerun tests, and one caused the `VOLUME` round trip. The single
  whole-branch review found the important defects in all three superpowers runs.
- **The same design error appeared twice.** Attempts 1 and 4 both chose not to declare the Docker
  storage volume, and the human reversed it both times. In attempt 4 the orchestrator defended the
  choice with a false claim until told to search the web.
- **Grilling surfaced real decisions, at a price.** Systemd instead of a supervisor, secrets
  through `compgen -e`, and sysbox against privileged mode all came out of it. Four of the agent's
  proposals were rejected as bloat or poor design. The spec was written, re-grilled and rewritten;
  tickets were drafted twice.
- **Fix waves create defects.** Attempt 1's final fix wave introduced two new problems, and the
  "no second fix wave" rule then parked a three-line fix for nine hours.
- **Superpowers plans do not last.** They are execution scripts and are stale after the run:
  attempt 3's still says to copy `uv` from its image and attempt 1's still lists Node. The spec
  and the short ADRs are the durable part.
- **Claims of verification were mostly honest.** All four runs really built and tested locally.
  The weak spots were controllers reporting "diff checked" after reading only a subagent's
  summary (attempt 1) and an implementer writing its report before applying the fix (attempt 2).

## Not verified by anyone

- The publish path on `main` (manifest creation, `edge` tag) has never run in any attempt.
- No real Renovate PR has run against any of the configs; the 10-day rule for `x.y.0` releases is
  validated but not exercised.
- Attempt 4: the Coder template on real Coder, remote control after sign-in, the consent answer,
  fork PRs, and `pr-N` tag deletion.

## Recommendation

Use attempt 3 as the base for `core`. Bring over from attempt 4 the pieces that earn their place:
the ADR style, the lint workflow, and the overlay-storage check in `start-docker`. Treat the rest
of attempt 4 as a reference for the later images, not something to merge as it stands.

Given the fairness limits above, read this as "attempt 3 is the best result", not "superpowers
without subagents is the best method".
