# `new-rust-crate` Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Claude Code skill that scaffolds a new Rust package (workspace with library + CLI subcrate, Makefile, mdBook, CI/docs/release workflows, codecov, `.claude/CLAUDE.md` stub) matching the conventions of `~/rcode/problemreductions`.

**Architecture:** The skill is pure-markdown logic at `~/.claude/skills/new-rust-crate/SKILL.md` driving a bundled `templates/` tree of parameterized source files. Claude, when the skill is invoked, runs an 8-step wizard (preflight → identity → components → github → scaffold → verify → setup guides → extras → commit), using `sed`-based placeholder substitution on the templates to produce the new crate under `$(pwd)/<crate-name>/`. A mirror copy of the skill lives in `<VibeTraining-repo>/.claude/skills/new-rust-crate/` for hackathon participants.

**Tech Stack:** Markdown (SKILL.md), Bash (sed, git, gh, cargo), reference: `problemreductions` v0.5.0. No executable code in the skill itself — the skill's contents are instructions that Claude executes with its existing tools.

**Spec:** `docs/superpowers/specs/2026-04-20-new-rust-crate-skill-design.md`

---

## File Structure

The skill itself:

```
~/.claude/skills/new-rust-crate/
├── SKILL.md                                           # ~600 lines, 8-step wizard
├── references/
│   └── design-notes.md                                # what was borrowed from problemreductions
└── templates/
    ├── Cargo.toml.tmpl                                # root workspace + lib manifest
    ├── rustfmt.toml                                   # verbatim from problemreductions
    ├── codecov.yml                                    # 95% target, ignores CLI/macros
    ├── Makefile.tmpl                                  # build/test/fmt/clippy/doc/check/coverage/release/cli
    ├── README.md.tmpl                                 # badges + quick-start
    ├── book.toml.tmpl                                 # mdBook config
    ├── .gitignore                                     # target, book, Cargo.lock policy
    ├── .editorconfig                                  # 4-space, LF, trim trailing ws
    ├── LICENSE-MIT.tmpl
    ├── LICENSE-Apache-2.0.tmpl
    ├── AGENTS.md                                      # static redirect stub
    ├── .claude/CLAUDE.md.tmpl                         # agent instructions skeleton
    ├── .github/workflows/ci.yml.tmpl
    ├── .github/workflows/docs.yml.tmpl
    ├── .github/workflows/release.yml.tmpl
    ├── docs/src/SUMMARY.md                            # mdBook TOC
    ├── docs/src/introduction.md.tmpl                  # first chapter
    ├── src/lib.rs.tmpl                                # crate doc-comment + smoke test
    ├── tests/main.rs.tmpl                             # integration test placeholder
    ├── examples/starter.rs.tmpl
    └── cli/
        ├── Cargo.toml.tmpl                            # CLI subcrate manifest
        └── src/main.rs.tmpl                           # clap-derive with one subcommand
```

Mirror tree written to `<VibeTraining-repo>/.claude/skills/new-rust-crate/` in Task 17.

### Placeholder token set

All `.tmpl` files use these tokens — substituted in-flight during scaffold by a shell helper defined in SKILL.md:

| Token | Example value |
|-------|---------------|
| `{{CRATE_NAME}}` | `pagerank` |
| `{{CRATE_DESCRIPTION}}` | `Graph PageRank algorithm` |
| `{{AUTHOR_NAME}}` | `Jinguo Liu` |
| `{{AUTHOR_EMAIL}}` | `cacate0129@gmail.com` |
| `{{LICENSE}}` | `MIT` |
| `{{LICENSE_SPDX}}` | `MIT OR Apache-2.0` (for mixed licenses) |
| `{{CLI_BIN_NAME}}` | `pr` (or `<name>-cli`, or `<name>`) |
| `{{YEAR}}` | `2026` |
| `{{GH_OWNER}}` | `GiggleLiu` |
| `{{GH_REPO}}` | `pagerank` |

---

## Tasks

### Task 1: Scaffold the skill skeleton

**Files:**
- Create: `~/.claude/skills/new-rust-crate/SKILL.md`
- Create: `~/.claude/skills/new-rust-crate/templates/` (empty directory)
- Create: `~/.claude/skills/new-rust-crate/references/` (empty directory)

- [ ] **Step 1: Create directory tree**

Run:
```bash
mkdir -p ~/.claude/skills/new-rust-crate/templates/.github/workflows
mkdir -p ~/.claude/skills/new-rust-crate/templates/.claude
mkdir -p ~/.claude/skills/new-rust-crate/templates/docs/src
mkdir -p ~/.claude/skills/new-rust-crate/templates/src
mkdir -p ~/.claude/skills/new-rust-crate/templates/tests
mkdir -p ~/.claude/skills/new-rust-crate/templates/examples
mkdir -p ~/.claude/skills/new-rust-crate/templates/cli/src
mkdir -p ~/.claude/skills/new-rust-crate/references
```

- [ ] **Step 2: Write minimal SKILL.md stub so the skill registers**

Write `~/.claude/skills/new-rust-crate/SKILL.md`:

```markdown
---
name: new-rust-crate
description: Use when the user wants to create a new Rust crate or package — scaffolds a library (optionally with CLI subcrate), Makefile, mdBook docs, GitHub Actions (CI/docs/release), codecov wiring, and `.claude/CLAUDE.md` stub into `$(pwd)/<crate-name>/`. Trigger phrases include "new rust crate", "scaffold a rust package", "create a rust library", "新建 rust 项目".
---

# new-rust-crate

Stub — replaced in later tasks with the full playbook.
```

- [ ] **Step 3: Verify the skill directory layout**

Run:
```bash
ls ~/.claude/skills/new-rust-crate/
```
Expected output contains: `SKILL.md  references  templates`

- [ ] **Step 4: Commit**

The skill lives outside any git repo. Commit happens in Task 17 against the mirror in the VibeTraining repo.

---

### Task 2: Write core project templates (Cargo.toml, rustfmt, .gitignore, .editorconfig)

**Files:**
- Create: `~/.claude/skills/new-rust-crate/templates/Cargo.toml.tmpl`
- Create: `~/.claude/skills/new-rust-crate/templates/rustfmt.toml`
- Create: `~/.claude/skills/new-rust-crate/templates/.gitignore`
- Create: `~/.claude/skills/new-rust-crate/templates/.editorconfig`

- [ ] **Step 1: Write `Cargo.toml.tmpl`** — workspace-style root manifest that degrades to single-crate when CLI is unchecked. SKILL.md will edit the `[workspace]` block out at scaffold time if `CLI=no`.

Write `~/.claude/skills/new-rust-crate/templates/Cargo.toml.tmpl`:

```toml
[workspace]
members = [".", "{{CRATE_NAME}}-cli"]
resolver = "2"

[package]
name = "{{CRATE_NAME}}"
version = "0.1.0"
edition = "2021"
description = "{{CRATE_DESCRIPTION}}"
license = "{{LICENSE_SPDX}}"
repository = "https://github.com/{{GH_OWNER}}/{{GH_REPO}}"
authors = ["{{AUTHOR_NAME}} <{{AUTHOR_EMAIL}}>"]
keywords = []
categories = []

[dependencies]

[dev-dependencies]

[profile.release]
lto = true
codegen-units = 1
strip = true
```

- [ ] **Step 2: Write `rustfmt.toml`** — copied verbatim from `~/rcode/problemreductions/rustfmt.toml`, no placeholders.

Write `~/.claude/skills/new-rust-crate/templates/rustfmt.toml`:

```toml
# Rust formatting configuration
# See: https://rust-lang.github.io/rustfmt/

edition = "2021"
max_width = 100
tab_spaces = 4
newline_style = "Unix"
use_small_heuristics = "Default"

reorder_imports = true

use_field_init_shorthand = true
use_try_shorthand = true
force_explicit_abi = true
```

- [ ] **Step 3: Write `.gitignore`**

Write `~/.claude/skills/new-rust-crate/templates/.gitignore`:

```
/target
/book
Cargo.lock.bak
*.pdb
.DS_Store

# Editor
.vscode/
.idea/
*.swp
*.swo

# Coverage
lcov.info
tarpaulin-report.*
*.profraw

# mdBook
/docs/book
```

- [ ] **Step 4: Write `.editorconfig`**

Write `~/.claude/skills/new-rust-crate/templates/.editorconfig`:

```
root = true

[*]
charset = utf-8
end_of_line = lf
insert_final_newline = true
trim_trailing_whitespace = true
indent_style = space
indent_size = 4

[*.{yml,yaml,toml,md}]
indent_size = 2

[Makefile]
indent_style = tab
```

- [ ] **Step 5: Verify files exist and are non-empty**

Run:
```bash
for f in Cargo.toml.tmpl rustfmt.toml .gitignore .editorconfig; do
  wc -l ~/.claude/skills/new-rust-crate/templates/$f
done
```
Expected: all four print line counts > 0.

- [ ] **Step 6: Commit** — skill still outside git; batched at Task 17.

---

### Task 3: Write library source templates (src/lib.rs, tests/main.rs, examples/starter.rs)

**Files:**
- Create: `~/.claude/skills/new-rust-crate/templates/src/lib.rs.tmpl`
- Create: `~/.claude/skills/new-rust-crate/templates/tests/main.rs.tmpl`
- Create: `~/.claude/skills/new-rust-crate/templates/examples/starter.rs.tmpl`

- [ ] **Step 1: Write `src/lib.rs.tmpl`**

```rust
//! {{CRATE_DESCRIPTION}}
//!
//! This crate is a scaffold produced by the `new-rust-crate` skill.
//! Replace this doc comment with an overview of what the crate does.

#[cfg(test)]
mod tests {
    #[test]
    fn smoke() {
        assert_eq!(2 + 2, 4);
    }
}
```

- [ ] **Step 2: Write `tests/main.rs.tmpl`**

```rust
//! Integration tests for {{CRATE_NAME}}.

#[test]
fn integration_smoke() {
    // Replace with a real cross-module test once the crate has public API.
    assert!(true);
}
```

- [ ] **Step 3: Write `examples/starter.rs.tmpl`**

```rust
//! Minimal example. Run with `cargo run --example starter`.

fn main() {
    println!("hello from {{CRATE_NAME}}");
}
```

- [ ] **Step 4: Verify no stray placeholders**

Run:
```bash
grep -l '{{' ~/.claude/skills/new-rust-crate/templates/src/*.tmpl \
  ~/.claude/skills/new-rust-crate/templates/tests/*.tmpl \
  ~/.claude/skills/new-rust-crate/templates/examples/*.tmpl
```
Expected: prints the three template paths (they all contain `{{CRATE_*}}` tokens, which is correct).

---

### Task 4: Write CLI subcrate templates

**Files:**
- Create: `~/.claude/skills/new-rust-crate/templates/cli/Cargo.toml.tmpl`
- Create: `~/.claude/skills/new-rust-crate/templates/cli/src/main.rs.tmpl`

- [ ] **Step 1: Write `cli/Cargo.toml.tmpl`**

```toml
[package]
name = "{{CRATE_NAME}}-cli"
version = "0.1.0"
edition = "2021"
description = "CLI for {{CRATE_NAME}}"
license = "{{LICENSE_SPDX}}"
repository = "https://github.com/{{GH_OWNER}}/{{GH_REPO}}"

[[bin]]
name = "{{CLI_BIN_NAME}}"
path = "src/main.rs"

[dependencies]
{{CRATE_NAME}} = { version = "0.1.0", path = ".." }
clap = { version = "4", features = ["derive"] }
clap_complete = "4"
anyhow = "1"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
owo-colors = { version = "4", features = ["supports-colors"] }
```

- [ ] **Step 2: Write `cli/src/main.rs.tmpl`**

```rust
//! {{CRATE_NAME}} CLI.

use clap::{Parser, Subcommand};

#[derive(Parser)]
#[command(name = "{{CLI_BIN_NAME}}", version, about = "{{CRATE_DESCRIPTION}}")]
struct Cli {
    #[command(subcommand)]
    command: Commands,
}

#[derive(Subcommand)]
enum Commands {
    /// Print a greeting and exit.
    Hello {
        /// Name to greet.
        #[arg(default_value = "world")]
        name: String,
    },
}

fn main() -> anyhow::Result<()> {
    let cli = Cli::parse();
    match cli.command {
        Commands::Hello { name } => {
            println!("hello, {name}");
            Ok(())
        }
    }
}
```

- [ ] **Step 3: Verify**

Run:
```bash
cat ~/.claude/skills/new-rust-crate/templates/cli/Cargo.toml.tmpl | head -5
cat ~/.claude/skills/new-rust-crate/templates/cli/src/main.rs.tmpl | head -5
```
Expected: the template headers print, confirming both files exist.

---

### Task 5: Write Makefile template

**Files:**
- Create: `~/.claude/skills/new-rust-crate/templates/Makefile.tmpl`

- [ ] **Step 1: Write `Makefile.tmpl`** — a pruned descendant of `problemreductions/Makefile` keeping only generic targets.

```make
# Makefile for {{CRATE_NAME}}

.PHONY: help build test fmt fmt-check clippy doc mdbook coverage clean check release cli cli-run

# Default target
help:
	@echo "Available targets:"
	@echo "  build        - Build the project (workspace)"
	@echo "  test         - Run all tests"
	@echo "  fmt          - Format code with rustfmt"
	@echo "  fmt-check    - Check code formatting"
	@echo "  clippy       - Run clippy lints (-D warnings)"
	@echo "  doc          - Build mdBook documentation"
	@echo "  mdbook       - Build and serve mdBook (live reload)"
	@echo "  coverage     - Generate coverage report (requires cargo-llvm-cov)"
	@echo "  check        - Quick pre-commit check (fmt-check + clippy + test)"
	@echo "  clean        - Clean build artifacts"
	@echo "  cli          - Build the {{CLI_BIN_NAME}} CLI"
	@echo "  cli-run      - Run the {{CLI_BIN_NAME}} CLI with ARGS (e.g. make cli-run ARGS='hello')"
	@echo "  release V=x.y.z - Tag vX.Y.Z and push (triggers release workflow)"

build:
	cargo build --workspace

test:
	cargo test --workspace

fmt:
	cargo fmt --all

fmt-check:
	cargo fmt --all -- --check

clippy:
	cargo clippy --workspace --all-targets -- -D warnings

doc:
	mdbook build

mdbook:
	mdbook serve --open

coverage:
	cargo llvm-cov --workspace --lcov --output-path lcov.info

clean:
	cargo clean
	rm -rf book lcov.info

check: fmt-check clippy test

cli:
	cargo build -p {{CRATE_NAME}}-cli --release

cli-run:
	cargo run -p {{CRATE_NAME}}-cli -- $(ARGS)

release:
ifndef V
	$(error V is required. Usage: make release V=0.1.0)
endif
	git tag -a v$(V) -m "Release v$(V)"
	git push origin v$(V)
```

- [ ] **Step 2: Verify it parses**

Run:
```bash
make -nf ~/.claude/skills/new-rust-crate/templates/Makefile.tmpl help 2>&1 | head
```
Expected: prints the help-target echo lines without errors (make's dry-run mode accepts tokens like `{{CRATE_NAME}}`).

---

### Task 6: Write README template

**Files:**
- Create: `~/.claude/skills/new-rust-crate/templates/README.md.tmpl`

- [ ] **Step 1: Write `README.md.tmpl`**

````markdown
# {{CRATE_NAME}}

[![CI](https://github.com/{{GH_OWNER}}/{{GH_REPO}}/actions/workflows/ci.yml/badge.svg)](https://github.com/{{GH_OWNER}}/{{GH_REPO}}/actions/workflows/ci.yml)
[![Docs](https://github.com/{{GH_OWNER}}/{{GH_REPO}}/actions/workflows/docs.yml/badge.svg)](https://{{GH_OWNER}}.github.io/{{GH_REPO}}/)
[![Crates.io](https://img.shields.io/crates/v/{{CRATE_NAME}}.svg)](https://crates.io/crates/{{CRATE_NAME}})
[![Codecov](https://codecov.io/gh/{{GH_OWNER}}/{{GH_REPO}}/branch/main/graph/badge.svg)](https://codecov.io/gh/{{GH_OWNER}}/{{GH_REPO}})
[![License](https://img.shields.io/badge/license-{{LICENSE}}-blue.svg)](./LICENSE)

{{CRATE_DESCRIPTION}}

## Quick start

```toml
[dependencies]
{{CRATE_NAME}} = "0.1"
```

```rust
use {{CRATE_NAME}};
// TODO: example once public API lands
```

## CLI

```bash
cargo install {{CRATE_NAME}}-cli
{{CLI_BIN_NAME}} hello world
```

## Development

```bash
make check       # fmt-check + clippy + test
make mdbook      # preview docs locally at http://localhost:3000
make coverage    # generate lcov.info
```

## License

Licensed under {{LICENSE_SPDX}}.
````

- [ ] **Step 2: Verify**

Run:
```bash
grep -c '{{' ~/.claude/skills/new-rust-crate/templates/README.md.tmpl
```
Expected: prints a number ≥ 10 (many tokens present, all intentional).

---

### Task 7: Write mdBook templates

**Files:**
- Create: `~/.claude/skills/new-rust-crate/templates/book.toml.tmpl`
- Create: `~/.claude/skills/new-rust-crate/templates/docs/src/SUMMARY.md`
- Create: `~/.claude/skills/new-rust-crate/templates/docs/src/introduction.md.tmpl`

- [ ] **Step 1: Write `book.toml.tmpl`**

```toml
[book]
title = "{{CRATE_NAME}}"
authors = ["{{AUTHOR_NAME}}"]
description = "{{CRATE_DESCRIPTION}}"
language = "en"
src = "docs/src"

[output.html]
default-theme = "navy"
git-repository-url = "https://github.com/{{GH_OWNER}}/{{GH_REPO}}"
edit-url-template = "https://github.com/{{GH_OWNER}}/{{GH_REPO}}/edit/main/{path}"

[output.html.fold]
enable = true
level = 1

[output.html.search]
enable = true
```

- [ ] **Step 2: Write `docs/src/SUMMARY.md`** — verbatim, no placeholders.

```markdown
# Summary

- [Introduction](./introduction.md)
```

- [ ] **Step 3: Write `docs/src/introduction.md.tmpl`**

```markdown
# Introduction

{{CRATE_DESCRIPTION}}

This book documents the `{{CRATE_NAME}}` crate. Replace this page with an
overview of the problem your crate solves, a worked example, and links to
deeper chapters.
```

---

### Task 8: Write GitHub workflow templates (ci, docs, release)

**Files:**
- Create: `~/.claude/skills/new-rust-crate/templates/.github/workflows/ci.yml.tmpl`
- Create: `~/.claude/skills/new-rust-crate/templates/.github/workflows/docs.yml.tmpl`
- Create: `~/.claude/skills/new-rust-crate/templates/.github/workflows/release.yml.tmpl`

- [ ] **Step 1: Write `ci.yml.tmpl`** — fmt-check + clippy + test + coverage.

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  CARGO_TERM_COLOR: always
  RUST_BACKTRACE: 1

jobs:
  fmt:
    name: Format
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: rustfmt
      - run: cargo fmt --all -- --check

  clippy:
    name: Clippy
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: clippy
      - uses: Swatinem/rust-cache@v2
      - run: cargo clippy --workspace --all-targets -- -D warnings

  test:
    name: Test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2
      - run: cargo build --workspace --verbose
      - run: cargo test --workspace --verbose
      - run: cargo test --doc --verbose

  coverage:
    name: Coverage
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: llvm-tools-preview
      - uses: Swatinem/rust-cache@v2
      - uses: taiki-e/install-action@cargo-llvm-cov
      - run: cargo llvm-cov --workspace --lcov --output-path lcov.info
      - uses: codecov/codecov-action@v5
        with:
          files: lcov.info
          fail_ci_if_error: false
          token: ${{ secrets.CODECOV_TOKEN }}
```

Note: the skill's `sed` pass only substitutes tokens matching `{{UPPERCASE_UNDERSCORE}}`. GitHub Actions expressions like `${{ secrets.CODECOV_TOKEN }}` begin with `$` and contain a space and lowercase identifier, so they do not match the placeholder pattern and pass through unchanged. Keep the literal `${{ secrets.CODECOV_TOKEN }}` exactly as shown.

- [ ] **Step 2: Write `docs.yml.tmpl`** — mdBook → gh-pages.

```yaml
name: Docs

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - name: Install mdBook
        run: |
          mkdir -p "$HOME/bin"
          curl -sSL https://github.com/rust-lang/mdBook/releases/download/v0.4.37/mdbook-v0.4.37-x86_64-unknown-linux-gnu.tar.gz \
            | tar -xz -C "$HOME/bin"
          echo "$HOME/bin" >> $GITHUB_PATH
      - name: Build mdBook
        run: mdbook build
      - name: Build rustdoc
        run: RUSTDOCFLAGS="--default-theme=dark" cargo doc --no-deps
      - name: Combine docs
        run: |
          mkdir -p book/api
          cp -r target/doc/* book/api/
      - uses: actions/configure-pages@v4
      - uses: actions/upload-pages-artifact@v3
        with:
          path: ./book

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - id: deployment
        uses: actions/deploy-pages@v4
```

- [ ] **Step 3: Write `release.yml.tmpl`** — tag-triggered crates.io publish.

```yaml
name: Release

on:
  push:
    tags:
      - 'v*.*.*'

permissions:
  contents: write

env:
  CARGO_TERM_COLOR: always

jobs:
  create-release:
    name: Create GitHub Release
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: softprops/action-gh-release@v1
        with:
          draft: false
          prerelease: false
          generate_release_notes: true

  publish-crate:
    name: Publish to crates.io
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2
      - name: Publish main crate
        run: cargo publish --token ${{ secrets.CARGO_REGISTRY_TOKEN }}
      - name: Wait for main crate to be indexed
        run: sleep 30
      - name: Publish CLI crate
        run: cargo publish --manifest-path {{CRATE_NAME}}-cli/Cargo.toml --token ${{ secrets.CARGO_REGISTRY_TOKEN }}
```

- [ ] **Step 4: Verify workflows are syntactically valid**

Run:
```bash
for f in ~/.claude/skills/new-rust-crate/templates/.github/workflows/*.yml.tmpl; do
  python3 -c "import yaml, sys; yaml.safe_load(open('$f'))" && echo "OK: $f"
done
```
Expected: prints `OK: <path>` for all three files. (YAML parser ignores `{{...}}` tokens as they appear only inside string values.)

---

### Task 9: Write codecov config

**Files:**
- Create: `~/.claude/skills/new-rust-crate/templates/codecov.yml`

- [ ] **Step 1: Write `codecov.yml`** — no placeholders needed.

```yaml
# Codecov configuration
# https://docs.codecov.com/docs/codecov-yaml

coverage:
  precision: 2
  round: down
  range: "80...100"

  status:
    project:
      default:
        target: 80%
        threshold: 2%
    patch:
      default:
        target: 80%
        threshold: 2%

comment:
  layout: "reach, diff, flags, files"
  behavior: default
  require_changes: false
```

Note: target is 80% (vs. problemreductions' 95%). Fresh crates rarely hit 95% day one, and failing the coverage gate on every early PR is noise. User can raise it later.

---

### Task 10: Write `.claude/CLAUDE.md` and AGENTS.md templates

**Files:**
- Create: `~/.claude/skills/new-rust-crate/templates/.claude/CLAUDE.md.tmpl`
- Create: `~/.claude/skills/new-rust-crate/templates/AGENTS.md`

- [ ] **Step 1: Write `.claude/CLAUDE.md.tmpl`**

````markdown
# CLAUDE.md

## Project Overview
{{CRATE_DESCRIPTION}}

## Philosophy
- **Simple logic, maximum reuse.** Prefer straightforward code with fewer branches.
  Reuse existing logic rather than adding ad-hoc special cases.
- **Root-cause fixes over patches.** When a bug surfaces, trace it to its origin.
- **Tests over implementation.** Spend more time designing tests than implementing code.

## Commands
```bash
make help        # List all Makefile targets
make build       # Build workspace
make test        # Run all tests
make check       # fmt-check + clippy + test (pre-commit gate)
make cli         # Build the {{CLI_BIN_NAME}} CLI (release)
make mdbook      # Serve docs locally
make coverage    # Generate lcov.info
make release V=x.y.z  # Tag and push, triggers release workflow
```

## Git Safety
- **NEVER force push** to main (`--force`, `-f`, `--force-with-lease`).
- Never skip hooks (`--no-verify`) or bypass signing unless explicitly asked.
- Always create new commits instead of amending published ones.

## Architecture
Fill in once the crate takes shape. Describe the main modules, what each is
responsible for, and how data flows between them.
````

- [ ] **Step 2: Write `AGENTS.md`** — static, no placeholders.

```markdown
# AGENTS.md

The canonical agent instructions for this repository live in
[`./.claude/CLAUDE.md`](./.claude/CLAUDE.md).

All agents should read and follow that file before making changes.

Do not duplicate or fork those instructions here.
Update `./.claude/CLAUDE.md` instead.
```

---

### Task 11: Write LICENSE templates

**Files:**
- Create: `~/.claude/skills/new-rust-crate/templates/LICENSE-MIT.tmpl`
- Create: `~/.claude/skills/new-rust-crate/templates/LICENSE-Apache-2.0.tmpl`

- [ ] **Step 1: Write `LICENSE-MIT.tmpl`** — canonical MIT with placeholders.

```
MIT License

Copyright (c) {{YEAR}} {{AUTHOR_NAME}}

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

- [ ] **Step 2: Write `LICENSE-Apache-2.0.tmpl`** — download full text into the skill directory once.

Run:
```bash
curl -sSL https://www.apache.org/licenses/LICENSE-2.0.txt \
  -o ~/.claude/skills/new-rust-crate/templates/LICENSE-Apache-2.0.tmpl
wc -l ~/.claude/skills/new-rust-crate/templates/LICENSE-Apache-2.0.tmpl
```
Expected: ≥ 200 lines. The Apache-2.0 text has no author field; it stands as-is. The NOTICE file (which does have author info) is left for the user to add if they want it.

---

### Task 12: Write references/design-notes.md

**Files:**
- Create: `~/.claude/skills/new-rust-crate/references/design-notes.md`

- [ ] **Step 1: Write `references/design-notes.md`**

```markdown
# design notes for new-rust-crate

## what was borrowed from problemreductions

| Area | Source | Adaptation |
|------|--------|------------|
| rustfmt.toml | verbatim copy | no changes |
| Cargo workspace shape | `[workspace] members = [".", "<name>-cli"]` | dropped the `<name>-macros` member; deferred to Step 7 extras |
| Makefile targets | build/test/fmt/clippy/doc/check/coverage/release/cli | dropped problemreductions-specific targets (mcp-test, paper, rust-export, qubo-testdata, pipeline, board, papers, run-plan/issue/review) |
| CI workflow | fmt/clippy/test/coverage jobs | removed ilp-highs features; removed typst/paper build |
| Docs workflow | mdBook → gh-pages | removed typst PDF, removed problem-specific `export_*` cargo-run steps |
| Release workflow | tag-triggered crates.io publish | removed macro crate publish |
| codecov.yml | 95% target → 80% | lowered threshold so fresh crates don't fail coverage gate |
| `.claude/CLAUDE.md` stub | philosophy + commands + git safety | removed project-specific sections (skills, papers, codex compat) |
| AGENTS.md | verbatim redirect to .claude/CLAUDE.md | no changes |

## what was deliberately NOT borrowed

- The `.claude/skills/*` tree (~20 skills) — problem-reductions specific domain logic.
- Proc-macro subcrate — most crates don't need it; promoted to Step 7 extras.
- Benches (criterion) — premature optimisation; promoted to Step 7 extras.
- Typst paper pipeline — academic-paper specific.
- Julia parity / QUBO test data scripts — problem-reductions specific.
- CHANGELOG, Dependabot, pre-commit hook, tests/suites/ layout — author declined.
- Rich feature flags (`ilp-highs`, `example-db`, `mcp`) — domain-specific.

## placeholder substitution policy

Only tokens matching `{{[A-Z][A-Z0-9_]*}}` are substituted.
GitHub Actions expressions (`${{ secrets.FOO }}`, `${{ steps.bar }}`) contain
`{{` but the interior starts with a space or letter in non-matching case, so
they pass through unchanged. The skill's `sed` pipeline uses explicit
per-token replacements; there is no generic substitute-everything-in-braces
rule.

## divergence log

Track here any time the reference problemreductions conventions change in a
way we want to mirror.
```

---

### Task 13: Write SKILL.md — frontmatter, overview, Step 0 (preflight)

**Files:**
- Modify: `~/.claude/skills/new-rust-crate/SKILL.md` (overwrite stub from Task 1)

- [ ] **Step 1: Replace the stub with the full frontmatter + overview + Step 0**

Write `~/.claude/skills/new-rust-crate/SKILL.md`:

````markdown
---
name: new-rust-crate
description: Use when the user wants to create a new Rust crate or package — scaffolds a library (optionally with CLI subcrate), Makefile, mdBook docs, GitHub Actions (CI/docs/release), codecov wiring, and `.claude/CLAUDE.md` stub into `$(pwd)/<crate-name>/`. Trigger phrases include "new rust crate", "scaffold a rust package", "create a rust library", "new rust package", "新建 rust 项目", "新建 rust 包".
---

# new-rust-crate

Scaffold a Rust package that mirrors the conventions of the `problemreductions` reference: workspace + library + optional CLI subcrate, Makefile-driven workflow, mdBook, GitHub Actions (CI/docs/release), codecov wiring, `.claude/CLAUDE.md` stub.

## When to use

Trigger on: "new rust crate", "scaffold a rust package", "new rust library", "create rust project", and the Chinese equivalents. If the user is already inside a Rust project and wants to *add* a feature (CLI, benches, docs), decline — this skill creates a fresh package; adding pieces to an existing one is a separate job.

## When NOT to use

- User wants a single-file script — suggest `cargo script` or a `bin/` under an existing crate.
- User wants to migrate an existing crate — this skill only creates; never mutates an existing tree.
- User is not creating a Rust project — bail out immediately.

## Inputs this skill collects

Via the Step 1 wizard: crate name, description, author name/email (defaults from git config), license, CLI binary name (if CLI enabled).

Via Step 2: checkbox list of components to include (all default-on; user unchecks unwanted ones).

Via Step 3: whether to run `git init` and whether to create a remote with `gh repo create`.

## Output

`$(pwd)/<crate-name>/` populated with all selected components, passing `cargo build`, `cargo test`, `cargo clippy -- -D warnings`, `cargo fmt --all -- --check`, and with an initial git commit (if git was enabled).

---

## Step 0 — Preflight

Check for required and recommended tools. For each **required** tool that is missing, stop and tell the user how to install it — do not proceed. For each **recommended** tool that is missing, ask the user once whether to install it; accept yes/no/skip.

### Required tools

| Tool | Check | Install command |
|------|-------|-----------------|
| cargo | `command -v cargo` | `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh` |
| git | `command -v git` | platform-specific; tell user to install |

If `cargo` is missing, halt. Tell the user: "Install Rust first with rustup (https://rustup.rs), then re-run this skill."

### Recommended tools

Detect each; for each missing tool, ask the user before installing:

| Tool | Needed for | Install command |
|------|------------|-----------------|
| rustfmt | `make fmt-check`, CI fmt job | `rustup component add rustfmt` |
| clippy | `make clippy`, CI clippy job | `rustup component add clippy` |
| mdbook | `make mdbook` / `make doc`, docs workflow | `cargo install mdbook` |
| cargo-llvm-cov | `make coverage`, CI coverage job | `cargo install cargo-llvm-cov` |
| gh | Step 3 remote creation, setup guides | `brew install gh` (macOS) / see https://cli.github.com |

For each missing recommended tool, use `AskUserQuestion` (or inline prompt) with three options: **Install**, **Skip (disable dependent components)**, **Abort**.

If the user picks **Skip** for a tool, record that and in Step 2 auto-uncheck the components that depend on it (e.g., skipping mdbook auto-unchecks mdBook docs + docs workflow; skipping cargo-llvm-cov does not auto-uncheck coverage, but notes that `make coverage` will fail locally until installed).

### Environment sanity check

Also verify:

- `$(pwd)` is writable — `test -w .` — otherwise abort with an explanatory error.
- The current directory is not already a Rust crate — check for `Cargo.toml` in `$(pwd)`; if present, halt with: "Current directory is already a Rust project. Either `cd ..` first, or ask me to add components to the existing crate via a different skill."

---
````

- [ ] **Step 2: Verify frontmatter parses**

Run:
```bash
head -5 ~/.claude/skills/new-rust-crate/SKILL.md
```
Expected: shows the `---` frontmatter with `name` and `description` fields.

---

### Task 14: Write SKILL.md — Step 1 (identity) and Step 2 (components)

**Files:**
- Modify: `~/.claude/skills/new-rust-crate/SKILL.md` — append below the Step 0 section.

- [ ] **Step 1: Append Step 1 (identity wizard)**

Append to `~/.claude/skills/new-rust-crate/SKILL.md`:

````markdown

## Step 1 — Identity (batched question)

Prompt the user with a single structured message asking for all of the following. Pre-fill defaults in the prompt so the user can accept by saying "defaults" or "yes".

Required fields:

1. **Crate name** — must match `^[a-z][a-z0-9_-]*$`. If the input fails the regex, explain and re-ask. If the directory `$(pwd)/<name>/` already exists, halt with an error.
2. **One-line description** — free text. Will appear in Cargo.toml, README, book.toml, CLAUDE.md.
3. **Author name** — default from `git config --get user.name`; blank if unset, prompt until filled.
4. **Author email** — default from `git config --get user.email`; blank if unset, prompt until filled.
5. **License** — four options: `MIT`, `Apache-2.0`, `MIT OR Apache-2.0` (dual), `none`. Default `MIT OR Apache-2.0`. If `none`, no LICENSE file is written and Cargo.toml's `license = "..."` is replaced with the TOML value `license = ""` and a commented TODO.
6. **GitHub owner** — default from `gh api user -q .login` if gh is installed and authenticated; else prompt. Used only to fill URLs in README/Cargo.toml/book.toml; no network calls yet.
7. **GitHub repo name** — default `<crate-name>`; user can override if they want different upstream naming.

If the CLI component is still in the default-on set (user has not yet reached Step 2), also collect:

8. **CLI binary name** — present three options:
   - **(a) `<crate-name>-cli`** — long, explicit
   - **(b) Short nickname** — prompt for a short name; suggest a first-syllable guess (e.g., `pagerank` → `pr`, `problemreductions` → `pred`)
   - **(c) Plain `<crate-name>`** — binary shares the crate's name

   Default is (b) with the guess. Record as `{{CLI_BIN_NAME}}`.

After collecting, echo the filled values back and ask "Proceed with these? (yes/edit)". If `edit`, re-open whichever field the user names.

---

## Step 2 — Component review

Present the default-on components as a numbered checkbox list and invite the user to un-check any.

```
Components to include (all default-on):

 1. [x] Library crate (src/lib.rs, tests/)           [always on; not togglable]
 2. [x] CLI subcrate (<crate>-cli/)
 3. [x] Makefile
 4. [x] rustfmt.toml
 5. [x] README.md with badges
 6. [x] .gitignore, .editorconfig
 7. [x] LICENSE
 8. [x] examples/starter.rs
 9. [x] mdBook docs (book.toml, docs/src/)
10. [x] .github/workflows/ci.yml
11. [x] .github/workflows/docs.yml              (requires mdBook)
12. [x] .github/workflows/release.yml
13. [x] codecov.yml + coverage job in CI        (requires CI)
14. [x] .claude/CLAUDE.md + AGENTS.md stub

Reply with:
  - "defaults" to keep everything on
  - "skip 2 9 11" to un-check items 2, 9, and 11
  - "only 1 3 4" to keep only those items
```

Apply dependency rules after the user responds:

- If mdBook (#9) is off → force docs workflow (#11) off and warn the user.
- If CI (#10) is off → force codecov (#13) off and warn the user.
- If CLI (#2) is off → also clear the `{{CLI_BIN_NAME}}` value collected in Step 1 and skip CLI manifests + Makefile CLI targets.

Print the final component list back to the user and continue.

---
````

- [ ] **Step 2: Verify the appended sections are syntactically valid markdown**

Run:
```bash
grep -c '^## Step' ~/.claude/skills/new-rust-crate/SKILL.md
```
Expected: `3` (Step 0, Step 1, Step 2 headings so far).

---

### Task 15: Write SKILL.md — Step 3 (github) + Step 4 (scaffold) + Step 5 (verify)

**Files:**
- Modify: `~/.claude/skills/new-rust-crate/SKILL.md` — append below Step 2.

- [ ] **Step 1: Append Step 3 (github bootstrap)**

Append:

````markdown

## Step 3 — GitHub bootstrap

Ask two questions:

1. **Initialise git?** — default yes. If the user says yes, remember to run `git init` + initial commit at Step 8.
2. **Create a remote with `gh repo create`?** — default no. Only offered if `gh` is available and authenticated (`gh auth status` returns 0). If yes, ask:
   - public or private? (default public)
   - owner (default the `{{GH_OWNER}}` collected in Step 1)

Do not run `gh repo create` yet — defer to Step 8 so the user sees the scaffold first.

---

## Step 4 — Scaffold

### 4.1 Create target directory

Run:
```bash
mkdir "<crate-name>" && cd "<crate-name>"
```

If the directory already exists, halt (this was already validated in Step 1 but the check is idempotent here).

### 4.2 Copy templates with placeholder substitution

For each enabled component, copy the corresponding template file(s) from `~/.claude/skills/new-rust-crate/templates/` into the target tree, performing placeholder substitution on files whose name ends in `.tmpl` (strip the `.tmpl` suffix on copy). Files without `.tmpl` are copied byte-for-byte.

Define a helper function (inlined in the skill's Bash calls, not in a separate script file):

```bash
substitute() {
  # $1 = src path (inside skill's templates/)
  # $2 = dest path (inside new crate dir)
  sed \
    -e "s|{{CRATE_NAME}}|${CRATE_NAME}|g" \
    -e "s|{{CRATE_DESCRIPTION}}|${CRATE_DESCRIPTION}|g" \
    -e "s|{{AUTHOR_NAME}}|${AUTHOR_NAME}|g" \
    -e "s|{{AUTHOR_EMAIL}}|${AUTHOR_EMAIL}|g" \
    -e "s|{{LICENSE}}|${LICENSE}|g" \
    -e "s|{{LICENSE_SPDX}}|${LICENSE_SPDX}|g" \
    -e "s|{{CLI_BIN_NAME}}|${CLI_BIN_NAME}|g" \
    -e "s|{{YEAR}}|${YEAR}|g" \
    -e "s|{{GH_OWNER}}|${GH_OWNER}|g" \
    -e "s|{{GH_REPO}}|${GH_REPO}|g" \
    "$1" > "$2"
}
```

Shell-escape user-supplied values before running `sed`: escape `|`, `&`, `\`, and `/` in each placeholder value with a preceding backslash. This matters mostly for `{{CRATE_DESCRIPTION}}` which can contain punctuation.

Copy plan (only copy files for enabled components):

| Component | Source → Destination |
|-----------|----------------------|
| library (always) | `templates/Cargo.toml.tmpl` → `Cargo.toml` |
|  | `templates/src/lib.rs.tmpl` → `src/lib.rs` |
|  | `templates/tests/main.rs.tmpl` → `tests/main.rs` |
| rustfmt | `templates/rustfmt.toml` → `rustfmt.toml` |
| .gitignore/.editorconfig | `templates/.gitignore` → `.gitignore` |
|  | `templates/.editorconfig` → `.editorconfig` |
| LICENSE | `templates/LICENSE-<license>.tmpl` → `LICENSE` (for MIT or Apache alone) |
|  | for dual MIT+Apache: write both as `LICENSE-MIT` and `LICENSE-APACHE` |
| Makefile | `templates/Makefile.tmpl` → `Makefile` |
| README | `templates/README.md.tmpl` → `README.md` |
| examples | `templates/examples/starter.rs.tmpl` → `examples/starter.rs` |
| mdBook | `templates/book.toml.tmpl` → `book.toml` |
|  | `templates/docs/src/SUMMARY.md` → `docs/src/SUMMARY.md` (no .tmpl, no substitution) |
|  | `templates/docs/src/introduction.md.tmpl` → `docs/src/introduction.md` |
| CI | `templates/.github/workflows/ci.yml.tmpl` → `.github/workflows/ci.yml` |
| docs workflow | `templates/.github/workflows/docs.yml.tmpl` → `.github/workflows/docs.yml` |
| release workflow | `templates/.github/workflows/release.yml.tmpl` → `.github/workflows/release.yml` |
| codecov | `templates/codecov.yml` → `codecov.yml` (no substitution needed) |
| .claude/CLAUDE.md | `templates/.claude/CLAUDE.md.tmpl` → `.claude/CLAUDE.md` |
| AGENTS.md | `templates/AGENTS.md` → `AGENTS.md` |
| CLI subcrate | `templates/cli/Cargo.toml.tmpl` → `<crate>-cli/Cargo.toml` |
|  | `templates/cli/src/main.rs.tmpl` → `<crate>-cli/src/main.rs` |

### 4.3 Post-substitution adjustments

If CLI is **off**: edit `Cargo.toml` to remove the `[workspace]` section entirely (it was templated assuming workspace).

If CI is off: also skip copying `docs.yml.tmpl` and `release.yml.tmpl` even if those components are individually on — there's no workflow dir to put them in. Print a warning.

If license is `none`: write no LICENSE file; in `Cargo.toml` replace `license = "{{LICENSE_SPDX}}"` with `# license = "TODO — pick a license"` (commented TODO preserves Cargo.toml validity).

If CLI is **off**:
- Before substitution, set `CLI_BIN_NAME` to the empty string (the placeholder then drops out cleanly).
- After writing `README.md`, delete the `## CLI` section: drop the block starting at the `## CLI` heading up to (but not including) the next `## ` heading.
- Do not copy the `cli/` templates and do not include the workspace's `[workspace]` section (handled in the adjustment above).

---

## Step 5 — Verify

Run the following commands in sequence, inside the new crate directory. Print each command to the user as it runs. Abort on the first failure without proceeding to Step 6+:

```bash
cargo build --workspace
cargo test --workspace
cargo clippy --workspace --all-targets -- -D warnings
cargo fmt --all -- --check
```

If `--workspace` flags fail because CLI was disabled, fall back to the same commands without `--workspace`.

If any step fails:
- Keep the scaffolded tree in place.
- Do NOT initialise git (defer that to after the user fixes).
- Report the failing command and its stderr to the user.
- Ask: "Do you want me to attempt to fix, or do you want to inspect first?"

Record the result of each verification step — all four must be green before proceeding to the commit in Step 8.

---
````

- [ ] **Step 2: Verify section headings**

Run:
```bash
grep '^## Step' ~/.claude/skills/new-rust-crate/SKILL.md
```
Expected: lists Step 0 through Step 5.

---

### Task 16: Write SKILL.md — Step 6 (setup guides) + Step 7 (extras) + Step 8 (commit)

**Files:**
- Modify: `~/.claude/skills/new-rust-crate/SKILL.md` — append below Step 5.

- [ ] **Step 1: Append Step 6 (setup guides)**

Append:

````markdown

## Step 6 — Setup guides

For each enabled integration that requires out-of-band setup, print the exact steps the user must take outside the terminal. Only print guides for components that are **enabled** in the final component set.

### If codecov is enabled

```
Codecov setup:
  1. Visit https://app.codecov.io and link {{GH_OWNER}}/{{GH_REPO}}.
  2. Copy the upload token shown after linking.
  3. Add it as a repo secret:
       gh secret set CODECOV_TOKEN --body '<paste-token-here>' --repo {{GH_OWNER}}/{{GH_REPO}}
  4. The first push to main will upload coverage.
```

### If release workflow is enabled

```
crates.io publish setup:
  1. Visit https://crates.io/me and create an API token scoped to `publish-new` + `publish-update`.
  2. Add it as a repo secret:
       gh secret set CARGO_REGISTRY_TOKEN --body '<paste-token-here>' --repo {{GH_OWNER}}/{{GH_REPO}}
  3. To cut a release, run:
       make release V=0.1.0
     which tags v0.1.0 and pushes; CI will publish the crate.
```

### If docs workflow is enabled

```
GitHub Pages setup:
  1. After the first push to main, go to:
       https://github.com/{{GH_OWNER}}/{{GH_REPO}}/settings/pages
  2. Set "Source" to "GitHub Actions".
  3. The docs workflow will then deploy to https://{{GH_OWNER}}.github.io/{{GH_REPO}}/.
  (Alternatively: gh api -X PUT repos/{{GH_OWNER}}/{{GH_REPO}}/pages -f build_type=workflow)
```

Consolidate the printed guides into one message so the user sees the full todo list at once.

---

## Step 7 — Extras prompt

Ask the user:

```
Want to add any of these now? (pick any subset; reply "n" to skip all)
  a. criterion benches (benches/<name>_bench.rs + criterion dev-dep)
  b. proc-macro subcrate (<crate>-macros/ with a passthrough derive stub)
  c. GitHub issue templates (.github/ISSUE_TEMPLATE/{bug,feature}.yml)
```

For each item picked, write the corresponding files **in addition** to the tree scaffolded in Step 4.

### 7a. Criterion benches

Create `benches/<crate-name>_bench.rs`:
```rust
use criterion::{criterion_group, criterion_main, Criterion};

fn baseline(c: &mut Criterion) {
    c.bench_function("add", |b| b.iter(|| 2 + 2));
}

criterion_group!(benches, baseline);
criterion_main!(benches);
```

Append to `Cargo.toml`:
```toml
[dev-dependencies]
criterion = "0.8"

[[bench]]
name = "<crate-name>_bench"
harness = false
```

Append a `bench` target to `Makefile`:
```make
bench:
	cargo bench
```

### 7b. Proc-macro subcrate

Create directory `<crate-name>-macros/`, then write:

`<crate-name>-macros/Cargo.toml`:
```toml
[package]
name = "{{CRATE_NAME}}-macros"
version = "0.1.0"
edition = "2021"
description = "Procedural macros for {{CRATE_NAME}}"
license = "{{LICENSE_SPDX}}"
repository = "https://github.com/{{GH_OWNER}}/{{GH_REPO}}"

[lib]
proc-macro = true

[dependencies]
proc-macro2 = "1"
quote = "1"
syn = { version = "2", features = ["full"] }
```

`<crate-name>-macros/src/lib.rs`:
```rust
//! Proc-macros for {{CRATE_NAME}}.

use proc_macro::TokenStream;

#[proc_macro_derive(Hello)]
pub fn hello_derive(_input: TokenStream) -> TokenStream {
    "impl Hello for () { fn hello() { println!(\"hello\"); } }"
        .parse()
        .unwrap()
}
```

Add `"<crate-name>-macros"` to the workspace's `members` array in root `Cargo.toml`.

### 7c. Issue templates

Create `.github/ISSUE_TEMPLATE/bug.yml`:
```yaml
name: Bug Report
description: Something is broken
title: "[Bug]: "
labels: ["bug"]
body:
  - type: textarea
    id: what-happened
    attributes:
      label: What happened?
      description: A clear description of the bug.
    validations:
      required: true
  - type: textarea
    id: reproduce
    attributes:
      label: Reproduction steps
  - type: textarea
    id: version
    attributes:
      label: Environment (cargo version, OS, etc.)
```

Create `.github/ISSUE_TEMPLATE/feature.yml`:
```yaml
name: Feature Request
description: Suggest a new feature
title: "[Feature]: "
labels: ["enhancement"]
body:
  - type: textarea
    id: problem
    attributes:
      label: What problem are you trying to solve?
    validations:
      required: true
  - type: textarea
    id: solution
    attributes:
      label: Proposed solution
```

After any extra is added, re-run `cargo build --workspace` to make sure the tree still compiles. Do not re-run clippy/fmt/test (the user can do that with `make check`).

---

## Step 8 — Final commit and remote

If git was enabled in Step 3, run:

```bash
git init --initial-branch=main
git add -A
git commit -m "chore: scaffold <crate-name>"
```

If `gh repo create` was accepted:

```bash
gh repo create {{GH_OWNER}}/{{GH_REPO}} --<public|private> --source=. --remote=origin --push
```

(The `--push` flag handles `git push -u origin main` in one call.)

Finally, print a summary to the user:

```
✓ Scaffolded <crate-name> at <absolute-path>

Verified:
  ✓ cargo build
  ✓ cargo test
  ✓ cargo clippy -- -D warnings
  ✓ cargo fmt --check

Enabled components:
  <list each enabled component>

Next steps you must take outside this session:
  <repeat the Step 6 setup guides that were printed>

Useful commands:
  cd <crate-name>
  make help
  make check        # pre-commit gate
  make mdbook       # preview docs
```

---

## Rules and safety

- Never overwrite an existing directory. Abort on collision.
- Never force-push, never skip hooks, never bypass signing.
- Never create a remote without explicit Step 3 confirmation.
- Never claim success without running and reporting all four verify commands.
- Every tool install requires the user's explicit per-tool approval.
- If the user's crate name matches an existing crates.io name, warn but do not block — they may be publishing a renamed fork.

## Failure handling

- If any template is missing from `~/.claude/skills/new-rust-crate/templates/`, halt with an error pointing at the missing file — do not attempt to generate it on the fly.
- If `cargo build` fails after scaffold, print the stderr and ask the user whether to inspect or debug together.
- If the user interrupts mid-wizard, clean up only the target directory you created; never touch files outside it.
````

- [ ] **Step 2: Verify the SKILL.md is complete**

Run:
```bash
grep -c '^## Step' ~/.claude/skills/new-rust-crate/SKILL.md
wc -l ~/.claude/skills/new-rust-crate/SKILL.md
```
Expected: `9` (Step 0 through Step 8; counted as 9 because Step 0 and Steps 1–8 = 9 headings) and line count ≥ 400.

---

### Task 17: Mirror skill into VibeTraining repo, commit both

**Files:**
- Create: `/Users/liujinguo/website/VibeTraining/.claude/skills/new-rust-crate/` (mirror of `~/.claude/skills/new-rust-crate/`)
- Modify: (none; just a new directory tree committed to the repo)

- [ ] **Step 1: Copy the skill directory into the repo**

Run:
```bash
mkdir -p /Users/liujinguo/website/VibeTraining/.claude/skills
cp -R ~/.claude/skills/new-rust-crate /Users/liujinguo/website/VibeTraining/.claude/skills/
```

- [ ] **Step 2: Verify the mirror matches**

Run:
```bash
diff -r ~/.claude/skills/new-rust-crate /Users/liujinguo/website/VibeTraining/.claude/skills/new-rust-crate
```
Expected: no output (perfect copy).

- [ ] **Step 3: Commit the mirror in VibeTraining**

Run:
```bash
cd /Users/liujinguo/website/VibeTraining
git add .claude/skills/new-rust-crate docs/superpowers/plans/2026-04-20-new-rust-crate-skill.md
git status
```

Expected: the new skill tree and the plan are staged.

Then commit:
```bash
git commit -m "$(cat <<'EOF'
feat: add new-rust-crate skill

Scaffolds a Rust package shaped like ~/rcode/problemreductions:
workspace with lib + CLI subcrate, Makefile, mdBook docs,
CI/docs/release workflows, codecov, .claude/CLAUDE.md stub.
Also mirrors the personal copy at ~/.claude/skills/ for cross-project use.

Plan: docs/superpowers/plans/2026-04-20-new-rust-crate-skill.md
Spec: docs/superpowers/specs/2026-04-20-new-rust-crate-skill-design.md
EOF
)"
```

Expected: commit succeeds; `git status` after shows a clean tree.

- [ ] **Step 4: Final smoke test (manual)**

This step is manual — it runs the completed skill against a fresh directory and verifies the output. Document the test but do not try to script its execution inside the plan runner (the skill needs an interactive Claude Code session).

Test procedure:
```bash
cd /tmp
mkdir test-new-rust-crate && cd test-new-rust-crate
# In a fresh Claude Code session in this cwd, say:
#   "new rust crate"
# and answer the wizard with:
#   name: smoketest
#   description: Smoke test crate
#   author: (defaults)
#   license: MIT
#   GitHub owner: (defaults)
#   CLI binary name: (b) short nickname → "smk"
#   Components: defaults
#   git init: yes
#   gh repo create: no
```

Expected after the skill completes:
- `/tmp/test-new-rust-crate/smoketest/` exists with all default components.
- `cd smoketest && cargo build && cargo test && cargo clippy -- -D warnings && cargo fmt --all -- --check` all pass.
- `git log --oneline` shows `chore: scaffold smoketest`.
- Setup-guide output was printed for codecov, release, and docs-pages.
- Step 7 extras prompt was offered and declined cleanly.

If any of those fail, file the failure as a bug against the skill, fix in SKILL.md or the relevant template, and re-test. Clean up `/tmp/test-new-rust-crate/` when done.

---

## Self-review (done inline during plan writing)

- [x] **Spec coverage** — every component listed in the spec's "Default-on components" section has a corresponding template task (2–11) and a copy entry in Step 4 (Task 15).
- [x] **Placeholder scan** — no "TBD", "TODO in code", "fill in later" text remains in the plan. Placeholder tokens in templates (e.g., `{{CRATE_NAME}}`) are intentional and documented.
- [x] **Type consistency** — the same token names are used in templates (Task 2–11), in the substitute() helper (Task 15), and in the placeholder table at the top of the plan.
- [x] **Dependency rules** — mdBook → docs workflow and CI → codecov dependencies are encoded both in Step 2's auto-uncheck logic and in Step 4.3's post-substitution adjustments, keeping behaviour consistent.
- [x] **Scope** — this plan covers exactly the scope from the spec (one skill, its templates, a mirror copy). No creep.
