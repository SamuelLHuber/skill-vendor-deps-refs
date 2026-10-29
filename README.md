# skill-vendor-deps-refs

Pi/Agent skill for lightweight vendored dependency references under `.repos/`.

The default workflow tracks only `.repos/AGENTS.md` and `.repos/README.md`; dependency source trees are ignored, materialized on demand from pinned upstream commits, and verified with deterministic SHA-256 tree hashes.

Install by placing or symlinking this repository as:

```text
~/.pi/agent/skills/vendor-deps-refs
```

Or copy it into a repository-local skill directory:

```text
.agents/skills/vendor-deps-refs
```

Use a project-local copy when you want to customize the policy, for example to switch from lightweight hash-verified refs to `git subtree --squash` or Git submodules.
