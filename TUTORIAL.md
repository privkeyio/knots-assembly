# Reproduce Knots, then build your own

Bitcoin Knots is not a hand-maintained fork. Each release is rebuilt from three
public inputs:

1. a base Bitcoin Core release tag,
2. a spec file: the list of PR branches to merge, in order,
3. the branches themselves.

A script called `assemble-knots.pl` reads the spec, checks out the base, and
merges each branch in turn. Because it is deterministic, anyone can rebuild a
release and confirm it matches what Luke published, and anyone can write their
own spec to build a custom Knots.

This guide does both. Every command below was run for real.

## Part 1: Reproduce an official release

This confirms your machine and the public sources produce the exact release
commit Luke shipped.

```bash
git clone --recurse-submodules https://github.com/chrisguida/knots-assembly
cd knots-assembly
./.ci/run-all.sh v29.3.knots20260508
```

The tag is all you need. The script finds the matching spec, fetches every
branch, runs the assembler, and compares the result to the release tag. If it
matches, that release is reproducible from public sources. If it does not, a
branch upstream moved or was deleted.

You only need `git` and `perl`. The assembler is plain Perl with no extra
modules to install.

## Part 2: Build your own sub-Knots

A sub-Knots is a derivative of Knots, built the same way Knots is built from
Core. Instead of checking out Core and rebuilding all of Knots, you check out a
finished Knots release and merge only your own changes on top. The recipe stays
short and the build is fast.

### Simplest case: reproduce garbageman

garbageman is a filtering node that peers with libre-relay nodes. Its whole
recipe is three lines:

```
checkout v29.3.knots20260508
timestamp 2026-05-07 03:45:44
	n/a   chrisguida/garbageman-29.3						last=ccc7097e240 chrisguida/garbageman-29.3
```

Save it as `knots-garbageman.spec` and assemble it (see "Assembling by hand"
below, since it is not a release spec). The result is a clean merge with no
conflicts, and the source is identical to `chrisguida/bitcoin:garbageman-29.3`.
garbageman is just "Knots 29.3 plus one pinned branch."

### Real case: stack several branches

A spec is an ordered list of merges, so add one line per branch. The branch name
tells the assembler which remote to use:

| name in the spec | where it comes from |
|---|---|
| `chrisguida/garbageman-29.3` | `chrisguida/bitcoin`, branch `garbageman-29.3` |
| `privkeyio/rdts-miner-override` | `privkeyio/bitcoin`, branch `rdts-miner-override` |
| `origin-pull-k/319/head` | `bitcoinknots/bitcoin` PR 319 |
| `origin-pull/NNNN/head` | `bitcoin/bitcoin` PR NNNN |

The first column is the PR number, padded to six characters (`n/a` when there is no PR). Put the author's branch in the second column, the way the official specs do.

Here is garbageman plus four reduced-data policy PRs, written as branches on top of 29.3. These are refinements, not the RDTS softfork itself (that is already merged into Knots as #238). As of Knots 29.4.3, #319 and a reworked #320 have shipped, #323 was closed, and #324 is still open. They are here only as an example of stacking your own branches:

```
checkout v29.3.knots20260508
timestamp 2026-05-07 03:45:44
	n/a   chrisguida/garbageman-29.3						last=ccc7097e240 chrisguida/garbageman-29.3
	k319  privkeyio/fix-bip110-envelope-datacarrier			last=5e3153edef4
	k320  privkeyio/rdts-guard-opif-tapscript				last=74a8dd54ae8
	k323  privkeyio/fix-rdts-script-cache-inactive			last=d58e6f82909
	k324  privkeyio/rdts-miner-override						last=909736fb33b
```

All five merges come out clean, with 14 commits over the release. Both the
garbageman code and the PR changes are present.

Two things to know:

- Any branch with a `/` in its name needs `last=<sha>`, the branch head you last reviewed. Leave it off and the assembler stops with "Please provide last= for direct remote merge." On every build it compares that sha to the current head and stops with "needs updates from upstream" if the branch moved, so nothing new slips in unreviewed. For a PR line it checks the PR head, so `last=<sha>` alone is enough. An `n/a` line has no PR, so name the branch to check after the sha, as the garbageman line does.
- A PR's base is usually not the release. These PRs target `29.x-knots`, which is
  a little ahead of the 29.3 release, so each one also brings the few dev commits
  between the release and the branch. Here that is about two commits each and
  merges cleanly. If a branch is far ahead you will get conflicts, which is the
  sign to rebase it onto the release first.

### Assembling by hand

`run-all.sh` compares against a release tag, and a custom spec has none, so run
the assembler directly:

```bash
# 1. a bitcoin repo with the base tag and your branches as remotes
git clone https://github.com/bitcoinknots/bitcoin.git bitcoin && cd bitcoin
git remote add chrisguida https://github.com/chrisguida/bitcoin.git
git remote add privkeyio https://github.com/privkeyio/bitcoin.git
git remote add origin-pull-k https://github.com/bitcoinknots/bitcoin.git
git fetch --all
git fetch origin-pull-k '+refs/pull/*/head:refs/remotes/origin-pull-k/*/head'
git fetch https://github.com/bitcoin/bitcoin.git master
git branch -f master FETCH_HEAD   # the assembler's poison check needs a local master (Core's)

# 2. work from the base tag and run the assembler on your spec
git worktree add --detach ../build v29.3.knots20260508
cd ../build
perl ../knots-assembly/assemble-knots/assemble-knots.pl ../my.spec
```

Run `extract-remotes.pl < my.spec` to print the exact remotes and refs a spec
needs, then script the fetches from that.

### When a merge conflicts

Pick branches that touch the same lines and some merges will conflict. You resolve each conflict once, and the assembler reuses your fix on every later build.

When a merge conflicts, the assembler hashes the conflicted diff with `git patch-id --stable` to get a conflict ID. If `assemble-knots-resolutions/<id>.diff` exists, it applies that file and moves on:

```
Conflict ID: <id> AUTOPATCHED with assemble-knots-resolutions/<id>.diff
```

If not, it shows the conflict and the file name to save a fix under, then asks:

```
Conflict ID: <id> (use assemble-knots-resolutions/<id>.diff)
Conflict found: 1) Fix, 2) Abort, or 3) Tree-merge
```

Type `1` and the assembler suspends itself, leaving the conflicted merge in your tree. Resolve the files, save the fix, and resume:

```bash
git diff HEAD > assemble-knots-resolutions/<id>.diff
fg
```

The next build hits the same conflict, gets the same ID, and applies your fix without asking. Commit the `.diff` files next to your spec so anyone can rebuild your mix. This folder is how the official specs merge hundreds of PRs and still rebuild unattended.

The folder is looked up in the directory you run the assembler from, so in the setup above, link it into `../build` before you run:

```bash
ln -s ../knots-assembly/assemble-knots-resolutions .
```

## Part 3: Compile

Knots 29.3 adds the reduced-data softfork (RDTS, BIP110), so the build asks you
to state your consent to the deployment logic:

```bash
cmake -B build -D RDTS_CONSENT=IMPLICIT \
      -D BUILD_GUI=OFF -D BUILD_TESTS=OFF -D BUILD_BENCH=OFF
cmake --build build -j"$(nproc)" --target bitcoind bitcoin-cli
```

Drop the `OFF` flags if you want the GUI and tests. For a filtering node, do not
run with `-corepolicy`: it switches to Core policy and turns off the filtering
that a garbageman node is there to do.

## Part 4: Run it and check it is stable

A custom build is only as good as your testing. Do not put an untested binary on
mainnet.

1. Sync it yourself first. A full mainnet initial sync runs your build against
   millions of real transactions and is the best stress test there is. Policy
   bugs tend to show up during sync.
2. Watch for asserts. Filtering and reduced-data code add new checks. Grep the
   log for `Assertion failed` and `Internal bug` and treat any hit as a blocker.
3. Run the functional tests before you deploy:
   `build/test/functional/test_runner.py`. Add tests for whatever your PRs
   change.
4. Leave it running for days, not minutes. Slow leaks, reorg handling, and peer
   churn only show up over time. Keep an eye on memory and peer count.
5. If it is a garbageman node, check the disguise. A filtering node can be spotted
   by what it will not relay, so audit your own node before you rely on it.

For a one-command deploy with a node, Tor, and a watchdog, see
`libre-recon/deploy`.

## Part 5: Run it on Start9 (optional)

> ## ⚠️ STOP. ADVANCED USERS ONLY. ⚠️
>
> **You are about to run an unsigned Bitcoin binary you built yourself, on a node
> that may hold real money. If you do not fully understand every step below, do
> not do this. Nobody else can undo a mistake here for you.**
>
> This replaces the signed, widely reproduced Knots release on your node with a
> binary you built yourself. That is a power tool, and it cuts both ways:
>
> - **It is unsigned and unaudited.** If your build, a dependency, or the recipe
>   is wrong or tampered with, you can lose coins. The signatures you are
>   bypassing exist for a reason.
> - **Back up your wallet first.** Assume something can go wrong and keep a seed
>   backup off the box.
> - **You can corrupt your chain.** Reusing the existing datadir with the wrong
>   version can force a full re-sync, or worse. Never go backwards in version.
> - **You are on your own.** Start9 and Knots maintainers do not support
>   sideloaded custom builds.
> - **garbageman is adversarial software.** Running it means your node may be
>   identified and banned. That is the point; know you signed up for it.
>
> If any of that is unclear, this is not for you yet. Nobody else is responsible
> for what you do to your own box.

**Requires StartOS 0.4.0 or newer.** This uses the flavor-versioned package
format (`#knots:...`, SDK 1.5.3), which does not exist on StartOS 0.3.5.x. If your
Knots package version already shows the `#knots:` prefix you are on the 0.4.0
line; if not, this will not install. Check before you start.

If you run a Start9 (StartOS) box, you can run your sub-Knots on it and reuse the
chain it already synced, no 800GB re-download. Two facts make this work: a
sub-Knots built on Knots 29.3 uses the identical Bitcoin Core data format, and
StartOS keeps a package's data across version updates. So you ship your build as
an update to the existing Knots package.

Start from a fork of the official package,
[`Start9Labs/bitcoin-knots-startos`](https://github.com/Start9Labs/bitcoin-knots-startos),
and change two things.

First, build from source instead of downloading the signed release. In the
`Dockerfile`, replace the "download tarball, verify signatures" build stage with
a from-source build of your recipe: check out the Knots release, merge your
feature branches, then `cmake` with `RDTS_CONSENT=IMPLICIT`. The final image also
needs the runtime libraries your binary links (libevent, libsqlite3), which the
self-contained release tarball did not.

Second, bump the version in `startos/versions/current.ts` (for example
`29.3.1:10` to `29.3.1:11`), keeping the same package id `bitcoind` and `knots`
flavor. That is what makes StartOS treat it as an in-place update and keep your
chain volume.

A complete working example, both changes applied for garbageman plus the four
policy branches from Part 2, is in the
[`garbageman-rdts` fork branch](https://github.com/privkeyio/bitcoin-knots-startos/tree/garbageman-rdts).

Then build the package for your box's architecture and sideload it:

```bash
make x86        # Start9 boxes are usually arm64: use `make arm`
# produces bitcoind_x86_64.s9pk -> sideload it via the StartOS UI
```

StartOS updates the `bitcoind` package, keeps the `main` volume, and your node
comes back up on the same chain, now running your sub-Knots.

Two cautions:

- You are trusting your own build, not the release signatures. The spec is what
  makes that auditable: anyone can reproduce the same binary and check it.
- Never install a version older than the one that wrote the datadir. A
  same-version sub-Knots (29.3) is safe.

## Sharing

Once your Knots is a spec file, you can share the recipe instead of a binary.
Anyone can rebuild your exact node, and "what is in this build" is answered by
reading a few lines.
