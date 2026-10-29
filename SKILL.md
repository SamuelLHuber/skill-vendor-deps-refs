---
name: vendor-deps-refs
description: Set up or maintain ignored `.repos/` dependency references where only `.repos/AGENTS.md` and `.repos/README.md` are tracked. Use when vendoring dependency references, materializing upstream source for inspiration, or verifying pinned third-party source hashes.
---

# Vendor dependency refs

Use this skill when a project wants lightweight, reproducible access to upstream dependency source under `.repos/` without committing the source trees.

## Default model: lightweight hash-verified refs

By default:

- `.repos/AGENTS.md` and `.repos/README.md` are tracked.
- `.repos/<name>/` source trees are gitignored.
- `.repos/README.md` is the manifest of pinned upstream URLs, refs, immutable commits, deterministic tree SHA-256 hashes, and file counts.
- Agents materialize source by cloning/fetching the pinned commit, removing nested VCS metadata, and verifying the tree hash.

This is best when dependencies are used for reference, dependency inspection, API inspiration, migrations, or one-off source reading, and the project does not need the vendored code committed.

## Heavier alternatives: subtree or submodule

The user may choose to change these instructions for a specific repository.

If the user wants the dependency source brought into the repository instead of using ignored hash-verified refs, copy this skill into the project at:

```text
.agents/skills/vendor-deps-refs/
```

Then edit the project-local copy and `.repos/AGENTS.md` / `.repos/README.md` to document the chosen policy.

Recommended heavier option: `git subtree --squash` under `.repos/<name>/`. In that case, overwrite the `.repos/` gitignore policy so the subtree source is tracked, and document the allowed `git subtree add --squash` / `git subtree pull --squash` commands.

Alternative heavier option: Git submodules. Use only when the repository explicitly wants submodule semantics and contributors accept the operational complexity.

For the default lightweight mode, keep `.repos/<name>/` ignored and hash-verified.

## Setting up a project

Copy the bundled templates into the target repository:

- `templates/AGENTS.md` → `.repos/AGENTS.md`
- `templates/README.md` → `.repos/README.md`

Add this to the project root `.gitignore` for lightweight mode:

```gitignore
/.repos/*
!/.repos/
!/.repos/AGENTS.md
!/.repos/README.md
```

Verify ignore behavior:

```bash
git check-ignore -v .repos/example || true
git check-ignore -v .repos/AGENTS.md || true
git check-ignore -v .repos/README.md || true
```

`.repos/example` should be ignored. `.repos/AGENTS.md` and `.repos/README.md` should not be ignored.

## Adding or updating a dependency ref

1. Read `.repos/README.md`.
2. Resolve the requested upstream ref/tag to an immutable source commit.
3. Materialize the source into `.repos/<name>/`.
4. Remove nested VCS metadata.
5. Compute the deterministic tree SHA-256 and file count.
6. Update `.repos/README.md`.
7. Run project verification.

Use the exact commands and hash algorithm documented in `.repos/AGENTS.md`.
