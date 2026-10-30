# Maintaining a Knots spec

[`TUTORIAL.md`](TUTORIAL.md) shows how to build your own Knots and [`CONTRIBUTING.md`](CONTRIBUTING.md) shows how to get a change into Knots. This covers the part in between: keeping the in-progress spec (`knots-next-29.spec`) ready for the next release, so the maintainer can assemble it and tag. The worked example throughout is Knots 29.4.3: its release spec [`knots-29.4.3.knots20260508.spec`](knots-29.4.3.knots20260508.spec) and the `knots-next-29.spec` that followed it. Behavior described here comes from [`assemble-knots.pl`](https://github.com/bitcoinknots/assemble-deriv) itself, cited by function, and from the maintainer's own spec commits.

## The cycle

1. After a release is tagged, its output spec becomes the new `knots-next-NN.spec`.
2. Lines are added, updated and removed as PRs land.
3. When Core ships a new point release, the spec moves onto it.
4. The release tail (version bump, release notes, manpages) is updated for each rc and the final.
5. The spec is verified and handed to the maintainer, who assembles it on his machine and tags the result.

## Spec format

Tabs are 4 wide. A merge line is a tab, the PR number padded to six characters (a few tail lines use five), the branch, the hash column, then `last=`. Two lines from the "New -next-29" spec:

```text
	33433 qafix_rpc_bind_nonloopback_unavail	ab92a1c42e4
	k451  privkeyio/fix-reindex-init-deadlock-29+knots	12e68a2226a	last=3296b30e78f 2rdzy/fix-reindex-init-deadlock
```

- The PR field is `k<n>` for a Knots PR, a bare number for a Core PR, `g<n>` for a Core GUI PR, and `-` or `n/a` when there is none.
- A branch with a `/` is `<owner>/<branch>` on that owner's fork. A bare branch name is one of the maintainer's own branches.
- The hash column is the merge commit this line produced in the last assembly. It starts at column 48 (counting from 0, with 4-wide tabs), and `last=` starts at column 60. A line too long to reach a column gets a single tab instead. Use 11-character hashes.
- `last=<sha>` is the upstream head that was last reviewed. If the branch column is our own copy, `last=` names the author's branch after the sha, so the update check still watches it. Never repeat the branch-column name after `last=`.
- A prefix in front of the tab changes the merge: `TM` is a tree merge (see below), `NM` a null merge.
- Everything after `#` is a comment, so a line starting with `#` (including markers like `#30.xTODO#`) is not merged.

Extra upstreams are checked with `(CHECK-LAST)` lines, placed right after the line they belong to and before its comments:

```text
	13789 asm_bypass_cxxflags					529608a4a19
	(CHECK-LAST)	last=6878dd8f1ab origin-pull-k/309/head
		# + knots#309
```

The assembler stops with `CHECK-LAST failed for upstream <upstream>` if that upstream has moved. The upstream can be a PR head such as `origin-pull-k/309/head` or an author branch such as `bigshiny90/gui-darkmode-updates`, and the maintainer's specs use both. A PR head stays fetchable after the author deletes the branch, which matters for tools that fetch exact branch names, like this repo's CI. The `# + knots#NNN` comment names a follow-up PR that is tracked this way.

The header sets `timestamp` (the author and committer date for every assembled commit) and `checkout <tag>` (the base). `@<name>` lines name a section of the assembly, `n/a (cherrypick=…)` and `n/a (bump_version=…)` lines make up most of the release tail, and an `n/a (delete_release_notes_fragments)` line just before it removes release-note fragments.

## The update check

For every ordinary merge line with `last=` (not `TM` or `NM` lines, which skip the check), the assembler looks up the newest upstream and stops if `last=` does not match it (`<prnum> <branch> needs updates from upstream <upstream>`). `get_latest_upstream` builds the candidate list from the branch named after `last=` plus the PR head, `origin-pull/<n>/head` (`origin-pull-k` for `k` lines, `origin-pull-g` for `g` lines), and takes whichever has the newest commit time. A `!` in front of the named branch drops the PR head from the list, so do not use `!` normally. A branch with a `/` and no `last=` stops the run with `Please provide last= for direct remote merge of …`.

`--skip-update-check` turns this off. It exists for reproducing old releases, whose branches have moved on since. Never rely on it for a spec you are preparing: a mistyped `last=` passes a skipped check and then stops the maintainer's run. `(CHECK-LAST)` lines are checked even with the flag.

Before pushing a spec change, resolve every hash you added or changed on GitHub. List them:

```bash
git diff <base> HEAD -- '*.spec' | grep '^+[^+]' | sed 's/#.*//' | grep -oE '(last=|cherrypick=)?[0-9a-f]{10,40}' | sort -u
```

Then check each one. The commits endpoint catches typos but resolves a sha anywhere in the fork network, so for every `last=<sha> <owner>/<branch>` also check that the full sha is that branch's current head:

```bash
gh api repos/<owner>/bitcoin/commits/<sha> --jq .sha
gh api repos/<owner>/bitcoin/branches/<branch> --jq .commit.sha
```

## Starting the next spec

After a release is tagged, assemble its spec with `-o`. The output spec is the same file with every hash column rewritten to the merge commits that assembly made, and it becomes the new `knots-next-NN.spec`. The maintainer's "New -next-29" commit after 29.4.3 is exactly the output spec of `knots-29.4.3.knots20260508.spec`, with 11-character hashes.

The maintainer runs the assembler on his own branches. To produce the same file without them, use the reproduction from TUTORIAL Part 1:

```bash
./.ci/run-all.sh v29.4.3.knots20260508
```

It writes `out/29.4.3.knots20260508.out.spec`. Its 472 hash columns match the maintainer's "New -next-29" commit. Three things differ, all from the reproduction rather than the assembler: it comments out the `(CHECK-LAST)` lines, it prints 10-character hashes, and it may substitute a cherry-pick source that is not public. Put those back as they are in the release spec and lengthen the hashes with `git rev-parse --short=11`.

Lines added since the last assembly have no hash column yet. The next assembly fills it in.

## Moving onto a new Core point release

The maintainer's "next-29: Begin rebase on 29.4" commit starts the move from 29.3 to 29.4 (later "update" commits continue it). An excerpt:

```diff
-checkout v29.3
+checkout v29.4
-	34093 fix_freebsd15_netlink_warn			166ab5d85a4	last=c1361fc42dd vasild/fix_nlmsg_ok_compilation_fbsd15
+TM	34093 fix_freebsd15_netlink_warn			166ab5d85a4	last=c1361fc42dd vasild/fix_nlmsg_ok_compilation_fbsd15
-	34561 docfix_rpcwallet_send_eg_pr34561-23	a9524a0ef99	last=50cf6838e6a
+TM	34561 docfix_rpcwallet_send_eg_pr34561-23	a9524a0ef99	last=50cf6838e6a
```

1. Change `checkout` to the new Core tag.
2. Mark every line the tag already includes as `TM`. A tree merge commits HEAD's tree unchanged, with the line's previous merge commit as an extra parent, so the history stays connected without changing any code.
3. Never delete a line that has a hash. Each merge takes the previous assembly's merge for that line as a parent, and the assembler checks that this previous merge is exactly one first-parent step beyond what it has assembled so far. Delete a hashed line and its old merge is never assembled, so the next line fails that check with `Skipping a parent in rebase! Aborting`. Mark it `TM` instead.
4. Add any new line without a hash column.

A line that conflicts with the new base or with an earlier line gets a merge branch, not a hand fix during assembly. k451 in the 29.4.3 spec is one:

```text
	k451  privkeyio/fix-reindex-init-deadlock-29+knots		last=3296b30e78f 2rdzy/fix-reindex-init-deadlock
```

The author's head 3296b30e78f conflicts in `src/init.cpp`. The merge branch starts from the commit the assembler made for the line just before k451 (`Merge 34253 via lockless_isibd-26+knots`, 79045eca69, from the 29.4.2 assembly) and merges the author's head on top:

```bash
git checkout -b fix-reindex-init-deadlock-29+knots 79045eca69
git merge 2rdzy/fix-reindex-init-deadlock
# resolve the conflicts, then
git commit
git push privkeyio fix-reindex-init-deadlock-29+knots
```

Name it `<branch>+knots` (with the series, for example `-29+knots`) on your fork. The spec line names your branch and keeps watching the author's after `last=`. When the author pushes, the update check stops the build. If no pending release spec uses your merge branch, merge their new commits into it, push, and update `last=`. If one does, leave that branch alone and push the update under a new name.

### Old-style resolution files

Older specs resolve conflicts with files in `assemble-knots-resolutions/`. When a merge conflicts, `mymerger` takes the `git patch-id` of the conflicted `git diff HEAD`, and applies `assemble-knots-resolutions/<id>.diff` (or follows `<id>.res`) if one exists. The assembler still reads them, and released specs still depend on them. New conflicts get a merge branch instead, which is now the standard (k451 above). Two settings the conflict handling depends on:

- `merge.conflictStyle diff3`. The automatic conflict healer (`smartconflicthealer`) parses diff3 markers.
- Default diff prefixes. `diff.mnemonicPrefix` or `diff.noprefix` changes the diff the healer reads, so it gives up with the warning `Unparsable file-begin line`, and conflicts it would have resolved fall through to the base-branch test or the interactive prompt. Turn both off for assembler runs, for example with `GIT_CONFIG_COUNT=2 GIT_CONFIG_KEY_0=diff.mnemonicPrefix GIT_CONFIG_VALUE_0=false GIT_CONFIG_KEY_1=diff.noprefix GIT_CONFIG_VALUE_1=false`.

The resolution directory is looked up relative to the current directory, so run the assembler where `assemble-knots-resolutions` is visible (CI links it into the worktree).

## The release tail

The tail is the block of merge lines at the end of the spec. Its main lines in the 29.4.3 spec (other lines in between are left out):

```text
	n/a   (cherrypick=488640fe20b)				37b6ff5929f	# doc/{bips,files}
	n/a  (bump_version=knots20260508)			39900a25630
	n/a   (cherrypick=60960ffdcb9)				05089efa62a	# bump to 29.4.3
	n/a   (cherrypick=e9fe32cca5f)				fcb5053e061  # release notes: write/update, including change log and credits
	n/a  (cherrypick=a7c4617524c)				58398baf33e  # update manpages (build first)
```

A `cherrypick=` line cherry-picks that commit, and a `bump_version=` line merges its previous commit and rewrites the version. Both record the previous assembly's commit for the line as an extra parent, so the tail keeps one history across rcs. For each rc:

- Point `cherrypick=` at the exact commit generated in the previous assembled branch.
- Revise the one release-notes line. Do not add a second one.
- Swap the rc bump line (`# bump to rcN`).
- Regenerate manpages, the example `bitcoin.conf`, and shell completions, then point the manpages line at the new commit.

The manpages must come from a Release build, or the `lock` log level ends up in them (the spec carries this warning). These commands reproduced the 29.4.3 manpages commit a7c4617524 byte for byte, from its parent:

```bash
cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=Release -DBUILD_GUI=ON -DWITH_ZMQ=ON -DWITH_MINIUPNPC=ON
cmake --build build -j"$(nproc)"
env -u DISPLAY -u WAYLAND_DISPLAY QT_QPA_PLATFORM=offscreen QT_QPA_PLATFORMTHEME= TOPDIR="$PWD" BUILDDIR="$PWD/build" MANDIR="$PWD/doc/man" python3 contrib/devtools/gen-manpages.py
TOPDIR="$PWD" BUILDDIR="$PWD/build" bash contrib/devtools/gen-bitcoin-conf.sh
mkdir -p contrib/completions/zsh
python3 build/test/functional/tool_cli_completion.py --overwrite
```

`gen-manpages.py` needs `help2man`, and without a usable display `bitcoin-qt --version` aborts unless the Qt variables above are set. Stale completions fail the functional test `tool_cli_completion.py` (`assert_equal(bash_completion, existing_bash)`), so a tree without the regenerated tail fails it, locally and on branch pushes. Pull request CI skips the test.

## What the output spec does

`-o` writes the spec back out with the hash column of every merge line it ran replaced by the commit it just made (`replace_lastapply`, filled from `git rev-parse --short HEAD`). The assembler sets the committer name to `merge-script` and the date from the `timestamp` line, which has no time zone. So the commits still depend on your committer email and local time zone, and a local assembly only produces the maintainer's hashes when both match his, as the CI reproduction pins them. Your output spec is for checking your own work. What must be right in a spec you hand over:

- branch names and the order of lines,
- every `last=` and its upstream,
- every `cherrypick=` source,
- `(CHECK-LAST)` lines,
- the hash columns, which must stay the maintainer's (from the last "New -next" commit) because each one becomes a parent of the next merge. Leave new lines without one.

## Verifying before handing a spec over

1. Assemble the spec from scratch with the update check on (see "Assembling by hand" in [`TUTORIAL.md`](TUTORIAL.md)), and check the tree matches the assembly you tested. Fetch every remote the spec names first, including the upstreams after `last=` and in `(CHECK-LAST)` lines. The assembler only fetches when run with `-f`, so otherwise it compares against whatever refs are already local.
2. Build and run the unit tests in a Release build and in a Debug build. Debug builds define `ABORT_ON_FAILED_ASSUME`, along with `DEBUG_LOCKORDER` and `RPC_DOC_CHECK`, so a failed `Assume()` aborts there while a Release build carries on. A test that only breaks an `Assume()` passes in Release.
   ```bash
   cmake -B build-debug -G Ninja -DCMAKE_BUILD_TYPE=Debug
   cmake --build build-debug -j"$(nproc)" --target test_bitcoin
   build-debug/bin/test_bitcoin
   ```
3. Run the functional tests, `build/test/functional/test_runner.py`.
4. Push the final tree, release tail included, to a branch of your fork and let CI run. Use a branch push, not a pull request, since pull request CI skips `tool_cli_completion.py`.

## After the maintainer tags

The maintainer assembles the spec on his machine and tags the result. His tagged commits are the release, his release spec is committed as `knots-<version>.spec`, and his output spec becomes the next `knots-next-NN.spec` (see "Starting the next spec"). From then on every hash column refers to his commits. Community test builds of an rc are built from his tag.
