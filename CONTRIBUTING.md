# Get your change into Knots

[`TUTORIAL.md`](TUTORIAL.md) shows how to build *your own* Knots. This shows how
to get a change into *everyone's* Knots: the official releases.

Knots is not merged, it is assembled. There is no "merge to main" that ships a
Knots release. Each release is rebuilt by replaying a spec: a base tag plus an
ordered list of branches and PRs to merge on top (see [`README.md`](README.md)
for the spec model). Every release so far has used a Bitcoin Core release tag as
that base, but nothing in the process requires it: a future release could just as
well base on a previous Knots release. Either way, "contributing to Knots" means
getting an entry for
your change added to that spec. Your job is to produce a branch [the
assembler](https://github.com/bitcoinknots/assemble-deriv) can replay cleanly;
the maintainer's job is to add the line.

## Step 0: where to open the PR

Open it against [`bitcoinknots/bitcoin`](https://github.com/bitcoinknots/bitcoin).
That is the whole answer, GUI changes
included: Knots is a single tree with `src/qt` in it, there is no separate Knots
GUI repo, and GUI fixes land the same way as anything else (for example
`k297  qt_sweepprivkeys-29`). Your PR becomes a `k<n>` entry in the spec.

Do not rely on Core to carry your change in. A PR to
[`bitcoin/bitcoin`](https://github.com/bitcoin/bitcoin) or
[`bitcoin-core/gui`](https://github.com/bitcoin-core/gui) reaches a Knots release
only if a maintainer separately pulls it into the spec, and Core is not where
Knots' direction is decided. If a fix is genuinely useful to Core on its own
merits you can send it upstream as well, but to actually land it in Knots, open it
against the Knots repo.

The current spec carries hundreds of `<n>` (Core) and `g<n>` (Core GUI) entries
merged on top of the Core base. That is inherited Core content, not where you send
a Knots change; see [`README.md`](README.md) for how the prefixes map.

## Step 1: open the PR

Standard PR hygiene applies, and `bitcoinknots/bitcoin` ships its own
[`CONTRIBUTING.md`](https://github.com/bitcoinknots/bitcoin/blob/29.x-knots/CONTRIBUTING.md)
at the repo root: one logical change, tests for what you change, a clear
rationale. Sign your commits.

Open the PR against the current Knots dev branch, [`29.x-knots`](https://github.com/bitcoinknots/bitcoin/tree/29.x-knots), but base the branch itself on the oldest commit your change applies to: the commit that introduced the bug for a fix, or the oldest release that has the code you touch. The assembler merges your branch into each release, so an old base merges cleanly into every newer one without dragging in unreleased dev commits.

## Step 2: make the branch the assembler can replay

These rules are enforced by the assembler, which refuses a branch that breaks
them. They are not style preferences.

- **Rebase, never merge.** Do not merge `master`, `29.x-knots`, or any upstream
  branch *into* your feature branch. The assembler marks the first base-relative
  upstream commit as "poison" and hard-rejects any branch that contains it
  (`Branch <x> is poisoned`). Keep your branch a clean stack of your own commits
  on top of the base; if you do need to move it, `git rebase`, do not merge.
  Staleness alone is not a reason to move: the assembler only requires that your
  branch still merges cleanly, so a branch pinned well behind the base is fine
  as-is.
- **Merge cleanly.** When a branch conflicts, the maintainer has to resolve it in a separate merge branch and keep that branch in step with yours. A branch that applies clean is far more likely to be taken and costs no one maintenance. If you conflict with the base, rebase onto it first.
- **Keep the branch stable.** Once your PR is in the spec it is pinned with
  `last=<sha>`. If you force-push, that pin goes stale and the assembler stops
  with `<pr> needs updates from upstream`, and
[`check-pr-updates.pl`](check-pr-updates.pl) flags your
  PR as drifted. Land review changes *before* it is picked up; after that, treat
  the branch as append-mostly and tell the maintainer when it must move.

## Step 3: how it lands

When your change is accepted it becomes one line in
[`knots-next-29.spec`](knots-next-29.spec), the
in-progress spec for the next 29.x release. Here is a real one:

```text
	k319  privkeyio/fix-bip110-envelope-datacarrier	2b742c24b31	last=5e3153edef4
```

Reading left to right:

- `k319` is Knots PR #319.
- `privkeyio/fix-bip110-envelope-datacarrier` is the author's branch that gets merged.
- `2b742c24b31` is the merge commit the last assembly produced.
- `last=5e3153edef4` is the PR head the maintainer last reviewed. Each build compares it against the PR head and stops if your branch has moved, so new commits are not picked up until someone reviews them.

A Core PR entry looks the same without the `k` prefix.

If the maintainer has to rebase your branch, the line names the maintainer's copy and puts your branch after `last=`, so the build still watches your branch for updates:

```text
	k320  miniscript_guard_tapscript_opif-29	70d4ff8cea1	last=74a8dd54ae8 privkeyio/rdts-guard-opif-tapscript
```

You do not add this line; the maintainer does, when assembling the next release.
What gets you there is review. Tested ACKs are what move a PR from "open" to "in
the spec." The Knots reviewer pool is small, so a clean-merging, well-tested PR
with a clear rationale gets picked up; a messy one waits. Live triage status
(ready / NACK / high-priority / ACK level) is at <https://bosun.privkey.io>.

## Step 4: after it lands

Landing is not the end. When Core or the Knots dev branch moves under your
change:

- `check-pr-updates.pl` compares every PR referenced in the spec against its
  GitHub `updated_at` and reports the ones that have moved since the last
  assembly. That is the drift signal.
- If your branch stops merging cleanly, rebase it onto the oldest commit it now applies to and tell the maintainer the new `last=` sha. Re-homing a fix onto a new release base is exactly what the [`daggy`](https://wiki.monotone.ca/DaggyFixes/) tooling automates, but the source of truth is still your rebased branch.
- If your change is a bug fix, add a regression test in the same PR. It is what
  lets the fix be bisected to its introducing commit and re-homed safely later.

## The short version

1. Open the PR against `bitcoinknots/bitcoin`, GUI included. Core is an optional
   courtesy, never the path into Knots.
2. Base the branch on the oldest commit it applies to and keep it a clean, rebased stack. Never merge upstream into it.
3. Make it merge cleanly and stay pinnable.
4. Earn tested ACKs. The maintainer adds the spec line; you keep the branch alive.

New to the assembler? Build a one-line spec of your own first with
[`TUTORIAL.md`](TUTORIAL.md). Watching it replay a branch makes these rules
concrete.
