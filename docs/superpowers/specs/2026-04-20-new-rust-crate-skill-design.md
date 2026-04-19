# `new-rust-crate` skill — design

Status: approved, ready for implementation plan
Date: 2026-04-20
Author: Jinguo Liu (`cacate0129@gmail.com`)

## Purpose

A personal Claude Code skill that scaffolds a new Rust package with the
structural conventions the author already maintains in
`~/rcode/problemreductions` — a library crate plus optional CLI subcrate,
Makefile-driven workflow, mdBook documentation, GitHub Actions for CI /
docs / release, codecov integration, and an agent-instruction stub under
`.claude/`.

The skill turns a 30-minute manual setup (copy configs, edit names, wire
CI, enable Pages, add secrets) into a single guided conversation.

## Reference package

The design draws all conventions from `~/rcode/problemreductions` (v0.5.0),
specifically:

- Cargo workspace with root lib + `problemreductions-cli` + `problemreductions-macros`
- `Makefile` targets: `build / test / fmt / fmt-check / clippy / doc / mdbook /
  coverage / check / release`
- `rustfmt.toml` (edition 2021, `max_width = 100`, explicit abi)
- `codecov.yml` with a 95% project target and patch target, ignoring the macro
  and CLI crates
- `book.toml` driving an mdBook under `docs/src/`
- `.github/workflows/{ci,docs,release}.yml`
- `AGENTS.md` redirecting to `.claude/CLAUDE.md`
- Examples under `examples/`, tests under `tests/main.rs`

Problem-reductions-specific assets (the `.claude/skills/` tree, benches,
Typst paper pipeline, Julia parity tooling, research papers) are **not**
carried into the skill — they are domain-specific and would misshape
unrelated projects.

## Skill location

Dual installation:

1. `~/.claude/skills/new-rust-crate/` — primary copy; always available
   across projects.
2. `<VibeTraining-repo>/.claude/skills/new-rust-crate/` — shared with
   VibeYoga hackathon participants.

Both copies are kept in sync manually (not symlinked) so the
hackathon copy can diverge if participants need a different baseline.

## Skill directory layout

```
new-rust-crate/
├── SKILL.md                       # the playbook this document describes
├── templates/
│   ├── Cargo.toml.tmpl
│   ├── rustfmt.toml
│   ├── codecov.yml
│   ├── Makefile.tmpl
│   ├── README.md.tmpl
│   ├── book.toml.tmpl
│   ├── .gitignore
│   ├── .editorconfig
│   ├── LICENSE-MIT.tmpl
│   ├── LICENSE-Apache-2.0.tmpl
│   ├── AGENTS.md
│   ├── .claude/CLAUDE.md.tmpl
│   ├── .github/workflows/{ci,docs,release}.yml.tmpl
│   ├── docs/src/
│   │   ├── SUMMARY.md
│   │   └── introduction.md.tmpl
│   ├── src/lib.rs.tmpl            # empty public surface + one smoke test
│   ├── tests/main.rs.tmpl
│   ├── examples/starter.rs.tmpl
│   └── cli/
│       ├── Cargo.toml.tmpl
│       └── src/main.rs.tmpl       # clap-derive stub with one subcommand
└── references/
    └── design-notes.md            # what was borrowed from problemreductions and why
```

All templates use `{{PLACEHOLDER}}` tokens substituted at generation time.

## Placeholders

| Token | Source |
|-------|--------|
| `{{CRATE_NAME}}` | wizard, validated against `^[a-z][a-z0-9_-]*$` |
| `{{CRATE_DESCRIPTION}}` | wizard |
| `{{AUTHOR_NAME}}` | default from `git config user.name`, editable |
| `{{AUTHOR_EMAIL}}` | default from `git config user.email`, editable |
| `{{LICENSE}}` | wizard (MIT / Apache-2.0 / MIT OR Apache-2.0 / none) |
| `{{CLI_BIN_NAME}}` | wizard 3-option pick; omitted if CLI disabled |
| `{{YEAR}}` | current year |
| `{{GH_OWNER}}` | inferred from `gh api user -q .login`; editable |
| `{{GH_REPO}}` | defaults to `{{CRATE_NAME}}` |

Substitution via `sed -e 's/{{FOO}}/value/g'` during the copy pass.
Values are shell-escaped before substitution to block injection from
odd but possible inputs (e.g. author names with `&` or `/`).

## Flow

```
Step 0  Preflight
        For each of: cargo, rustfmt, clippy, mdbook, cargo-llvm-cov, gh, typst*
          - if present: skip
          - if missing: show install command (rustup component / cargo install / brew)
            and ask user to approve before running it
        *typst only checked if mdBook diagrams are desired; purely informational

Step 1  Identity (one batched question)
        - crate name
        - one-line description
        - author name / email (defaults from git config)
        - license choice
        - CLI binary naming: 3 options shown
            (a) <crate-name>-cli
            (b) short nickname — prompt for one; suggests first-syllable guess
            (c) plain <crate-name>
          skipped entirely if the user unchecks the CLI component in Step 2

Step 2  Component review
        Show the default-on list (Cargo/workspace, Makefile, rustfmt,
        README, CI, docs workflow, release workflow, codecov, mdBook,
        .claude/CLAUDE.md, AGENTS.md, LICENSE, .gitignore, .editorconfig,
        src/lib.rs, tests, examples, CLI subcrate).
        Let user uncheck by number. Recompute dependencies:
          - docs workflow requires mdBook (unchecking mdBook auto-unchecks docs workflow)
          - release workflow works standalone
          - codecov requires CI workflow

Step 3  GitHub bootstrap (two questions)
        - "git init + initial commit?" (default yes)
        - "also create a remote with `gh repo create`?" (default no;
          if yes, ask public/private and push after final commit)

Step 4  Scaffold
        - Abort if $(pwd)/<crate-name>/ already exists
        - mkdir <crate-name>, cd into it
        - Copy+substitute each selected template file
        - If CLI enabled: create the CLI subcrate at <crate-name>-cli/

Step 5  Verify
        Run in sequence:
          cargo build
          cargo test
          cargo clippy -- -D warnings
          cargo fmt --all -- --check
        If any fails: print the failure, halt before git commit, leave the
        tree on disk so the user can inspect or ask the skill to retry.

Step 6  Setup guides (only for enabled integrations)
        Print actionable, copy-pasteable next-step instructions:
          - crates.io: "gh secret set CARGO_REGISTRY_TOKEN" with the URL to mint a token
          - codecov: URL to link the repo, where to find CODECOV_TOKEN, gh secret command
          - GitHub Pages: the UI path to switch Pages source to "GitHub Actions"
                          (or the equivalent `gh api` call if available)
          - mdBook: `make mdbook` to preview locally

Step 7  Extras prompt
        "Add any of the following now? (multiple allowed, skip with 'n')"
          - criterion benches (benches/<name>_bench.rs + Cargo entry + Makefile target)
          - proc-macro subcrate (<name>-macros/ with passthrough derive stub)
          - issue templates (.github/ISSUE_TEMPLATE/{bug,feature}.yml)
        Each generates into the existing tree without re-running verify.
        A final `cargo build` runs if any extra was added.

Step 8  Final commit (if git init was accepted)
        git add -A
        git commit -m "chore: scaffold {{CRATE_NAME}}"
        If `gh repo create` was accepted: create and push.
        Report the final tree, the green/red status of each verify step,
        and any pending setup guides the user still needs to act on.
```

## Default-on components

Preselected in Step 2 (user can uncheck):

- Library crate (`src/lib.rs`, `tests/main.rs`) — not user-unselectable
- CLI subcrate
- Makefile
- `rustfmt.toml`
- `codecov.yml`
- README with badges (CI, docs, crates.io, codecov)
- `.gitignore`, `.editorconfig`
- LICENSE
- Examples (`examples/starter.rs`)
- mdBook docs (`book.toml`, `docs/src/`)
- `.claude/CLAUDE.md` stub + `AGENTS.md` redirect
- `.github/workflows/ci.yml`
- `.github/workflows/docs.yml` (depends on mdBook)
- `.github/workflows/release.yml`
- Codecov wiring in CI

Deferred to Step 7:

- Criterion benches
- Proc-macro subcrate
- Issue templates

Explicitly **not** included (author declined):

- CHANGELOG.md
- Dependabot config
- Git pre-commit hook
- `tests/suites/` layout

## Placeholder content

- `src/lib.rs`:
  ```rust
  //! {{CRATE_DESCRIPTION}}

  #[cfg(test)]
  mod tests {
      #[test]
      fn smoke() {
          assert_eq!(2 + 2, 4);
      }
  }
  ```
- `tests/main.rs`: one passing assertion so the integration test binary exists.
- `examples/starter.rs`: `fn main() { println!("hello from {{CRATE_NAME}}"); }`
- CLI `src/main.rs`: clap-derive with a single `hello` subcommand,
  `anyhow::Result` return, colored output via `owo-colors`.

## Cargo.toml shape

Root workspace Cargo.toml (only when CLI or proc-macro subcrate is
selected; otherwise a single-crate manifest):

The CLI subcrate directory is always `<crate-name>-cli/` regardless of
which binary-name option the user picks — only the `[[bin]] name` inside
that subcrate's Cargo.toml changes.

```toml
[workspace]
members = [".", "{{CRATE_NAME}}-cli"]

[package]
name = "{{CRATE_NAME}}"
version = "0.1.0"
edition = "2021"
description = "{{CRATE_DESCRIPTION}}"
license = "{{LICENSE}}"
repository = "https://github.com/{{GH_OWNER}}/{{GH_REPO}}"
authors = ["{{AUTHOR_NAME}} <{{AUTHOR_EMAIL}}>"]

[dependencies]

[dev-dependencies]

[profile.release]
lto = true
codegen-units = 1
strip = true
```

CLI subcrate manifest mirrors `problemreductions-cli/Cargo.toml`:
`clap` with derive, `anyhow`, `serde`, `serde_json`, `owo-colors`,
`clap_complete` for shell completion.

## Safety & verification

- Abort on existing target directory; never overwrite.
- Never call destructive git (`reset --hard`, `push --force`).
- Never skip hooks (`--no-verify`) or sign-off (`--no-gpg-sign`).
- Never claim completion without running `cargo build / test / clippy /
  fmt --check` and reporting results.
- Tool installation requires explicit per-tool user approval.
- No automatic `gh repo create` — always opt-in with a second confirmation
  once the user sees the planned owner/name.

## Success criteria

The skill is complete when, starting from an empty directory and a clean
system:

1. A new crate is created in `$(pwd)/<name>/` with all selected components.
2. `cargo build`, `cargo test`, `cargo clippy -- -D warnings`,
   and `cargo fmt --all -- --check` all pass against the generated tree.
3. `git log -1` shows the scaffold commit (if git was enabled).
4. If `gh repo create` was accepted, the remote exists and `main` has been pushed.
5. The user sees a final report listing every enabled integration and the
   exact manual next step required to activate it (tokens, Pages config).

## Open questions

None — all design decisions have been resolved in brainstorming.

## Out of scope

- Updating existing crates (this is a scaffolder, not a migrator).
- Publishing the first release — the user runs `make release V=0.1.0`
  after verifying the tree.
- Non-Rust projects.
- Supporting editions other than 2021 by default (user can edit
  `Cargo.toml` afterward).
