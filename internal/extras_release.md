# cracktrader_extras Release Notes

The `cracktrader-extras/` directory is maintained in-tree so development and CI
can cover the core library and optional packages together. When you need to
publish or sync the extras package to its dedicated repository, use `git subtree`
from the root of this repo:

```bash
# create/update a split branch that only contains the extras directory
EXTRAS_BRANCH=extras/latest
PREFIX=cracktrader-extras

git subtree split --prefix "$PREFIX" --branch "$EXTRAS_BRANCH"

# push to the remote repo you use for cracktrader-extras
# (replace origin-extras with the remote that points at github.com/.../cracktrader-extras)
git push origin-extras "$EXTRAS_BRANCH":main
```

After pushing, tag/releases can be cut in the extras repo as usual. When you
want to bring changes from the external repo back here, `git subtree pull` keeps
history clean:

```bash
git subtree pull --prefix "$PREFIX" origin-extras main
```

Keeping the subtree locally means tests and documentation stay in lockstep,
while publishing still produces a clean, standalone repository for package
releases.
