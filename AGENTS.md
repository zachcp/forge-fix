# AGENTS.md - Forge Fix Project

## Project Overview

This repository tracks upstream conda-forge packages that build using the classic recipe format (`meta.yaml`) but are being migrated to the new recipe format (`recipe.yaml`) using `rattler-build` instead of `conda-build`.

### Mission

Help conda-forge migrate from the classic recipe format (CEP 13/14 Version 0) to the new recipe format (CEP 13/14 Version 1) by:

1. Tracking packages that need migration
2. Testing recipes with `rattler-build`
3. Identifying and fixing compatibility issues
4. Contributing fixes back to upstream feedstocks
5. Monitoring progress across the ecosystem

### Phased Approach

**Phase 1 (Current): Pure Python Packages**
- Focus on noarch python packages (simpler conversions)
- Build locally with `rattler-build`
- Test with basic imports + pip check
- Create PRs for upstream submission
- Examples: pydantic, sqlalchemy, pytest, black, flake8

**Phase 2 (Deferred): C++ Extensions & Complex Builds**
- Packages with C/C++ extensions: numpy, pandas, scipy, matplotlib, cryptography, pillow
- Requires cross-platform testing: Linux, Windows, macOS, multiple architectures
- Plan: Docker-based build testing before PR submission
- Complex variant handling: BLAS/LAPACK, backend options, optional dependencies
- Status: Issues created but held until Phase 2 infrastructure ready

## Key Resources

### Documentation
- **Progress Tracker**: https://tdejager.github.io/are-we-recipe-v1-yet/
- **CEP 13** (YAML Syntax): https://github.com/conda/ceps/blob/main/cep-0013.md
- **CEP 14** (Schema Definition): https://github.com/conda/ceps/blob/main/cep-0014.md
- **Rattler-Build**: https://github.com/prefix-dev/rattler-build
- **Beads Documentation**: https://github.com/steveyegge/beads

### Key Differences Between Formats

#### File Names
- **Old format**: `meta.yaml`
- **New format**: `recipe.yaml`

#### Template Syntax
- **Old**: `{{ variable }}` (Jinja2)
- **New**: `${{ variable }}` (Valid YAML)

#### Recipe.yaml Key Ordering (CRITICAL)
The top-level keys must be in this exact order, or conda-smithy linting will fail:
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

**Important**: This is NOT the same order as meta.yaml. Never mix orders.

#### Selectors
- **Old**: Comments with `# [selector]`
- **New**: YAML dictionaries with `if/then/else`

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

#### Context Section
- **Old**: `{% set name = "package" %}`
- **New**: 
```yaml
context:
  name: package
  version: 1.2.3
```

#### Python Version Pinning (noarch: python)
For `noarch: python` recipes, conda-forge uses a default `python_min` context variable. **Do NOT explicitly set it in your context section** - it will be provided by conda-forge defaults unless you need to override it.

```yaml
# For packages using conda-forge default python_min:
requirements:
  host:
    - python ${{ python_min }}.*
    - pip
  run:
    - python >=${{ python_min }}

tests:
  - python:
      imports:
        - package
      pip_check: true
      python_version: ${{ python_min }}.*

# ONLY if you need to override the default:
context:
  python_min: "3.10"  # Only set if package requires Python >3.9
```

**Key Points (CRITICAL):**
- **Do NOT set `python_min` in context unless you need to override the default**
- **host**: Use `python ${{ python_min }}.*` for exact match during build
- **run**: Use `python >=${{ python_min }}` for minimum version at runtime
- **tests**: Use `python_version: ${{ python_min }}.*` for test environment (REQUIRED for noarch python)
  - This MUST be included in tests section for conda-smithy linting to pass
  - conda-forge will inject python_min at build time
- Override `python_min` ONLY if package needs newer Python than conda-forge default (3.9)

## Repository Structure

```
forge-fix/
├── .beads/                # Beads issue tracking (git-integrated)
├── AGENTS.md              # This file - AI agent instructions
├── README.md              # User-facing documentation
├── packages/              # Feedstock submodules/clones
└── docs/                  # Additional documentation
    ├── migration_guide.md
    └── common_patterns.md
```

## Build Number Guidelines

When adding `recipe.yaml` to an existing feedstock:

### Rule 1: Bump the build number if version is unchanged

Since we're adding a new file format but not changing the package contents or version, we MUST bump the build number:

```yaml
# meta.yaml has:
build:
  number: 1

# recipe.yaml should have:
build:
  number: 2  # Increment by 1
```

### Rule 2: Remove meta.yaml after conda-forge.yml is configured

`recipe.yaml` is a **complete replacement** for `meta.yaml`, not a supplement. Once recipe.yaml is working AND conda-forge.yml is configured for rattler-build:

```bash
cd packages/<package>-feedstock

# FIRST: Configure conda-forge.yml
# Ensure conda-forge.yml contains:
# conda_build_tool: rattler-build

# THEN: Re-render to update CI
conda-smithy rerender --commit auto

# FINALLY: Remove meta.yaml
git rm recipe/meta.yaml
git commit -m "Remove meta.yaml (replaced by recipe.yaml)"
```

**Important:** Both formats should NOT coexist in the same feedstock, but you MUST configure `conda-forge.yml` with `conda_build_tool: rattler-build` BEFORE removing `meta.yaml`, otherwise CI will fail with "No valid recipes found".

### Why Bump Build Number?

- Adding `recipe.yaml` changes the feedstock but not the upstream source
- conda-forge requires a build number bump when the version stays the same
- This creates a new package build: `package-version-build_2.conda`
- Users can distinguish between old meta.yaml build and new recipe.yaml build

### Checklist (from conda-forge PR template)

When creating PR:
- ✅ Used a personal fork of the feedstock
- ✅ **Bumped the build number (if the version is unchanged)**
- ✅ **Removed meta.yaml (recipe.yaml is complete replacement)**
- ⬜ Reset the build number to `0` (if the version changed)
- ⬜ Re-rendered with latest conda-smithy
- ✅ Ensured the license file is being packaged

### In Your Workflow

Always check `meta.yaml` for current build number before creating `recipe.yaml`:

```bash
# Check current build number
grep "number:" recipe/meta.yaml

# In recipe.yaml, increment it
# meta.yaml: number: 1
# recipe.yaml: number: 2
```

### Critical Configuration Requirement

Before removing `meta.yaml`, you MUST configure `conda-forge.yml`:

```yaml
conda_build_tool: rattler-build
```

Without this, conda-forge CI will use `conda-build` which only recognizes `meta.yaml`, causing build failures.

## Beads Integration

This project uses **Beads** - an AI-native, git-integrated issue tracking system that lives directly in the repository.

### Why Beads?

- **AI-Native**: CLI-first design perfect for AI agents and automated workflows
- **Git-Integrated**: Issues sync automatically with commits, no separate service needed
- **In-Repo**: Everything lives in `.beads/` alongside your code
- **Offline-First**: Works offline, syncs when you push
- **Flexible**: Use descriptions, comments, and labels to store rich knowledge

### Essential Beads Commands

```bash
# Create issues
bd create "Migrate xtensor to recipe v1"
bd create "Pattern: CMake config path differences" --labels pattern,cmake

# List and filter issues
bd list
bd list --status open
bd list --labels pattern
bd list --labels blocked

# View and update issues
bd show <issue-id>
bd update <issue-id> --status in_progress
bd update <issue-id> --status closed
bd update <issue-id> --labels migration,python,success

# Add detailed notes via comments
bd comment <issue-id> "Build failed with error: ..."
bd comment <issue-id> "Solution: Use \${{ compiler('cxx') }} instead"

# Sync with git
bd sync
```

## How to Use Beads for This Project

### Issue Types & Labels

Use labels to categorize issues:

- **Package migrations**: `migration`, `<package-name>`
- **Problem patterns**: `pattern`, `blocker`, `<category>` (cmake, python, testing, etc.)
- **Solutions**: `solution`, `<problem-type>`
- **Status**: `blocked`, `needs-testing`, `success`, `upstream-pr`
- **Complexity**: `simple`, `medium`, `complex`
- **Package type**: `python`, `cpp`, `header-only`, `multi-output`

### Issue Templates

#### For Package Migration
```bash
bd create "Migrate colorama to recipe v1" \
  --description "Convert colorama feedstock from meta.yaml to recipe.yaml.
Package type: Pure Python
Priority: HIGH - Good template for simple packages
Feedstock: https://github.com/conda-forge/colorama-feedstock

Important: Bump build number since version unchanged"
```

Then use comments to track progress:
```bash
bd comment <id> "✓ Cloned feedstock to packages/colorama-feedstock"
bd comment <id> "✓ Created recipe.yaml with new format"
bd comment <id> "✓ Build successful with rattler-build"
bd comment <id> "✓ Tests passed"
bd comment <id> "✓ Pushed to recipe-v1 branch"
```

#### For Problem Patterns
```bash
bd create "Pattern: CMake config path differences" \
  --labels pattern,cmake,high-frequency \
  --description "CMake config files installed to different paths between conda-build and rattler-build.

Common manifestations:
- Tests expect files in \${PREFIX}/lib/cmake/package/
- rattler-build may install to \${PREFIX}/share/cmake/package/

Affects: xtensor, xsimd, xtl, xtensor-blas"
```

Then document solution:
```bash
bd comment <id> "SOLUTION: Check both possible locations in tests:
\`\`\`yaml
tests:
  - script:
      - if: unix
        then:
          - test -f \${PREFIX}/share/cmake/pkg/Config.cmake || test -f \${PREFIX}/lib/cmake/pkg/Config.cmake
\`\`\`"
```

#### For Solutions
```bash
bd create "Solution: Python noarch conversion template" \
  --labels solution,python,template \
  --description "Template for converting simple Python noarch packages.

Based on successful colorama migration.

Steps:
1. Add schema_version: 1
2. Create context section
3. Change {{ }} to \${{ }}
4. Convert test section to tests with python element
5. Update about section field names
6. Replace variant config values with literals

Verified packages: colorama"
```

## Complete Migration Workflow

### Prerequisites
```bash
# Install GitHub CLI (if not already installed)
brew install gh

# Authenticate with GitHub
gh auth login

# Install rattler-build
pixi global install rattler-build
```

### Phase 1: Discovery & Setup
1. Check progress tracker for packages that need migration
2. Search existing Beads issues: `bd list --labels migration`
3. Create Beads issue:
   ```bash
   bd create "Migrate <package> to recipe v1" --labels migration,<package>
   bd update <id> --status in_progress
   ```

### Phase 2: Fork & Clone as Submodule

**Using GitHub CLI (recommended):**
```bash
# Fork the feedstock to your GitHub account
gh repo fork conda-forge/<package>-feedstock --clone=false

# Add as submodule (replace YOUR_USERNAME with your GitHub username)
cd packages
git submodule add https://github.com/YOUR_USERNAME/<package>-feedstock.git <package>-feedstock
cd <package>-feedstock

# Create recipe-v1 branch
git checkout -b recipe-v1
```

Document progress:
```bash
bd comment <id> "✓ Forked and added as submodule"
```

### Phase 3: Analysis
1. Analyze the current `meta.yaml`
2. **Verify recipe maintainers in CODEOWNERS file** - Do NOT copy invalid maintainers from recipe-maintainers section. Always check `.github/CODEOWNERS` and git history for correct maintainer names.
3. Identify patterns and potential issues
4. Search for similar patterns: `bd list --labels pattern`
5. Add findings:
   ```bash
   bd comment <id> "Package type: Pure Python noarch
   Build system: hatchling
   Complexity: LOW
   Maintainers verified in .github/CODEOWNERS"
   ```

### Phase 4: Conversion
1. Create `recipe.yaml` following CEP 13/14
2. **Configure conda-forge.yml for rattler-build**:
   ```bash
   # Check if conda-forge.yml exists
   cat conda-forge.yml
   
   # Add or update to include:
   # conda_build_tool: rattler-build
   ```
   
   Your `conda-forge.yml` should include:
   ```yaml
   conda_build_tool: rattler-build
   ```
   
   This tells conda-forge CI to use `rattler-build` instead of `conda-build`.
   
3. **Bump build number** - Since version is unchanged but we're adding new format:
   ```yaml
   build:
     number: X+1  # Increment from meta.yaml's current build number
   ```
4. Apply known solution patterns from previous issues
5. Document conversion:
   ```bash
   bd comment <id> "✓ Created recipe.yaml with:
   - schema_version: 1
   - New context section
   - \${{ }} template syntax
   - New tests format
   - Build number bumped to X+1
   - conda-forge.yml configured for rattler-build
   - meta.yaml will be removed after testing"
   ```

### Phase 5: Linting Checklist (Before PR Submission)

**CRITICAL: Do NOT submit a PR without passing all these checks:**

1. **Key ordering** - Must be: schema_version, context, package, source, build, requirements, tests, about, extra
2. **Field names** - Use `recipe-maintainers` (hyphens), not `recipe_maintainers` (underscores)
3. **About fields** - Use `homepage`, `documentation`, `repository` (not `home`, `doc_url`, `dev_url`)
4. **Build backend** - Python pip recipes MUST have setuptools, hatchling, flit-core, or poetry-core in host section
5. **Python pinning** - For `noarch: python`:
   - **DO NOT** explicitly set `python_min` in context (use conda-forge default)
   - Only override if package needs Python >3.9
   - Use `python ${{ python_min }}.*` in host, `python >=${{ python_min }}` in run
   - Use `python_version: ${{ python_min }}.*` in tests

### Phase 5: Testing and Linting

1. **Lint the recipe first**:
   ```bash
   cd packages/<package>-feedstock
   conda-smithy lint recipe
   ```
   
   Fix any issues before building (see "Common Lint Warnings" section below).

2. **Test build with rattler-build**:
   ```bash
   rattler-build build --recipe recipe/recipe.yaml
   ```

3. **Document results**:
   ```bash
   bd comment <id> "✓ Linting passed
   ✓ Build successful
   Package: <package>-<version>-<build>.conda
   ✓ Tests passed"
   ```

4. If blocked:
   - Search for pattern: `bd list --labels pattern,<issue-type>`
   - Create pattern issue if new
   - Link issues: `bd comment <id> "Blocked by forge-fix-xyz"`

### Phase 6: Re-render and Commit

Once recipe.yaml is tested and working, re-render the feedstock and finalize:

```bash
cd packages/<package>-feedstock

# Remove meta.yaml (recipe.yaml is complete replacement)
git rm recipe/meta.yaml

# Re-render with conda-smithy to update CI configuration
# This reads conda-forge.yml and regenerates all CI scripts for rattler-build
conda-smithy rerender --commit auto

# Or if using pixi:
pixi run conda-smithy rerender --commit auto

# Commit any remaining changes
git add recipe/recipe.yaml conda-forge.yml
git commit -m "Replace meta.yaml with recipe.yaml (CEP 13/14)

- Convert to recipe.yaml following CEP 13/14
- Use new \${{ }} template syntax
- Configure conda-forge.yml for rattler-build
- Re-render with conda-smithy
- Bump build number to X (version unchanged, adding new format)
- Remove meta.yaml (complete replacement)
- Tested successfully with rattler-build"

# Push to your fork
git push origin recipe-v1
```

Document:
```bash
bd comment <id> "✅ Pushed to fork
Branch: https://github.com/YOUR_USERNAME/<package>-feedstock/tree/recipe-v1
Note: meta.yaml removed (recipe.yaml is complete replacement)
Note: conda-forge.yml configured with rattler-build, feedstock re-rendered"
```

### Phase 7: Update Main Repo
```bash
# Return to main repo
cd ../..

# Commit the submodule
git add packages/<package>-feedstock .gitmodules
git commit -m "Add <package>-feedstock as submodule

- Forked from conda-forge/<package>-feedstock
- recipe-v1 branch pushed with converted recipe.yaml"

git push origin main
```

### Phase 8: Finalize
```bash
# Close the issue
bd update <id> --status closed --set-labels migration,<package>,success

# Document lessons learned
bd comment <id> "Lessons learned:
- [Key insight 1]
- [Key insight 2]
- [Key insight 3]"

# Sync Beads
bd sync
```

### Phase 9: Create Solution Template (if applicable)
If you found a reusable pattern:
```bash
bd create "Solution: <Pattern Name>" \
  --labels solution,<category>,template

bd comment <new-id> "Template based on successful <package> migration.
Verified packages: <package>
Pattern works for: [description]

Steps:
1. [Step 1]
2. [Step 2]
..."
```

## Common Migration Patterns

Document patterns as Beads issues with `pattern` label.

### Pattern Categories

- **Build System**: cmake, meson, autotools
- **Language**: python, cpp, rust, r
- **Testing**: test-commands, imports, downstream
- **Packaging**: multi-output, noarch, entry-points
- **Dependencies**: pin-subpackage, compiler, run-exports

### Searching Patterns

```bash
# Find all documented patterns
bd list --labels pattern

# Find patterns by category
bd list --labels pattern,cmake
bd list --labels pattern,python

# Find solutions
bd list --labels solution

# Find successful migrations for reference
bd list --labels success,migration
```

## Success Metrics

Track progress via Beads:

```bash
# Active migrations
bd list --status open --labels migration

# Completed migrations
bd list --status closed --labels success

# Blocked packages
bd list --labels blocked

# Known patterns
bd list --labels pattern

# Available solutions
bd list --labels solution
```

## Agent Collaboration Protocol

Multiple agents can work on this repository simultaneously using Beads:

1. **Claim work**: Create or update issue, set status to `in_progress`
2. **Check for conflicts**: `bd list --status in_progress` before starting
3. **Share discoveries**: Create pattern/solution issues immediately
4. **Link related issues**: Use comments to reference other issues
5. **Sync frequently**: `bd sync` after significant updates
6. **Document thoroughly**: Use comments for detailed progress notes

## Query Examples

### Find packages to work on
```bash
bd list --status open --labels migration
```

### Find if a pattern is documented
```bash
bd list --labels pattern | grep -i "cmake"
```

### Get details on a solution
```bash
bd show <solution-issue-id>
```

### Find similar successful migrations
```bash
bd list --labels success,python
bd list --labels success,header-only
```

### Find blocked packages
```bash
bd list --labels blocked
```

## Using Beads Comments for Rich Documentation

Beads comments support markdown, so use them for:

### Code snippets
```bash
bd comment <id> "Solution code:
\`\`\`yaml
context:
  name: package
  version: 1.0.0
\`\`\`"
```

### Error logs
```bash
bd comment <id> "Build error:
\`\`\`
Error: CMake config not found at expected path
\`\`\`"
```

### Links to resources
```bash
bd comment <id> "Related: forge-fix-123
Upstream issue: https://github.com/..."
```

### Checklists
```bash
bd comment <id> "Progress:
- [x] Clone feedstock
- [x] Create recipe.yaml
- [ ] Test build
- [ ] Push branch"
```

## Example Workflow: colorama Migration

Complete example showing the full process:

```bash
# 1. Create tracking issue
bd create "Migrate colorama to recipe v1" --labels migration,python,noarch
bd update forge-fix-962 --status in_progress

# 2. Fork and add as submodule using GitHub CLI
gh repo fork conda-forge/colorama-feedstock --clone=false
# Output: https://github.com/zachcp/colorama-feedstock

cd packages
git submodule add https://github.com/zachcp/colorama-feedstock.git colorama-feedstock
cd colorama-feedstock
git checkout -b recipe-v1

bd comment forge-fix-962 "✓ Forked and added as submodule"

# 3. Analyze and convert
# (Create recipe.yaml following CEP 13/14)
# Important: Bump build number from 1 to 2 since version unchanged

bd comment forge-fix-962 "✓ Created recipe.yaml with:
- schema_version: 1
- New context section
- \${{ }} template syntax
- New tests format with python element
- Build number bumped to 2"

# 4. Configure conda-forge.yml for rattler-build
# Check current conda-forge.yml
cat conda-forge.yml

# Add the critical line if missing:
# conda_build_tool: rattler-build

bd comment forge-fix-962 "✓ Updated conda-forge.yml with conda_build_tool: rattler-build"

# 5. Test with rattler-build locally
rattler-build build --recipe recipe/recipe.yaml

bd comment forge-fix-962 "✓ Build successful with rattler-build
✓ Tests passed (imports + pip check)
Package: colorama-0.4.6-pyh4616a5c_2.conda"

# 6. Remove meta.yaml and re-render
git rm recipe/meta.yaml

# Re-render to update CI configuration for rattler-build
conda-smithy rerender --commit auto

# Commit all changes
git add recipe/recipe.yaml conda-forge.yml
git commit -m "Replace meta.yaml with recipe.yaml (CEP 13/14)

- Convert to recipe.yaml following CEP 13/14
- Use new \${{ }} template syntax
- Configure conda-forge.yml for rattler-build
- Re-render with conda-smithy
- Bump build number to 2 (version unchanged, adding new format)
- Remove meta.yaml (complete replacement)
- Tested successfully with rattler-build"

git push origin recipe-v1

bd comment forge-fix-962 "✅ Pushed to fork
Branch: https://github.com/zachcp/colorama-feedstock/tree/recipe-v1
✅ conda-forge.yml configured with rattler-build
✅ Feedstock re-rendered for CI compatibility"

# 7. Update main repo with submodule
cd ../..
git add packages/colorama-feedstock .gitmodules
git commit -m "Add colorama-feedstock as submodule"
git push origin main

# 8. Document lessons and close
bd comment forge-fix-962 "Lessons learned:
- Pure Python packages are straightforward to convert
- No issues with noarch builds in rattler-build
- About section field renames: home→homepage, doc_url→documentation, dev_url→repository
- CRITICAL: Must add conda_build_tool: rattler-build to conda-forge.yml
- Must re-render with conda-smithy after adding conda-forge.yml config"

bd update forge-fix-962 --status closed --set-labels migration,python,noarch,success

# 9. Create solution template for reuse
bd create "Solution: Pure Python noarch template" --labels solution,python,template

bd comment forge-fix-quw "Template based on successful colorama migration.
Verified packages: colorama
Pattern works for: simple Python packages with hatchling/setuptools"

bd sync

# 10. Create PR to upstream
cd packages/colorama-feedstock

gh pr create \
  --repo conda-forge/colorama-feedstock \
  --base main \
  --head zachcp:recipe-v1 \
  --title "Migrate to recipe.yaml (CEP 13/14)" \
  --body "This PR migrates the feedstock from meta.yaml to recipe.yaml following CEP 13 and CEP 14.

## Changes
- Convert to recipe.yaml format with new \${{ }} template syntax
- Configure conda-forge.yml for rattler-build
- Bump build number from 1 to 2 (version unchanged, new format)
- Remove meta.yaml (complete replacement)
- Re-render with conda-smithy for CI compatibility

## Testing
- ✅ Built successfully with rattler-build locally
- ✅ All tests passed (imports + pip check)

## Checklist
- [x] Used a personal fork of the feedstock
- [x] Bumped the build number (version unchanged)
- [x] Re-rendered with latest conda-smithy
- [x] Ensured the license file is being packaged
- [x] Removed meta.yaml (recipe.yaml is complete replacement)
- [x] Tested successfully with rattler-build"

bd comment forge-fix-962 "PR created: https://github.com/conda-forge/colorama-feedstock/pull/XX"
```

**Result:** 
- Fork: https://github.com/zachcp/colorama-feedstock
- Branch: https://github.com/zachcp/colorama-feedstock/tree/recipe-v1
- PR: Created via `gh pr create` command

## Status Definitions

- **open**: Package identified, not yet started
- **in_progress**: Active migration work happening
- **closed**: Work completed (use labels to indicate success/failure)

Use labels for additional state:
- **blocked**: Cannot proceed due to unresolved issue
- **needs-testing**: Conversion done, needs verification
- **upstream-pr**: PR submitted to feedstock
- **success**: Fully completed and verified

## Contributing Fixes Upstream

Once a package successfully builds and tests:

1. Ensure feedstock is forked to your GitHub
2. Push recipe-v1 branch
3. Create PR to upstream feedstock using GitHub CLI:
   ```bash
   cd packages/<package>-feedstock
   
   gh pr create \
     --repo conda-forge/<package>-feedstock \
     --base main \
     --head zachcp:recipe-v1 \
     --title "Migrate to recipe.yaml (CEP 13/14)" \
     --body "This PR migrates the feedstock from meta.yaml to recipe.yaml following CEP 13 and CEP 14.

   ## Changes
   - Convert to recipe.yaml format with new \${{ }} template syntax
   - Configure conda-forge.yml for rattler-build
   - Bump build number (version unchanged, new format)
   - Remove meta.yaml (complete replacement)
   - Re-render with conda-smithy for CI compatibility

   ## Testing
   - ✅ Built successfully with rattler-build locally
   - ✅ All tests passed (imports + pip check)

   ## Checklist
   - [x] Used a personal fork of the feedstock
   - [x] Bumped the build number (version unchanged)
   - [x] Re-rendered with latest conda-smithy
   - [x] Ensured the license file is being packaged
   - [x] Removed meta.yaml (recipe.yaml is complete replacement)
   - [x] Tested successfully with rattler-build"
   ```

4. Update Beads issue with PR link:
   ```bash
   bd comment <id> "PR submitted: https://github.com/conda-forge/xxx-feedstock/pull/123"
   bd update <id> --labels upstream-pr
   ```
5. Track PR status in comments

**Note:** The `gh pr create` command automatically detects fork relationships and creates the PR against the upstream repository.

## Linting and Quality Checks

### conda-smithy lint

Run linting on your recipe to check for common issues:

```bash
cd packages/<package>-feedstock
conda-smithy lint recipe
```

**Note:** `conda-smithy lint` supports both `meta.yaml` and `recipe.yaml` formats.

### Linting Workflow Checklist

Always run the linter before committing:

```bash
# 1. Lint the recipe
conda-smithy lint recipe

# 2. Fix any issues (see Common Lint Warnings below)

# 3. Lint again to verify
conda-smithy lint recipe

# Expected output when passing:
# "recipe is in fine form"
```

**When to lint:**
- After creating `recipe.yaml`
- Before committing changes
- Before pushing to your fork
- Before creating a PR

**Common workflow:**
```bash
# Create recipe.yaml
vim recipe/recipe.yaml

# Lint and fix issues
conda-smithy lint recipe
# ... make fixes ...
conda-smithy lint recipe  # Verify

# Test build
rattler-build build --recipe recipe/recipe.yaml

# Commit
git add recipe/recipe.yaml
git commit -m "Add recipe.yaml with all lint checks passing"
```

### recipe.yaml Field Name Reference

When migrating from meta.yaml to recipe.yaml, these field names change:

**extra section (CRITICAL - uses hyphens, not underscores):**
- `meta.yaml`: `extra: recipe-maintainers:`
- `recipe.yaml`: `extra: recipe-maintainers:` ✅ (same!)

**about section:**
- `meta.yaml`: `home:`, `doc_url:`, `dev_url:`
- `recipe.yaml`: `homepage:`, `documentation:`, `repository:`

**Python build backends:**
For pip-based Python recipes, you MUST list a build backend in `host`:
- `setuptools` - Standard, uses setup.py or pyproject.toml
- `hatchling` - Modern, uses pyproject.toml
- `flit-core` - Lightweight, uses pyproject.toml
- `poetry-core` - For poetry projects

Without this, conda-smithy will warn about missing build backend.

### Common Lint Warnings

#### 1. Python Version Pinning (noarch: python)

**Warning:**
```
noarch: python recipes should usually follow the syntax in our documentation 
for specifying the Python version.
```

**Solution:** Use the `python_min` pattern in your `recipe.yaml`:

```yaml
context:
  name: package
  version: 1.0.0
  python_min: "3.9"  # Set minimum Python version

requirements:
  host:
    - python ${{ python_min }}.*  # Exact match for build
    - pip
  run:
    - python >=${{ python_min }}  # Minimum version for runtime

tests:
  - python:
      python_version: ${{ python_min }}.*  # Test with minimum version
      imports:
        - package
```

**Why this matters:**
- Ensures package is tested with minimum supported Python version
- Makes Python version requirements explicit and consistent
- Allows easy updates when dropping old Python versions

#### 2. Missing Build Backend

**Warning:**
```
No valid build backend found for Python recipe for package X using pip. 
Python recipes using pip need to explicitly specify a build backend in the host section.
```

**Solution:** Add the appropriate build backend to the `host` section:

```yaml
requirements:
  host:
    - pip
    - python ${{ python_min }}.*
    - setuptools  # or hatchling, flit-core, etc.
```

**Common build backends:**
- `setuptools` - Most common, uses setup.py or pyproject.toml
- `hatchling` - Modern build backend, uses pyproject.toml
- `flit-core` - Lightweight, uses pyproject.toml
- `poetry-core` - For poetry projects

**How to determine which to use:**
1. Check the package's `pyproject.toml` for `[build-system]` section
2. If it has `setup.py`, use `setuptools`
3. Check PyPI or the source repository

#### 3. PyPI URL (pypi.io vs pypi.org)

**Warning:**
```
PyPI default URL is now pypi.org, and not pypi.io. 
You may want to update the default source url.
```

**Solution:** Change `pypi.io` to `pypi.org` in the source URL:

```yaml
# Old (incorrect)
source:
  url: https://pypi.io/packages/source/p/package/package-1.0.0.tar.gz

# New (correct)
source:
  url: https://pypi.org/packages/source/p/package/package-1.0.0.tar.gz
```

### rattler-build Commands

**Note:** `rattler-build` does **not** currently have a dedicated `lint` command. However, you can:

1. **Validate syntax** by running a build:
   ```bash
   rattler-build build --recipe recipe/recipe.yaml
   ```

2. **Use conda-smithy lint** (supports recipe.yaml):
   ```bash
   conda-smithy lint recipe
   ```

3. **Check schema** using YAML language server in your editor:
   ```yaml
   # yaml-language-server: $schema=https://raw.githubusercontent.com/prefix-dev/recipe-format/main/schema.json
   schema_version: 1
   ```

## Known Issues & Limitations

### conda-smithy re-render and rattler-build

Currently, conda-smithy's re-render command doesn't fully support rattler-build CI script generation for all platforms. This means:

- Configure `conda_build_tool: rattler-build` in conda-forge.yml
- BUT: Do NOT run `conda-smithy rerender` yet (it removes the conda-build scripts without proper rattler-build replacements)
- CI will still show warnings but the feedstock maintainers can manually configure rattler-build after the PR is merged
- Alternative: Manually update `.azure-pipelines/`, `.circleci/`, `.github/workflows/` to use rattler-build instead of conda-build

This is a temporary limitation while conda-smithy adds full rattler-build support.

## Troubleshooting Common Issues

### Error: "No valid recipes found" in CI

**Symptom:**
```
ValueError: No valid recipes found for input: ['/home/conda/recipe_root']
```

**Cause:** Conda-forge CI is using `conda-build` (which only recognizes `meta.yaml`) but you only have `recipe.yaml`.

**Solution:**
1. Add `conda_build_tool: rattler-build` to `conda-forge.yml`:
   ```yaml
   conda_build_tool: rattler-build
   ```

2. Re-render the feedstock:
   ```bash
   conda-smithy rerender --commit auto
   ```

3. Commit and push:
   ```bash
   git add .
   git commit -m "Configure CI for rattler-build"
   git push origin recipe-v1
   ```

**Key Points:**
- Conda-forge defaults to `conda-build` unless explicitly configured
- `conda-forge.yml` must specify `conda_build_tool: rattler-build`
- Re-rendering regenerates all CI scripts to use rattler-build
- You can have ONLY `recipe.yaml` after this configuration
- See example: [python-librt-feedstock](https://github.com/conda-forge/python-librt-feedstock)

### Error: Both meta.yaml and recipe.yaml exist

**Symptom:** Feedstock has both `recipe/meta.yaml` and `recipe/recipe.yaml`.

**Cause:** During migration, both files were kept.

**Solution:** After configuring `conda-forge.yml` for rattler-build, remove `meta.yaml`:
```bash
git rm recipe/meta.yaml
git commit -m "Remove meta.yaml (replaced by recipe.yaml)"
```

**Note:** `recipe.yaml` is a complete replacement, not a supplement.

### Error: Build number not bumped

**Symptom:** CI rejects the PR because version unchanged but build number not incremented.

**Cause:** Adding `recipe.yaml` changes the feedstock but not package version.

**Solution:** In `recipe.yaml`, increment build number:
```yaml
build:
  number: X+1  # Where X is the current build number in meta.yaml
```

### Error: Azure CI still uses conda-build (even after recipe.yaml added)

**Symptom:**
```
ValueError: No valid recipes found for input: ['/home/conda/recipe_root']
```

When Azure CI runs but `.scripts/build_steps.sh` still invokes `conda-build` instead of `rattler-build`.

**Root Cause:** The `.scripts/build_steps.sh` file is auto-generated by `conda-smithy rerender` and contains hardcoded `conda-build` commands. Simply adding `recipe.yaml` without updating the build script means CI still tries to use `conda-build`, which doesn't recognize `recipe.yaml` format.

**Solution - Fix the Build Script:**

Update `.scripts/build_steps.sh` to detect `recipe.yaml` and use `rattler-build`:

1. **Find the conda-build invocation** (usually around line 70):
   ```bash
   conda-build "${RECIPE_ROOT}" -m "${CI_SUPPORT}/${CONFIG}.yaml" \
       --suppress-variables ${EXTRA_CB_OPTIONS:-} \
       --clobber-file "${CI_SUPPORT}/clobber_${CONFIG}.yaml" \
       --extra-meta flow_run_id="${flow_run_id:-}" remote_url="${remote_url:-}" sha="${sha:-}"
   ```

2. **Replace with conditional logic:**
   ```bash
   # Use rattler-build if recipe.yaml exists, otherwise fall back to conda-build
   if [[ -f "${RECIPE_ROOT}/recipe.yaml" ]]; then
       rattler-build build \
           --recipe "${RECIPE_ROOT}/recipe.yaml" \
           --build-dir "${FEEDSTOCK_ROOT}/build_artifacts"
   else
       conda-build "${RECIPE_ROOT}" -m "${CI_SUPPORT}/${CONFIG}.yaml" \
           --suppress-variables ${EXTRA_CB_OPTIONS:-} \
           --clobber-file "${CI_SUPPORT}/clobber_${CONFIG}.yaml" \
           --extra-meta flow_run_id="${flow_run_id:-}" remote_url="${remote_url:-}" sha="${sha:-}"
   fi
   ```

3. **Install rattler-build in CI** (around line 36-38):
   ```bash
   micromamba install --root-prefix ~/.conda --prefix /opt/conda \
       --yes --override-channels --channel conda-forge --strict-channel-priority \
       pip  python=3.12 conda-build conda-forge-ci-setup=4 "conda-build>=24.1" rattler-build
   ```

4. **Commit and push:**
   ```bash
   git add .scripts/build_steps.sh
   git commit -m "Fix CI build script to support recipe.yaml with rattler-build
   
   - Add conditional logic to detect recipe.yaml and use rattler-build
   - Fall back to conda-build if only meta.yaml exists (backwards compatible)
   - Install rattler-build in CI environment
   - Fixes 'No valid recipes found' error when using recipe.yaml"
   
   git push origin recipe-v1
   ```

**Why This Happens:**
- `conda-smithy rerender` generates CI scripts based on `conda-forge.yml` settings
- Conda-smithy doesn't yet fully support `rattler-build` CI script generation
- The generated `.scripts/build_steps.sh` contains hardcoded `conda-build` invocations
- You must manually update the build script to support `recipe.yaml`

**Important Notes:**
- This fix is backwards compatible - if `meta.yaml` exists (old packages), it falls back to `conda-build`
- When only `recipe.yaml` exists, it uses `rattler-build`
- This allows the same feedstock to work during migration (tests both formats)
- The fix should be committed to your fork before the PR is created

## Tips for AI Agents

1. **Use descriptive issue titles** - Makes searching easier
2. **Label generously** - Multiple labels help with discovery
3. **Comment frequently** - Use comments like a notebook
4. **Link related issues** - Build knowledge graph via references
5. **Create pattern issues early** - Don't wait for 3+ occurrences
6. **Update solutions** - Add verified packages to solution issues
7. **Sync often** - Keep knowledge shared: `bd sync`

## Next Steps

1. Set up `packages/` directory for feedstock clones
2. Create initial migration issues for priority packages
3. Document first patterns as they emerge
4. Build solution library through successful migrations
5. Scale to more complex packages

---

**Last Updated**: 2025-11-29
**Tracking System**: Beads (`.beads/`)
**Maintainers**: AI Agents working on conda-forge migration
**Questions**: Search Beads issues or create new issue for discussion
