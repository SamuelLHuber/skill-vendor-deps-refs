# Agent Instructions: Ignored `.repos/` Dependency Refs

> Ground truth lives in `.repos/README.md`.
> The vendored source trees under `.repos/<name>/` are local materializations only.
> They are intentionally ignored by Git and must be recreated from the manifest.

## 1. Tracking policy

Only these files under `.repos/` should be tracked in lightweight mode:

```text
.repos/AGENTS.md
.repos/README.md
```

Everything else under `.repos/` is ignored and must be reproducible from the manifest.

The project root `.gitignore` should contain:

```gitignore
/.repos/*
!/.repos/
!/.repos/AGENTS.md
!/.repos/README.md
```

Do not commit vendored source trees, nested clones, archives, or generated dependency snapshots under `.repos/` in lightweight mode.

## 2. When to use a heavier mode

The default is lightweight hash-verified refs. It is good for dependency inspection, source reading, API inspiration, and reproducible local context for agents.

A repository may intentionally choose a heavier policy:

- **Recommended heavier option:** `git subtree --squash` under `.repos/<name>/` when the source should be committed into the repository.
- **Alternative:** Git submodules when the project explicitly wants submodule semantics.

If using subtree or submodules, overwrite this file and `.repos/README.md` to document that policy, and change the `.gitignore` rules so the intended files are tracked.

## 3. Manifest policy

`.repos/README.md` is the source of truth. Each lightweight dependency row must include:

- `Name`
- `Path`
- `Upstream`
- `Ref`
- `Source commit`
- `Tree SHA-256`
- `Files`

The `Ref` may be a tag, release branch, or other upstream ref, but the `Source commit` must be immutable.

If the manifest and the local `.repos/<name>/` tree disagree, the manifest wins unless the user explicitly asks to update the pin.

## 4. Materializing a dependency

From the repository root:

```bash
name="<name>"
upstream="<upstream-url>"
commit="<source-commit>"

tmp="$(mktemp -d)"
trap 'rm -rf "$tmp"' EXIT

git clone --no-checkout "$upstream" "$tmp/repo"
git -C "$tmp/repo" checkout --detach "$commit"

rm -rf "$tmp/repo/.git" "$tmp/repo/.jj" "$tmp/repo/.hg" "$tmp/repo/.svn"
rm -rf ".repos/$name"
mkdir -p .repos
mv "$tmp/repo" ".repos/$name"
```

Never leave nested VCS metadata under `.repos/<name>/` in lightweight mode.

## 5. Computing the deterministic tree hash

Use this exact hash algorithm unless the project documents a replacement.

```bash
python3 - "$PWD/.repos/<name>" <<'PY'
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

The first output field is the `Tree SHA-256`.
The second output field is the `Files` count.

## 6. Verifying all materialized dependencies

For each row in `.repos/README.md`:

1. Confirm `.repos/<name>/` exists.
2. Recompute the deterministic tree hash.
3. Compare it to `Tree SHA-256`.
4. Compare the file count to `Files`.

If any dependency fails verification, do not patch around it. Re-materialize from the pinned source commit.

## 7. Adding a new dependency ref

1. Resolve the upstream ref to an immutable source commit.

   Example:

   ```bash
   git ls-remote <upstream-url> <ref>
   ```

   For annotated tags, peel the tag and record the commit, not the tag object.

2. Materialize the dependency into `.repos/<name>/`.
3. Compute the deterministic tree hash.
4. Add a row to `.repos/README.md`.
5. Add or update the materialization command in `.repos/README.md`.
6. Run the project's verification command.

## 8. Updating an existing dependency ref

1. Read the current row in `.repos/README.md`.
2. Resolve the new upstream ref to an immutable source commit.
3. Replace `.repos/<name>/` by re-materializing from that commit.
4. Recompute the tree hash and file count.
5. Update `.repos/README.md`.
6. Run the project's verification command.

## 9. Forbidden actions in lightweight mode

Do not:

- commit `.repos/<name>/` source trees;
- use unpinned branch heads as the only source of truth;
- leave `.git`, `.jj`, `.hg`, or `.svn` directories inside `.repos/<name>/`;
- silently modify vendored files without updating the digest;
- trust an existing local `.repos/<name>/` tree without hash verification;
- use `curl | tar` or downloaded archives unless the project explicitly documents archive hashing instead of Git source commits.
