---
name: create-pull-request
description: Writes a complete Pull Request description from the branch diff using standardized pt-BR templates (feature, bug, release) and opens the PR with gh. Use when the user says "abrir PR", "criar pull request", "open a PR", "subir essa branch", "PR de release", "descrição do PR", or asks to document changes for review. Do not use for reviewing an existing PR's code, for writing commit messages, or for release notes and changelogs that are not a PR.
license: CC-BY-4.0
metadata:
  author: Humberto Barbosa
  version: 2.0.0
---

# Create Pull Request

Generate a PR description grounded in the actual diff — never in guesses — using the repository's standard templates, then open the PR.

## Core rules

- **The PR body is always written in Portuguese (pt-BR).** The templates are pt-BR; keep headings, checkboxes, and prose in pt-BR even when the conversation is in English.
- **Every claim must come from the diff, the commits, or the user.** If a template section cannot be filled from evidence, ask the user rather than inventing content. Never fabricate test evidence, incident links, or migration names.
- **Confirm before any outward action.** Pushing a branch and opening a PR are visible to the team — show the final body and the target base branch, and wait for approval before running `gh pr create`.

## Workflow

### Step 1: Gather context

Run these before writing anything:

```bash
git branch --show-current
git log --oneline origin/HEAD..HEAD
git diff origin/HEAD...HEAD --stat
```

Then read the actual changes (`git diff origin/HEAD...HEAD`) for the files that matter. For a large diff, read the full patch of source files and only the stat line for lockfiles, snapshots, and generated output.

Determine the base branch from `git symbolic-ref refs/remotes/origin/HEAD` unless the user names one.

If the branch has no commits ahead of base, stop and tell the user — there is nothing to open a PR for.

### Step 2: Select the template

Choose exactly one and read only that file:

| Signal | Template |
|---|---|
| Branch or commits prefixed `fix/`, `hotfix/`, `bugfix/`; diff corrects existing behavior; user mentions an error, incident, or regression | `./templates/bug.md` |
| Branch or commits prefixed `feat/`, `chore/`, `refactor/`, `perf/`; diff adds or changes capability | `./templates/feature.md` |
| Branch is a release/tag cut, or the diff is a merge of several PRs into a production branch | `./templates/release.md` |

When signals conflict, ask the user which one applies — do not merge two templates into one body.

### Step 3: Fill the template

Reproduce the chosen template's structure exactly — same headings, same order, same checkboxes. Then:

- **Replace every placeholder.** No `vX.Y.Z`, no empty numbered lists, no `@owner` left in the output.
- **Tick checkboxes only when verified.** A checked "Testes passando" must correspond to a test run you actually saw; otherwise leave it unchecked and say so.
- **Root cause (bug.md) must be technical.** "Erro de lógica" and similar vague answers are rejected by the template — name the specific condition: missing validation, race, incompatible migration, regression from a prior PR.
- **Rollback plan must be concrete steps**, not "reverter o PR".
- Cut sections that genuinely do not apply to a trivial diff (e.g. skip impact and migration detail for a pure lint or formatting change), but keep the template's skeleton recognizable.

### Step 4: Architecture and quality checklist

If the repo has a layered structure (`domain/`, `application/`, `infrastructure/`, or similar), add a short section splitting the changed files:

```markdown
## 🏛 Camadas Afetadas
**Core (Domain/Application):** <arquivos>
**Infraestrutura (Adapters/Config):** <arquivos>
```

Omit this section entirely when the repo is not layered.

Before presenting the body, verify and flag anything that fails:

- [ ] Business rules live in the entity/domain, not only in a controller or DTO
- [ ] Changed behavior has accompanying tests in the diff
- [ ] No debug logs, `console.log`, `.only`, or commented-out code in the diff
- [ ] No secrets, tokens, or `.env` values added

Report failures as a note to the user alongside the draft — do not silently fix them, and do not block on them.

For the repository's testing conventions, defer to `rules/testing.md` and the `testing-standards` skill. Do not restate them here.

### Step 5: Open the PR

Show the user the rendered body, the base branch, and the title. After approval:

```bash
git push -u origin <branch>            # only if the branch has no upstream
gh pr create --base <base> --title "<title>" --body-file <path>
```

Write the body to a temp file rather than passing it inline — `--body` mangles multi-line markdown.

Title format: `<tipo>: <resumo imperativo curto>` (e.g. `fix: corrige 500 em /payments com payload sem currency`). Match the repo's existing PR title style if `gh pr list` shows a clear convention.

Report the resulting PR URL. If `gh` is unavailable or unauthenticated, save the body to a file and give the user the path instead.

## Examples

### Example 1: Bug fix

User says: "abre o PR dessa branch"
Branch `fix/payments-null-currency`, diff adds a guard in `Payment` entity plus a unit test.

Actions:
1. Read commits and diff; detect `fix/` prefix → `templates/bug.md`.
2. Fill Problema from the failing behavior, Causa Raiz as "ausência de validação de `currency` no construtor de `Payment`, permitindo `undefined` chegar ao gateway".
3. Tick "Teste automatizado adicionado" (test is in the diff); leave "Teste em homolog" unchecked.
4. Add Camadas Afetadas: Core = `src/domain/payment.ts`, Infra = none.
5. Show draft, get approval, push, `gh pr create`.

Result: PR opened with title `fix: valida currency obrigatória em Payment`, URL returned.

### Example 2: Release

User says: "monta o PR de release da v2.3.0"

Actions:
1. Collect merged PRs with `gh pr list --state merged --base main --limit 30` and the commit range since the last tag.
2. Use `templates/release.md`; fill Tag, Data (range since previous tag), PRs incluídos with real numbers.
3. Scan the range for migration files; if found, fill Migrações with names and ordering, and ask the user to confirm rollback safety — never assert it yourself.
4. Present, confirm, open against the production branch.

### Example 3: Trivial change

User says: "PR só do prettier"

Actions: use `templates/feature.md`, mark Tipo as Refactor, fill Descrição and Checklist, and drop Contexto de negócio, Impactos, and Rollback detail — noting to the user that those were skipped as non-applicable.

## Troubleshooting

**`gh: command not found` or `gh auth status` fails** — Do not attempt to open the PR another way. Write the body to a file, report the path, and tell the user to run `gh auth login` or paste it manually.

**`origin/HEAD` is not set** — Run `git remote set-head origin -a`, or ask the user for the base branch. Do not assume `main`.

**Diff is too large to read fully** — Read the patch for source files, the stat for the rest, and state in your report which files you did not inspect so the user knows the description's coverage.

**The branch contains unrelated commits** — Say so and ask whether to describe everything or rebase first. Do not quietly describe only part of the diff.
