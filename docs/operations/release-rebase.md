# Fork stack and rebase

The fork is a short stack of feature commits on an upstream release tag (`fork/<tag>`, for
example `fork/v0.1.3`). Each commit is one feature, builds on its own, and owns its tests and its
OpenSpec capability spec (`openspec/specs/<capability>/spec.md`). Live voice is a separate branch
on top of the stack (`fork/<tag>-live-voice`).

## Changing a feature

Commit the change, including any update to the feature's spec, as
`fixup! <title of the stack commit that owns it>`, then fold it in with
`GIT_SEQUENCE_EDITOR=: git rebase -i --autosquash <tag>`. Keep a new feature as a new commit at
the top of the stack. Before shipping, check every commit:

```sh
git rebase --exec 'vp run typecheck' <tag>
T3CODE_PLAYWRIGHT_EXECUTABLE=/usr/bin/google-chrome vp run check:changed <tag>
```

`T3CODE_PLAYWRIGHT_EXECUTABLE` points browser tests at a system Chrome on hosts where Playwright
cannot download its own.

## Moving to a new upstream release

1. Tag the current stack tip (`archive/fork-<tag>-<date>`) so nothing is lost.
2. Create `fork/<new-tag>` from the new upstream tag in a sibling worktree, then rebase the stack
   onto it commit by commit (`git rebase --onto <new-tag> <old-tag> <branch>`).
3. Where upstream now solves a problem a stack commit solves, drop that part of the commit rather
   than merging two fixes.
4. Rebase the live voice branch onto the new stack tip last.

Prefer stable release tags. Nightlies can carry large internal refactors.

## Acceptance before a release

- `check:changed` against the base tag passes, including browser tests.
- `node apps/server/scripts/responsiveness-e2e.ts --variants broker --repos 40` shows no
  multi-second probe stalls.

## Releasing

Tag the branch tip `v<upstream version>-t3custom.<n>`, for example `v0.1.7-t3custom.1`, and push
that one tag. `.github/workflows/fork.yml` checks it, builds the Linux x64 CLI archive, and publishes
it, with `SHA256SUMS`, as this repository's latest GitHub Release. Push tags one at a time: an
upstream prerelease tag would also match the workflow.

The build sets `T3CODE_RELEASE_REPOSITORY` to this repository, so a runtime installed from these
releases updates from them too: `t3 update` installs the newest one, and `t3 update <version>` a
specific one. A new machine installs with:

```sh
curl -fsSL https://raw.githubusercontent.com/<owner>/t3code/<tag>/scripts/install.sh |
  T3CODE_REPOSITORY=<owner>/t3code T3CODE_VERSION=<version> sh
```

The same workflow runs `vp run check:changed` against the upstream base tag on every push to a
`fork/**` branch. Upstream's other workflows expect upstream's runners and secrets. Disable them in
the fork's Actions settings and keep only Fork enabled.
