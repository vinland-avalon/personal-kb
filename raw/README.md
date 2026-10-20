# raw/

All sources live here.

- Google Doc: export markdown, put the Doc URL in the file, drop it here, then `KB-SYNC` (or `KB-SYNC <path>`).
- Session notes: start a message with `KB-PERSIST`; the agent writes under `sessions/`. Fold them later with `KB-SYNC`.
- After `KB-SYNC`, the agent moves folded files to `processed/`, keeping the relative path.
- Agent must not edit existing file contents, except writing a new `sessions/` file on `KB-PERSIST`.
