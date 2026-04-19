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
