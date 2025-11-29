# AGENTS.md - Forge Fix Project

> **Mission:** Help conda-forge migrate from meta.yaml (CEP 13/14 v0) to recipe.yaml (CEP 13/14 v1) using rattler-build.

## Quick Reference

### Key Syntax Changes

| Aspect | Old (meta.yaml) | New (recipe.yaml) |
|--------|-----------------|-------------------|
| Template | `{{ var }}` | `${{ var }}` |
| Variables | `{% set name = "pkg" %}` | `context: { name: pkg }` |
| Selectors | `# [unix]` comments | `if/then/else` YAML |
| About fields | `home`, `doc_url`, `dev_url` | `homepage`, `documentation`, `repository` |

### Required Key Order (CRITICAL)

```yaml
schema_version: 1
context: { }
package: { }
source: { }
build: { }
requirements: { }
tests: { }
about: { }
extra: { }
```

### Python noarch Pattern

```yaml
# Do NOT set python_min in context - conda-forge provides it
requirements:
  host:
    - python ${{ python_min }}.*
    - pip
    - <build-backend>  # setuptools, hatchling, flit-core, or poetry-core
  run:
    - python >=${{ python_min }}

tests:
  - python:
      python_version: ${{ python_min }}.*  # REQUIRED for noarch
      imports:
        - package
      pip_check: true
```

**Local testing:** `rattler-build build --recipe recipe/recipe.yaml --variant python_min=3.10`

### Python C-Extension Pattern

```yaml
# For packages with compiled C extensions (NOT noarch)
build:
  number: X
  skip:
    - if: python_version < "3.7"
      then: true
  script: ${{ PYTHON }} -m pip install . -vv --no-deps --no-build-isolation

requirements:
  build:
    - if: build_platform != target_platform
      then:
        - python
        - cross-python_${{ target_platform }}
    - ${{ compiler('c') }}
    - ${{ stdlib('c') }}  # REQUIRED when using compiler
  host:
    - python
    - pip
    - <build-backend>  # setuptools, hatchling, flit-core, or poetry-core
  run:
    - python

tests:
  - python:
      imports:
        - package
      pip_check: true
```

**Key differences from noarch:**
- MUST include `${{ compiler('c') }}` and `${{ stdlib('c') }}` in build section
- Do NOT mark as `noarch: python` - compiled per platform
- No `python_version` constraint in tests section
- Skip uses `python_version` not `py` variable

**Local testing:** `rattler-build build --recipe recipe/recipe.yaml --variant python=3.11 --variant c_stdlib=macosx_deployment_target --variant c_stdlib_version=11.0`

**Example:** wrapt-feedstock

### Essential Commands

```bash
# Lint recipe
conda-smithy lint recipe

# Build locally (noarch python)
rattler-build build --recipe recipe/recipe.yaml --variant python_min=3.10

# Build locally (C-extension)
rattler-build build --recipe recipe/recipe.yaml --variant python=3.11

# Re-render feedstock
conda-smithy rerender --commit auto
```

---

## Project Overview

### Phased Approach

**Phase 1 (Current):** Pure Python noarch packages - simpler conversions, local builds, basic tests

**Phase 2 (In Progress):** C/C++ extensions & complex builds - cross-platform testing, Docker-based builds, complex variants
- **Status:** First C-extension package (wrapt) successfully migrated and tested
- **Next:** More C-extension packages, then full C++ libraries

### Resources

- [Progress Tracker](https://tdejager.github.io/are-we-recipe-v1-yet/)
- [CEP 13 - YAML Syntax](https://github.com/conda/ceps/blob/main/cep-0013.md)
- [CEP 14 - Schema](https://github.com/conda/ceps/blob/main/cep-0014.md)
- [Rattler-Build](https://github.com/prefix-dev/rattler-build)

### Repository Structure

```
forge-fix/
├── .beads/           # Issue tracking
├── AGENTS.md         # This file
├── README.md         # User documentation
├── packages/         # Feedstock submodules
└── docs/             # Additional docs
```

---

## Recipe Format Reference

### Selectors

```yaml
# Old format
dependencies:
  - make  # [unix]
  - m2-make  # [win]

# New format
dependencies:
  - if: unix
    then: make
  - if: win
    then: m2-make
```

### Field Name Changes

| meta.yaml | recipe.yaml |
|-----------|-------------|
| `home` | `homepage` |
| `doc_url` | `documentation` |
| `dev_url` | `repository` |
| `folder` (source) | `target_directory` |
| `recipe-maintainers` | `recipe-maintainers` (same!) |

### Build Backends

For pip-based recipes, MUST specify one in `host`:
- `setuptools` - Standard (setup.py or pyproject.toml)
- `hatchling` - Modern (pyproject.toml)
- `flit-core` - Lightweight (pyproject.toml)
- `poetry-core` - Poetry projects

Check package's `pyproject.toml` `[build-system]` section to determine which.

### Source URLs

Use `pypi.org` not `pypi.io`:
```yaml
source:
  url: https://pypi.org/packages/source/p/package/package-1.0.0.tar.gz
```

---

## Migration Workflow

### Prerequisites

```bash
brew install gh && gh auth login
pixi global install rattler-build
```

### Step 1: Setup

```bash
# Create tracking issue
bd create "Migrate <package> to recipe v1" --labels migration,<package>
bd update <id> --status in_progress

# Fork and clone
gh repo fork conda-forge/<package>-feedstock --clone=false
cd packages
git submodule add https://github.com/YOUR_USERNAME/<package>-feedstock.git
cd <package>-feedstock
git checkout -b recipe-v1
```

### Step 2: Analyze

1. Check current `meta.yaml` build number
2. Verify maintainers in `.github/CODEOWNERS` (not recipe-maintainers)
3. Search existing patterns: `bd list --labels pattern`

### Step 3: Convert

1. Create `recipe.yaml` following key order above
2. **Bump build number** (version unchanged = must bump)
3. Configure `conda-forge.yml`:
   ```yaml
   conda_build_tool: rattler-build
   conda_install_tool: pixi
   ```

### Step 4: Lint & Test

```bash
# Lint first
conda-smithy lint recipe

# Build (noarch python)
rattler-build build --recipe recipe/recipe.yaml --variant python_min=3.10
```

**Linting Checklist (noarch):**
- [ ] Key ordering correct
- [ ] Field names updated (homepage, documentation, repository)
- [ ] Build backend specified in host
- [ ] `python_version: ${{ python_min }}.*` in tests section
- [ ] Using pypi.org (not pypi.io)

**Linting Checklist (C-extension):**
- [ ] Key ordering correct
- [ ] Field names updated (homepage, documentation, repository)
- [ ] Build backend specified in host
- [ ] `${{ compiler('c') }}` and `${{ stdlib('c') }}` in build requirements
- [ ] Cross-compilation support with if/then/else
- [ ] NO `python_version` in tests section
- [ ] Using pypi.org (not pypi.io)

### Step 5: Finalize & Submit

```bash
# Remove old format and re-render
git rm recipe/meta.yaml
conda-smithy rerender --commit auto

# Commit
git add recipe/recipe.yaml conda-forge.yml
git commit -m "Replace meta.yaml with recipe.yaml (CEP 13/14)

- Convert to recipe.yaml with \${{ }} syntax
- Configure for rattler-build
- Bump build number to X
- Remove meta.yaml"

git push origin recipe-v1

# Create PR
gh pr create \
  --repo conda-forge/<package>-feedstock \
  --base main \
  --head YOUR_USERNAME:recipe-v1 \
  --title "Migrate to recipe.yaml (CEP 13/14)" \
  --body "Converts to recipe.yaml (CEP 13/14). Tested with rattler-build."
```

### Step 6: Update Tracking

```bash
bd comment <id> "PR: https://github.com/conda-forge/<package>-feedstock/pull/XX"
bd update <id> --status closed --set-labels success
bd sync

# Update main repo
cd ../..
git add packages/<package>-feedstock .gitmodules
git commit -m "Add <package>-feedstock as submodule"
```

---

## Beads Integration

Beads is an AI-native, git-integrated issue tracker in `.beads/`.

### Commands

```bash
bd create "Issue title" --labels label1,label2
bd list [--status open|closed] [--labels x,y]
bd show <id>
bd update <id> --status in_progress
bd comment <id> "Progress note"
bd sync
```

### Labels

- **Type:** `migration`, `pattern`, `solution`, `blocker`
- **Status:** `blocked`, `needs-testing`, `success`, `upstream-pr`
- **Package:** `python`, `cpp`, `header-only`, `multi-output`
- **Complexity:** `simple`, `medium`, `complex`

### Issue Templates

```bash
# Migration
bd create "Migrate colorama to recipe v1" \
  --description "Package type: Pure Python noarch
Feedstock: https://github.com/conda-forge/colorama-feedstock"

# Pattern (reusable problem)
bd create "Pattern: CMake config paths" --labels pattern,cmake

# Solution (reusable fix)
bd create "Solution: Python noarch template" --labels solution,python,template
```

---

## Troubleshooting

### "No valid recipes found" in CI

**Cause:** CI using conda-build but only recipe.yaml exists.

**Fix:**
```yaml
# conda-forge.yml
conda_build_tool: rattler-build
conda_install_tool: pixi
```
Then: `conda-smithy rerender --commit auto`

### python_min not substituted

**Symptom:** `error parsing 'python .*' as a match spec`

**Cause:** `${{ python_min }}` is empty - conda-forge CI injects it, but local builds need the variant.

**Fix (local):** `rattler-build build --variant python_min=3.10`

**Fix (CI):** Ensure conda-forge.yml is configured and re-rendered.

### Build number not bumped

**Cause:** Version unchanged requires build number increment.

**Fix:** Increment `build.number` from meta.yaml's value.

### Both meta.yaml and recipe.yaml exist

**Fix:** After configuring conda-forge.yml for rattler-build:
```bash
git rm recipe/meta.yaml
conda-smithy rerender --commit auto
```

### "undefined" package in build requirements (C-extension)

**Symptom:** `Cannot solve the request because of: No candidates were found for undefined *.`

**Cause:** `${{ stdlib('c') }}` not evaluating correctly - rattler-build support is still maturing.

**Fix (local testing):** Temporarily comment out `${{ stdlib('c') }}` OR provide variant:
```bash
rattler-build build --recipe recipe/recipe.yaml \
  --variant python=3.11 \
  --variant c_stdlib=macosx_deployment_target \
  --variant c_stdlib_version=11.0
```

**Fix (CI):** Keep `${{ stdlib('c') }}` - conda-forge CI provides proper variants.

### Skip condition not working locally (C-extension)

**Symptom:** Build skipped when using `if: python_version < "3.7"`

**Cause:** `python_version` not set in local variant (only `python` is set).

**Fix (local):** Comment out skip condition for testing, or add to CI-generated variant files.

**Fix (CI):** Condition works properly - conda-forge provides `python_version`.

---

## Example: Simple Python Package (colorama)

```bash
# 1. Setup
gh repo fork conda-forge/colorama-feedstock --clone=false
cd packages && git submodule add https://github.com/zachcp/colorama-feedstock.git
cd colorama-feedstock && git checkout -b recipe-v1

# 2. Check build number
grep "number:" recipe/meta.yaml  # Shows: 1

# 3. Create recipe.yaml (schema_version, context, package, source, build, requirements, tests, about, extra)
# - Use ${{ }} syntax
# - build.number: 2 (bumped from 1)
# - python ${{ python_min }}.* in host, python >=${{ python_min }} in run
# - python_version: ${{ python_min }}.* in tests

# 4. Configure
echo "conda_build_tool: rattler-build" >> conda-forge.yml
echo "conda_install_tool: pixi" >> conda-forge.yml

# 5. Lint & build
conda-smithy lint recipe
rattler-build build --recipe recipe/recipe.yaml --variant python_min=3.10

# 6. Finalize
git rm recipe/meta.yaml
conda-smithy rerender --commit auto
git add . && git commit -m "Replace meta.yaml with recipe.yaml"
git push origin recipe-v1

# 7. PR
gh pr create --repo conda-forge/colorama-feedstock --base main --head zachcp:recipe-v1 \
  --title "Migrate to recipe.yaml (CEP 13/14)"
```

**Key learnings:**
- Pure Python packages convert straightforwardly
- About field renames: home→homepage, doc_url→documentation, dev_url→repository
- MUST configure conda-forge.yml BEFORE removing meta.yaml
- MUST re-render after config changes

---

## Example: C-Extension Package (wrapt)

```bash
# 1. Setup
gh repo fork conda-forge/wrapt-feedstock --clone=false
cd packages && git submodule add https://github.com/zachcp/wrapt-feedstock.git
cd wrapt-feedstock && git checkout -b recipe-v1

# 2. Check build number and compiler requirements
grep "number:" recipe/meta.yaml  # Shows: 1
grep "compiler('c')" recipe/meta.yaml  # Confirms C extension

# 3. Create recipe.yaml with C compiler support
# - Use ${{ }} syntax
# - build.number: 2 (bumped from 1)
# - Add ${{ compiler('c') }} and ${{ stdlib('c') }} to build requirements
# - Add cross-compilation support with if/then/else
# - Convert skip selector: python_version < "3.7" with then: true
# - python in host, python in run (NO version constraints)
# - NO python_version in tests section

# 4. Configure
echo "conda_build_tool: rattler-build" >> conda-forge.yml
echo "conda_install_tool: pixi" >> conda-forge.yml

# 5. Lint & build
conda-smithy lint recipe
# For local testing, temporarily comment out skip or provide variants:
rattler-build build --recipe recipe/recipe.yaml --variant python=3.11

# 6. Finalize
git rm recipe/meta.yaml
conda-smithy rerender --commit auto
git add . && git commit -m "Replace meta.yaml with recipe.yaml (CEP 13/14)

- Convert to recipe.yaml with \${{ }} syntax
- Add C compiler and stdlib requirements
- Configure for rattler-build
- Bump build number to 2
- Maintain cross-compilation support"

git push origin recipe-v1

# 7. PR (when ready)
gh pr create --repo conda-forge/wrapt-feedstock --base main --head zachcp:recipe-v1 \
  --title "Migrate to recipe.yaml (CEP 13/14)" \
  --body "Converts to recipe.yaml (CEP 13/14) with C extension support. Tested with rattler-build."
```

**Key learnings:**
- C extensions require `${{ compiler('c') }}` and `${{ stdlib('c') }}`
- NOT noarch - builds per platform and Python version
- Cross-compilation uses if/then/else for build_platform != target_platform
- Skip conditions use `python_version` not `py`
- Local testing may require commenting skip or providing full variants
- Successfully built 89KB artifact with all tests passing

---

## Complex Example Notes: certifi

For packages with multiple sources:
- Use `target_directory` instead of `folder`
- Move `script_env` variables inline to `script` (not supported in rattler-build)
- Bootstrap wheels may trigger lint warnings (acceptable)

---

## Agent Collaboration

1. **Claim work:** `bd update <id> --status in_progress`
2. **Check conflicts:** `bd list --status in_progress`
3. **Share discoveries:** Create pattern/solution issues immediately
4. **Sync frequently:** `bd sync` after updates

---

## Version Info

- **conda-smithy:** 3.53.3+ (full rattler-build support)
- **rattler-build:** 0.53.0+
- **Last Updated:** 2025-01-29 (added C-extension pattern)