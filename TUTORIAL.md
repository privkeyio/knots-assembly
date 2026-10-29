# Reproduce Knots, then build your own

Bitcoin Knots is not a hand-maintained fork. Each release is rebuilt from three public inputs:

1. a base Bitcoin Core release tag,
2. a spec file: the list of PR branches to merge, in order,
3. the branches themselves.

A script called `assemble-knots.pl` reads the spec, checks out the base, and merges each branch in turn. Because it is deterministic, anyone can rebuild a release and confirm it matches what Luke published, and anyone can write their own spec to build a custom Knots.

This guide does both, using Knots 29.4.3. Every command in Parts 1 to 4 and the Part 5 package build was run for real. Installing the package on a StartOS server was not. For a later release, swap in its tag.

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
git config merge.conflictStyle diff3   # the assembler's conflict healer reads diff3 markers

# 2. work from the base tag and run the assembler on your spec
git worktree add --detach ../build v29.4.3.knots20260508
cd ../build
TZ=UTC perl ../knots-assembly/assemble-knots/assemble-knots.pl ../my.spec
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

## Part 5: Run it on StartOS (optional)

> **⚠️ STOP. ADVANCED USERS ONLY. ⚠️**
>
> **You are about to run an unsigned Bitcoin binary you built yourself, on a node that may hold real money. If you do not fully understand every step below, do not do this. Nobody else can undo a mistake here for you.**
>
> This replaces the signed, widely reproduced Knots release on your node with a binary you built yourself. That is a power tool, and it cuts both ways:
>
> - **It is unsigned and unaudited.** If your build, a dependency, or the recipe is wrong or tampered with, you can lose coins. The signatures you are bypassing exist for a reason.
> - **Back up your wallet first.** Assume something can go wrong. Use the package's Backup wallet action or a StartOS backup, and keep the copy off the box.
> - **You can corrupt your chain.** Reusing the existing datadir with the wrong version can force a full re-sync, or worse. Never go backwards in version.
> - **You are on your own.** Start9 and Knots maintainers do not support sideloaded custom builds.
>
> If any of that is unclear, this is not for you yet. Nobody else is responsible for what you do to your own box.

**Requires StartOS 0.4.0.2 or newer**, the OS version the package declares. It uses the flavor-versioned format (`#knots:...`).

If you run a StartOS server, you can run your sub-Knots on it and reuse the chain it already synced. A sub-Knots built on Knots 29.4.3 follows Bitcoin's change of proof-of-work algorithm to BLAKE2b just as the release does and uses the same data format, and StartOS keeps a package's data across version updates. So you ship your build as an update to the existing Knots package.

Start from the `POW` branch of [`Horizon-btc/knots-startos`](https://github.com/Horizon-btc/knots-startos/tree/POW), which packages Knots 29.4.3 as the `bitcoind` package with the `knots` flavor. You need Docker, Node.js with npm, `jq` and `start-cli` (see the [StartOS packaging setup](https://docs.start9.com/packaging/environment-setup.html)). `start-cli` only packs inside a packaging workspace, so make one first. From the directory that holds `my.spec`:

```bash
mkdir s9 && cd s9
start-cli s9pk init-workspace
git clone -b POW https://github.com/Horizon-btc/knots-startos
cd knots-startos
cp ../../my.spec sub-knots.spec
```

Then make two edits.

1. In the `Dockerfile`, build from your spec instead of downloading the signed release. Replace the whole build stage, from `# Build stage` down to the line before `# Final image`, with the assembly from Part 2 and a compile. For the stack spec above:
   ```dockerfile
   # Build stage: assemble sub-knots.spec and compile it
   FROM debian:stable-slim AS builder

   ARG ASSEMBLER_REF=892ab94e9ca71d11af07155daf43a398720ce43e

   RUN apt-get update && apt-get install -y --no-install-recommends \
           build-essential cmake pkgconf python3 git perl ca-certificates \
           libevent-dev libsqlite3-dev libboost-dev libzmq3-dev

   RUN git init -q /assembler \
    && git -C /assembler fetch -q --depth 1 https://github.com/bitcoinknots/assemble-deriv.git "$ASSEMBLER_REF" \
    && git -C /assembler checkout -q FETCH_HEAD

   COPY sub-knots.spec /sub-knots.spec

   WORKDIR /src
   RUN git init -q \
    && git config user.name sub-knots \
    && git config user.email sub-knots@localhost \
    && git config merge.conflictStyle diff3 \
    && git remote add origin https://github.com/bitcoinknots/bitcoin.git \
    && git remote add 2rdzy https://github.com/2rdzy/bitcoin.git \
    && git remote add origin-pull-k https://github.com/bitcoinknots/bitcoin.git \
    && git fetch -q --filter=blob:none origin tag v29.4.3.knots20260508 \
    && git fetch -q --filter=blob:none 2rdzy \
           +refs/heads/fix-muhash-self-division:refs/remotes/2rdzy/fix-muhash-self-division \
           +refs/heads/port-core-35550-sendcmpct-announce:refs/remotes/2rdzy/port-core-35550-sendcmpct-announce \
    && git fetch -q --filter=blob:none origin-pull-k \
           +refs/pull/453/head:refs/remotes/origin-pull-k/453/head \
           +refs/pull/452/head:refs/remotes/origin-pull-k/452/head \
    && git fetch -q --filter=tree:0 https://github.com/bitcoin/bitcoin.git master \
    && git branch -f master FETCH_HEAD \
    && git checkout -q --detach v29.4.3.knots20260508 \
    && TZ=UTC perl /assembler/assemble-knots.pl /sub-knots.spec </dev/null

   RUN cmake -B build -DCMAKE_BUILD_TYPE=Release -DBUILD_GUI=OFF -DBUILD_TESTS=OFF -DBUILD_BENCH=OFF -DWITH_ZMQ=ON \
    && cmake --build build -j"$(nproc)" --target bitcoind bitcoin-cli \
    && mkdir -p /build/bin \
    && strip -o /build/bin/bitcoind build/bin/bitcoind \
    && strip -o /build/bin/bitcoin-cli build/bin/bitcoin-cli
   ```
   Add one remote, branch fetch and PR head fetch per line in your spec. `ASSEMBLER_REF` is the commit the `assemble-knots` submodule in `knots-assembly` points at, which `git -C ../../knots-assembly rev-parse HEAD:assemble-knots` prints. The container has no git config, so this sets a git identity and `diff3` itself, and runs with `TZ=UTC` as Part 2 does.

   The release tarball is self-contained, but your build links shared libraries, so the final image needs them too. Add them to its `apt-get install` line:
   ```dockerfile
   RUN apt-get update && apt-get install -y curl e2fsprogs jq yq \
           libevent-core-2.1-7t64 libevent-extra-2.1-7t64 libevent-pthreads-2.1-7t64 libsqlite3-0 libzmq5
   ```
   These are the Debian 13 package names. If `debian:stable-slim` moves to a newer release, the build stops on any name that changed, so update them then.
2. In `startos/versions/current.ts`, bump the revision after the last colon, for example `#knots:29.4.3:1` to `#knots:29.4.3:2`, keeping the `bitcoind` package id and the `knots` flavor. That is what makes StartOS treat it as an in-place update and keep your chain volume. Before you build, check the installed version in your server's web interface: it must start with `#knots:` and be lower than the version you set here. Update `releaseNotes` in the same file to list your branches. Upstream's next revision will also count as newer, so an update from the registry replaces your build with the stock one. Check before accepting one.

Build it:

```bash
npm ci
make x86    # or make arm
```

The build log shows the assembler's `Merge k453 via ...` and `Merge k452 via ...` lines and `COMPLETE`, then the Knots compile, and ends with `bitcoind_x86_64.s9pk` (`bitcoind_aarch64.s9pk` for `make arm`). The package compiles its own Release binary in `debian:stable-slim` without tests, so it is not the exact binary you tested. Run Part 4 on your Part 3 build first, and watch the packaged node the same way once it runs. Then sideload the file from your server's web interface, or run `make install` with `start-cli` logged in to your server. StartOS should update the `bitcoind` package in place and keep the `main` volume, so your node comes back up on the same chain, now running your sub-Knots. That install step has not been tested for this guide.

The spec is not one of the files `make` watches, so after editing it run `rm -f *.s9pk` before `make`, or it repacks nothing and reports the old build. When a new release comes out, update your spec, discard the edits with `git checkout -- Dockerfile startos/versions/current.ts`, `git pull` the package, and redo them.

## Sharing

Once your Knots is a spec file, you can share the recipe instead of a binary. Anyone can rebuild your exact node, and "what is in this build" is answered by reading a few lines.
