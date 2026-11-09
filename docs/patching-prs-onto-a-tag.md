# Applying upstream PRs on top of a release tag and exporting them as patches

A general method for taking one or more pull requests from an upstream
repository, applying them on top of a specific release tag, and turning the
result into a numbered patch series that a build (CI, Nix, a packaging script)
can apply to a clean checkout of that tag.

Throughout, replace the placeholders:

| Placeholder | Meaning | Example |
|---|---|---|
| `<TAG>` | the release tag you want to build on | `open-design-v0.27.0` |
| `<UPSTREAM_URL>` | upstream repository | `https://github.com/nexu-io/open-design.git` |
| `<N>`, `<M>` | PR numbers | `8589`, `8773` |

---

## 1. The idea

You want `TAG + PR changes`, not `main + PR changes`. PR branches are usually
cut from a `main` that is newer than your tag, so a PR's raw diff or a naive
commit range can drag in unrelated upstream work. The method below avoids that
by working with the PR's *own commits* and replaying them onto the tag.

```
tag ──●  (your base)
       \
        ●─●─●─●   PR commits, replayed onto the tag with cherry-pick
```

Result: a branch whose history is `TAG` plus only the PR commits, exported with
`git format-patch`.

---

## 2. One-time setup

```bash
cd your-clone

# remote that points at the upstream repo (skip if "origin" already is upstream)
git remote add upstream <UPSTREAM_URL>
git fetch upstream --tags

# shallow clones break merge-base; make sure you have full history
git rev-parse --is-shallow-repository      # if "true":
git fetch --unshallow upstream

# lets git remember conflict resolutions, useful when you re-apply later
git config rerere.enabled true

# author identity (needed for git am / cherry-pick)
git config user.name  >/dev/null || git config user.name  "Your Name"
git config user.email >/dev/null || git config user.email "you@example.com"
```

Check the tag exists: `git rev-parse <TAG>^{commit}`.

---

## 3. Fetch the PRs as real commits

Fetch each PR head into a local branch. GitHub exposes every PR at
`refs/pull/<N>/head`:

```bash
git fetch upstream pull/<N>/head:pr-<N>
git fetch upstream pull/<M>/head:pr-<M>
git fetch upstream main
```

Do **not** rely on downloading `https://github.com/.../pull/<N>.patch` and
running `git am -3` on an old tag. If the PR was written against much newer
code, `git am -3` fails with
`sha1 information is lacking or useless ... could not build fake ancestor`,
because your repo does not contain the blobs the patch was made against.
Fetching the real commits (above) avoids that.

---

## 4. Find out what each PR really contains

```bash
# the PR's own commits, newest first, relative to current upstream main
git log --oneline --no-merges upstream/main..pr-<N>

# authoritative alternative if you have the GitHub CLI
gh pr view <N> --repo <owner>/<repo> --json commits --jq '.commits[].messageHeadline'
```

Sanity checks before you apply anything:

- **Count matches expectation.** A PR titled as one fix should show a handful
  of commits, not hundreds. A very long list means the range includes unrelated
  upstream commits (the PR branch merged `main` or is based oddly). Stop and
  inspect it.
- **Merge commits.** Compare `git log --oneline A..B | wc -l` with the
  `--no-merges` count. If they differ, a merge commit is in the range;
  cherry-pick the commits one by one instead of as a range
  (`git log --reverse --no-merges --format=%h upstream/main..pr-<N>`).
- **Already merged?** If the PR is already in upstream, a newer tag may contain
  it and you might not need a patch at all:
  ```bash
  git log upstream/main --oneline --grep='#<N>' | head -3
  git tag --contains <commit-sha> | head
  ```
- **Overlap between PRs.** Read the commit messages. A PR may already contain
  another PR's commits (for example "from upstream PR #6010"). Applying both
  duplicates work and conflicts. Pick the superset.
- **Stacking and order.** If PR B was built on PR A, apply A first. Apply
  independent PRs in the order that touches the most shared files first.

---

## 5. Build the branch on the tag

Always start from the tag, on a fresh branch, and keep earlier attempts under
their own names:

```bash
git branch backup-before-<topic>                    # optional safety copy
git switch -c <topic>-on-<TAG> <TAG>
```

Apply each PR's commits, oldest first. The range form handles ordering and
only uses the PR's own history:

```bash
# range form (when the earlier checks showed no merge commits)
git cherry-pick $(git merge-base upstream/main pr-<N>)..pr-<N>

# single-commit PR
git cherry-pick pr-<M>
```

Important: the lower bound is `merge-base(upstream/main, pr-<N>)`, **not**
`merge-base(<TAG>, pr-<N>)`. Using the tag's merge-base pulls in every upstream
commit between the tag and the point where the PR branched, and the first
unrelated commit produces confusing conflicts in files the PR never touched.

Confirm the result:

```bash
git log --oneline <TAG>..HEAD       # exactly the PR commits, nothing else
git status                          # clean, no cherry-pick in progress
```

---

## 6. Resolving conflicts

Conflicts are expected when the PR targets newer code than the tag. Work one
file at a time.

```bash
git status                                  # which files are unmerged
git checkout --conflict=diff3 <file>        # show the common ancestor too
git diff <file>                             # inspect the hunks
```

During a cherry-pick, the sides mean:

| Marker | Meaning |
|---|---|
| `<<<<<<< HEAD` ("ours") | your branch so far: tag + earlier commits |
| `>>>>>>> <sha>` ("theirs") | the commit being applied |
| `\|\|\|\|\|\|\| base` (diff3) | the common ancestor |

Guidelines:

- Keep the tag's code and re-add only what the PR intended. `git show <sha> --
  <file>` shows exactly what the PR changed in that file.
- Do **not** run `git checkout --theirs <file>` on a whole file unless you have
  confirmed the tag side has no changes that the PR does not know about. It
  silently deletes the tag's code.
- Imports are a common trap: merge both sides' import lists rather than
  choosing one.
- When both sides added a block at the same place (end of file, same list),
  keep both blocks and check the closing braces. Git's diff often pairs the
  wrong `}`; a missing or extra brace is the usual result.
- If two PRs both add the same helper or constant, keep the one that is a
  superset and delete the duplicate, otherwise you get "duplicate declaration"
  errors.

Before continuing, always verify, in this order:

```bash
# 1. no markers left anywhere
git grep -n -E '^(<<<<<<< |>>>>>>> )' -- . ':!*.md' || echo "clean"

# 2. the project's own typecheck / build / tests (names vary)
pnpm --filter <package> typecheck
pnpm --filter <package> test
```

Only then:

```bash
git add <resolved files>          # add AFTER the file content is final
git cherry-pick --continue
```

Common mistake: running `git add` on a file that still contains conflict
markers, or copying a resolved file in *after* `git add`. Git then commits the
markers. Nothing about the commit looks wrong, and a patch containing markers
still applies cleanly. The typecheck is what catches it.

Abort or skip when needed:

```bash
git cherry-pick --abort           # undo the whole in-progress pick
git cherry-pick --skip            # drop only this commit
```

If a commit turns out to be empty (the tag already has the change), skip it.

---

## 7. Export the patch series

```bash
rm -rf /tmp/patches
git format-patch -o /tmp/patches <TAG>..HEAD
ls /tmp/patches
```

This writes one file per commit, numbered in application order:

```
0001-first-commit-subject.patch
0002-second-commit-subject.patch
...
```

Each file carries the commit's author, date and message, so `git am` can
recreate the commits and `git apply` can apply the diffs.

Before you trust the series, check that no patch carries markers:

```bash
grep -l '^+<<<<<<<' /tmp/patches/*.patch || echo "no markers in patches"
```

---

## 8. Test the series on a clean tag checkout

This is the same thing your build will do. Use a detached worktree so your
working branch is untouched:

```bash
git worktree add --detach /tmp/tag-test <TAG>
cd /tmp/tag-test

# apply diffs only
for p in /tmp/patches/*.patch; do
  echo "== $p"
  git apply --verbose "$p" || break
done

# or recreate the commits (keeps messages, needs user.name/email)
# git am /tmp/patches/*.patch

cd - && git worktree remove --force /tmp/tag-test
```

`git apply` does not need the `patch` utility, and it handles renames and new
files reliably. A patch that passes here will pass in CI as long as CI starts
from the same tag.

Applying cleanly only proves the text fits. Also run the typecheck, build or
test suite inside the worktree before you remove it, for the reason in
section 6.

---

## 9. Deliver the series to your build

1. Put the patches in the build repo (for example `patches/`), and remove any
   old patch files first. A build step that applies `patches/*.patch`
   alphabetically will otherwise apply stale patches together with the new
   ones.
2. Apply with `git apply` in the build, from a checkout of the exact tag:

   ```yaml
   - name: Apply patch series
     working-directory: src
     run: |
       set -euo pipefail
       shopt -s nullglob
       patches=(../ci/patches/*.patch)
       [ "${#patches[@]}" -gt 0 ] || { echo "::error::no patches"; exit 1; }
       for p in "${patches[@]}"; do
         echo "applying $p"
         git apply --verbose "$p"
       done
   ```

3. Add a guard right after, so a bad patch fails in seconds instead of after a
   long install:

   ```yaml
   - name: Reject conflict markers
     working-directory: src
     run: |
       if git grep -n -E '^(<<<<<<< |>>>>>>> )' -- apps packages tools scripts; then
         echo "::error::conflict markers in patched source"; exit 1
       fi
   ```

4. If the build output is addressed by a hash (Nix, a release asset), update it
   after every rebuild. A forced re-release changes the file, so the hash
   changes too.
5. Update comments and release notes that name the PRs or the tag. They go
   stale quickly.

---

## 10. Updating to a newer tag later

Do not start over. Move your branch onto the new tag, resolving only what
changed:

```bash
git fetch upstream --tags
git rebase --onto <NEW_TAG> <OLD_TAG> <topic>-on-<OLD_TAG>
# resolve conflicts as in section 6, then:
git rebase --continue
```

With `rerere.enabled` set, resolutions you already made are replayed
automatically. Then re-export (section 7), re-test (section 8) and replace the
patches in the build repo (section 9). If the new tag already includes a PR,
git reports its commits as empty during the rebase; skip those and drop that
PR's patches from the build.

---

## 11. Undoing things

```bash
git cherry-pick --abort                      # in the middle of a pick
git reset --hard HEAD~1                      # drop the last commit entirely
git revert <sha>                             # add an undo commit instead
git reset --hard backup-before-<topic>       # return to the safety branch

# remove one commit from the middle of the branch
git rebase --onto <sha>~1 <sha>
```

Prefer `reset` over `revert` if you are about to export patches, because a
revert would add an extra "undo" patch to the series.

Never run `rm -rf` on a data directory in the same command as a restore
(`rm -rf X && mv X.bak X`) unless you have first confirmed `X` is the throwaway
copy. The `rm` runs first and cannot be undone.

---

## 12. Pitfalls checklist

| Symptom | Cause | Fix |
|---|---|---|
| `sha1 information is lacking or useless ... could not build fake ancestor` | `git am -3` on a patch whose base blobs your repo lacks | fetch the PR commits and cherry-pick (sections 3 and 5) |
| Conflicts in files the PR never touched | range started at the tag's merge-base, pulling in unrelated upstream commits | use `merge-base upstream/main pr-<N>` as the lower bound |
| Huge commit list for a small PR | PR branch includes merges or a stale base | inspect `git log --no-merges`; cherry-pick the PR's own commits only |
| Build fails with `Merge conflict marker encountered` | markers were committed (file added before it was resolved) | fix the file, `git commit --amend`, re-export; add the guard step |
| Duplicate declaration or missing brace after resolving | both sides added blocks at the same place | keep both blocks, check braces, remove duplicates |
| Patch applies locally but CI fails to build | `git apply` only checks text, not that the code compiles | run typecheck/tests in the clean worktree |
| Old behavior still present after swapping patches | old patch files left in `patches/` | `git rm patches/*.patch` before copying the new series |
| Hash mismatch in Nix or a release | the artifact was rebuilt | copy the new URL and hash from the job summary |

---

## 13. Command cheat sheet

```bash
# setup
git remote add upstream <UPSTREAM_URL>; git fetch upstream --tags

# fetch PRs
git fetch upstream pull/<N>/head:pr-<N>; git fetch upstream main

# inspect
git log --oneline --no-merges upstream/main..pr-<N>

# apply onto the tag
git switch -c topic-on-<TAG> <TAG>
git cherry-pick $(git merge-base upstream/main pr-<N>)..pr-<N>

# verify
git grep -n -E '^(<<<<<<< |>>>>>>> )' || echo clean
pnpm --filter <package> typecheck

# export
git format-patch -o /tmp/patches <TAG>..HEAD

# test on a clean tag
git worktree add --detach /tmp/tag-test <TAG>
(cd /tmp/tag-test && for p in /tmp/patches/*.patch; do git apply --verbose "$p" || break; done)
git worktree remove --force /tmp/tag-test
```
