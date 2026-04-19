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

If CLI is **off**: edit `Cargo.toml` to remove the `[workspace]` section entirely (it was templated assuming workspace). Also set `CLI_BIN_NAME` to the empty string before substitution, and after writing `README.md`, delete the `## CLI` section: drop the block starting at the `## CLI` heading up to (but not including) the next `## ` heading.

If CI is off: also skip copying `docs.yml.tmpl` and `release.yml.tmpl` even if those components are individually on — there's no workflow dir to put them in. Print a warning.

If license is `none`: write no LICENSE file; in `Cargo.toml` replace `license = "{{LICENSE_SPDX}}"` with `# license = "TODO — pick a license"` (commented TODO preserves Cargo.toml validity).

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
