# Reproduce Knots, then build your own

Bitcoin Knots is not a hand-maintained fork. Each release is rebuilt from three public inputs:

1. a base Bitcoin Core release tag,
2. a spec file: the list of PR branches to merge, in order,
3. the branches themselves.

A script called `assemble-knots.pl` reads the spec, checks out the base, and merges each branch in turn. Because it is deterministic, anyone can rebuild a release and confirm it matches what Luke published, and anyone can write their own spec to build a custom Knots.

This guide does both, using Knots 29.4.3. Every command below was run for real. For a later release, swap in its tag.

## Part 1: Reproduce an official release

This confirms your machine and the public sources produce the exact release commit Luke shipped.

```bash
git clone --recurse-submodules -b sub-knots https://github.com/privkeyio/knots-assembly
cd knots-assembly
./.ci/run-all.sh v29.4.3.knots20260508
```

The tag is all you need. The script finds the matching spec, fetches every branch, runs the assembler, and compares the result to the release tag. If it matches, that release is reproducible from public sources. Branches that moved after the release do not matter, because the script resets each one to the commit the release merged. A mismatch means a commit the spec needs is no longer public, or your git setup differs (see below).

You need `bash`, `git` and `perl`, plus the standard tools and disk space listed under "System requirements" in [`README.md`](README.md). The assembler is plain Perl with no extra modules to install.

The assembler reads git's diff output, so it fails on conflicts if your git config sets `diff.mnemonicPrefix` or `diff.noprefix`. Check with `git config --get diff.mnemonicPrefix` and `git config --get diff.noprefix`, and turn either off for these runs, for example by prefixing commands with `GIT_CONFIG_COUNT=1 GIT_CONFIG_KEY_0=diff.mnemonicPrefix GIT_CONFIG_VALUE_0=false` (git 2.31 or newer).

## Part 2: Build your own sub-Knots

A sub-Knots is a derivative of Knots, built the same way Knots is built from Core. Instead of checking out Core and rebuilding all of Knots, you check out a finished Knots release and merge only the changes you want on top. The recipe stays short and the build is fast.

### Simplest case: one branch

Here is Knots 29.4.3 plus one open Knots PR, a fix for MuHash3072 dividing by itself:

```
checkout v29.4.3.knots20260508
timestamp 2026-05-07 03:45:44
	k453  2rdzy/fix-muhash-self-division					last=5d5eb51cb5c
```

Save it as `my.spec` and assemble it (see "Assembling by hand" below). The result is a clean merge: the PR's one commit plus the merge commit.

Each line after the header is one merge:

- `k453` is the PR number, padded to six characters. `k` marks a Knots PR, a bare number is a Core PR, and `n/a` means there is no PR.
- `2rdzy/fix-muhash-self-division` is the author's branch, `<owner>/<branch>` on that owner's `bitcoin` fork.
- `last=5d5eb51cb5c` is the branch head you last reviewed.

The `timestamp` line sets the commit dates. Copy it from the release's own spec.

### Stack several branches

A spec is an ordered list of merges, so add one line per branch:

```
checkout v29.4.3.knots20260508
timestamp 2026-05-07 03:45:44
	k453  2rdzy/fix-muhash-self-division					last=5d5eb51cb5c
	k452  2rdzy/port-core-35550-sendcmpct-announce			last=c080c0e25f2
```

Both merges come out clean: each PR's one commit plus its merge commit. The second PR makes nodes reject a `sendcmpct` announce value greater than 1.

Two things to know:

- Any branch with a `/` in its name needs `last=<sha>`. Leave it off and the assembler stops with "Please provide last= for direct remote merge of ...". On every build it compares that sha to the PR head and stops with "needs updates from upstream" if the PR moved, so nothing new slips in unreviewed. Review the new commits, then update the sha. An `n/a` line has no PR head to check, so it must name the branch to watch after the sha, `last=<sha> <owner>/<branch>`, or the assembler stops with "Latest upstream is undefined!".
- A branch merges cleanly into a release when it is based on a commit the release already contains. Both PRs here are, so each brings only its own commit. A branch based on a newer dev branch also brings the dev commits between the release and its base, and if it is far ahead you will get conflicts. Use a branch based on an older commit, or resolve the conflict in your own branch as shown below.

### Assembling by hand

`run-all.sh` compares against a release tag, and a custom spec has none, so run the assembler directly. This assumes `knots-assembly` from Part 1 and `my.spec` sit side by side:

```bash
# 1. a bitcoin repo with the base tag and your branches as remotes
git clone https://github.com/bitcoinknots/bitcoin.git bitcoin && cd bitcoin
git remote add 2rdzy https://github.com/2rdzy/bitcoin.git
git remote add origin-pull-k https://github.com/bitcoinknots/bitcoin.git
git fetch 2rdzy
git fetch origin-pull-k '+refs/pull/*/head:refs/remotes/origin-pull-k/*/head'
git fetch https://github.com/bitcoin/bitcoin.git master
git branch -f master FETCH_HEAD   # the assembler's poison check needs a local master (Core's)

# 2. work from the base tag and run the assembler on your spec
git worktree add --detach ../build v29.4.3.knots20260508
cd ../build
perl ../knots-assembly/assemble-knots/assemble-knots.pl ../my.spec
```

Add one remote per branch owner in your spec. `perl ../knots-assembly/extract-remotes.pl < ../my.spec` prints the remotes for the branch column. It leaves out two kinds of ref the update check needs, so fetch those yourself: the PR heads (the `origin-pull-k` fetch above, or `origin-pull` from `bitcoin/bitcoin` for Core PRs), and any branch named after `last=`.

### When a merge conflicts

If a branch conflicts with the release or with an earlier line in your spec, resolve it once in a merge branch of your own and put that branch in the spec. The author's branch stays as it is. This is how the official specs handle conflicts.

Start from what it conflicts with, merge the author's branch into it, resolve, and push:

```bash
git checkout -b <name> <earlier-branch>    # or the release tag
git merge <owner>/<branch>
# resolve the conflicts, then
git commit
git push <your-remote> <name>
```

In the spec, the line names your branch and keeps watching the author's branch after `last=`:

```
	k451  privkeyio/fix-reindex-init-deadlock-29+knots		last=3296b30e78f 2rdzy/fix-reindex-init-deadlock
```

That line is from the 29.4.3 spec. The merge is now clean, and when the author pushes new commits the build stops with "needs updates from upstream". Review them, merge them into your branch, and update the sha.

## Part 3: Compile

Install the build dependencies first: `doc/build-unix.md` in the tree lists them (a C++ compiler, CMake, Boost, libevent, and SQLite for the wallet). Then, from `../build`:

```bash
cmake -B build
cmake --build build -j"$(nproc)"
```

That builds `bitcoind`, `bitcoin-cli`, the wallet and the unit tests. Add `-D BUILD_GUI=ON` for `bitcoin-qt`. That needs the Qt 5 development packages, or Qt 6 with `-D WITH_QT_VERSION=6` as well.

## Part 4: Run it and check it is stable

A custom build is only as good as your testing. Do not put an untested binary on mainnet.

1. Run the tests for what your branches change. For the example spec:
   ```bash
   build/bin/test_bitcoin --run_test=crypto_tests
   build/test/functional/p2p_compactblocks.py
   ```
   Then run the full suites, `ctest --test-dir build` and `build/test/functional/test_runner.py`.
2. Sync it yourself. A full mainnet initial sync runs your build against millions of real transactions and is the best stress test there is.
3. Watch for asserts. Grep the log for `Assertion failed` and `Internal bug` and treat any hit as a blocker.
4. Leave it running for days, not minutes. Slow leaks, reorg handling, and peer churn only show up over time. Keep an eye on memory and peer count.

Two cautions:

- You are trusting your own build, not the release signatures. The spec is what makes that auditable: anyone can rebuild the same source and check it.
- Never point it at a datadir written by a newer version. Moving forward is safe, going backwards can force a full re-sync.

## Sharing

Once your Knots is a spec file, you can share the recipe instead of a binary. Anyone can rebuild your exact node, and "what is in this build" is answered by reading a few lines.
