# Handoff: everything-app staging repo

`everything-staging.bundle` is a complete git bundle of the new
`agvaughan/everything` monorepo (plan, specs, working scaffold), built in a
session that could not push to that repo directly (repo-attach approval was
broken). This directory is a transport mechanism only — delete it once the
bundle has landed.

## To restore (from any session that has agvaughan/everything access)

```sh
git clone https://github.com/agvaughan/agvaughan.github.io -b claude/ultraplan-everything-app-ujnfto planning
git clone planning/handoff/everything-staging.bundle everything
cd everything
git remote set-url origin https://github.com/agvaughan/everything
git push -u origin main
```

Then delete this `handoff/` directory from the planning branch.

## One-line prompt for a fresh Claude session started on agvaughan/everything

> Restore the repo from the git bundle at
> `handoff/everything-staging.bundle` on branch
> `claude/ultraplan-everything-app-ujnfto` of `agvaughan/agvaughan.github.io`
> (see HANDOFF.md next to it), push it to main here, then read ROADMAP.md
> and continue the next ready work packages.
