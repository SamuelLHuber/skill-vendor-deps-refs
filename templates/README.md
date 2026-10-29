# Vendored Dependency Refs

This directory contains reproducible local materializations of upstream repositories for dependency inspection and inspiration.

Default lightweight mode commits only these files:

```text
.repos/AGENTS.md
.repos/README.md
```

All actual dependency source trees under `.repos/<name>/` are ignored by Git and must be recreated from this manifest.

See [AGENTS.md](AGENTS.md) for the full policy and verification commands.

## Git ignore policy

For lightweight mode, the project root `.gitignore` should contain:

```gitignore
/.repos/*
!/.repos/
!/.repos/AGENTS.md
!/.repos/README.md
```

If the project intentionally uses `git subtree --squash` or submodules to bring dependency source into the repository, replace this policy with project-specific instructions and track the intended files.

## Manifest

| Name | Path | Upstream | Ref | Source commit | Tree SHA-256 | Files |
|------|------|----------|-----|---------------|--------------|-------|
| `example` | `.repos/example/` | `https://github.com/example/example.git` | `v1.2.3` | `0123456789abcdef0123456789abcdef01234567` | `replace-with-tree-sha256` | `0` |

## Materialize dependencies

Run from the repository root.

### example

```bash
name="example"
upstream="https://github.com/example/example.git"
commit="0123456789abcdef0123456789abcdef01234567"

tmp="$(mktemp -d)"
trap 'rm -rf "$tmp"' EXIT

git clone --no-checkout "$upstream" "$tmp/repo"
git -C "$tmp/repo" checkout --detach "$commit"

rm -rf "$tmp/repo/.git" "$tmp/repo/.jj" "$tmp/repo/.hg" "$tmp/repo/.svn"
rm -rf ".repos/$name"
mkdir -p .repos
mv "$tmp/repo" ".repos/$name"
```

Then verify the hash:

```bash
python3 - "$PWD/.repos/example" <<'PY'
import hashlib
import os
import stat
import sys

root = sys.argv[1]
h = hashlib.sha256()
count = 0

for dirpath, dirnames, filenames in os.walk(root):
    dirnames[:] = sorted(
        d for d in dirnames
        if d not in {".git", ".jj", ".hg", ".svn"}
    )

    for filename in sorted(filenames):
        path = os.path.join(dirpath, filename)
        rel = os.path.relpath(path, root).replace(os.sep, "/")
        st = os.lstat(path)

        if stat.S_ISLNK(st.st_mode):
            kind = b"symlink"
            data = os.readlink(path).encode()
        elif stat.S_ISREG(st.st_mode):
            kind = b"file"
            with open(path, "rb") as f:
                data = f.read()
        else:
            continue

        h.update(kind + b"\0")
        h.update(rel.encode() + b"\0")
        h.update(data + b"\0")
        count += 1

print(f"{h.hexdigest()}  {count}")
PY
```

The output must match the `Tree SHA-256` and `Files` columns in the manifest.

## Updating a dependency ref

1. Resolve the new upstream ref to a commit.
2. Re-materialize `.repos/<name>/`.
3. Recompute the deterministic tree hash.
4. Update the manifest row.
5. Run project tests or verification.

Example:

```bash
git ls-remote https://github.com/example/example.git refs/tags/v1.2.4
```

For annotated tags, record the peeled commit, not the tag object.

## Verification checklist

Before finishing any change involving `.repos/`:

- [ ] `.repos/AGENTS.md` is tracked.
- [ ] `.repos/README.md` is tracked.
- [ ] `.repos/<name>/` source trees are ignored in lightweight mode.
- [ ] No nested VCS metadata exists under `.repos/<name>/` in lightweight mode.
- [ ] Every manifest row has an immutable source commit.
- [ ] Every materialized tree hash matches the manifest.
- [ ] Project tests or verification commands pass.
