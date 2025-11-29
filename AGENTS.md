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

### Rule 2: Remove meta.yaml after recipe.yaml is complete

`recipe.yaml` is a **complete replacement** for `meta.yaml`, not a supplement. Once recipe.yaml is working:

```bash
cd packages/<package>-feedstock
git rm recipe/meta.yaml
git commit -m "Remove meta.yaml (replaced by recipe.yaml)"
```

Both formats should NOT coexist in the same feedstock.

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
2. Identify patterns and potential issues
3. Search for similar patterns: `bd list --labels pattern`
4. Add findings:
   ```bash
   bd comment <id> "Package type: Pure Python noarch
   Build system: hatchling
   Complexity: LOW"
   ```

### Phase 4: Conversion
1. Create `recipe.yaml` following CEP 13/14
2. **Bump build number** - Since version is unchanged but we're adding new format:
   ```yaml
   build:
     number: X+1  # Increment from meta.yaml's current build number
   ```
3. Apply known solution patterns from previous issues
4. Document conversion:
   ```bash
   bd comment <id> "✓ Created recipe.yaml with:
   - schema_version: 1
   - New context section
   - \${{ }} template syntax
   - New tests format
   - Build number bumped to X+1
   - meta.yaml will be removed after testing"
   ```

### Phase 5: Testing
1. Test build with rattler-build:
   ```bash
   cd packages/<package>-feedstock
   rattler-build build --recipe recipe/recipe.yaml
   ```

2. Document results:
   ```bash
   bd comment <id> "✓ Build successful
   Package: <package>-<version>-<build>.conda
   ✓ Tests passed"
   ```

3. If blocked:
   - Search for pattern: `bd list --labels pattern,<issue-type>`
   - Create pattern issue if new
   - Link issues: `bd comment <id> "Blocked by forge-fix-xyz"`

### Phase 6: Remove meta.yaml & Commit

Once recipe.yaml is tested and working, remove the old format:

```bash
cd packages/<package>-feedstock

# Remove meta.yaml (recipe.yaml is complete replacement)
git rm recipe/meta.yaml

# Commit both changes
git add recipe/recipe.yaml
git commit -m "Replace meta.yaml with recipe.yaml (CEP 13/14)

- Convert to recipe.yaml following CEP 13/14
- Use new \${{ }} template syntax
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
Note: meta.yaml removed (recipe.yaml is complete replacement)"
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

# 4. Test with rattler-build
rattler-build build --recipe recipe/recipe.yaml

bd comment forge-fix-962 "✓ Build successful with rattler-build
✓ Tests passed (imports + pip check)
Package: colorama-0.4.6-pyh4616a5c_2.conda"

# 5. Remove meta.yaml and commit both changes
git rm recipe/meta.yaml
git add recipe/recipe.yaml
git commit -m "Replace meta.yaml with recipe.yaml (CEP 13/14)

- Convert to recipe.yaml following CEP 13/14
- Use new \${{ }} template syntax
- Bump build number to 2 (version unchanged, adding new format)
- Remove meta.yaml (complete replacement)
- Tested successfully with rattler-build"

git push origin recipe-v1

bd comment forge-fix-962 "✅ Pushed to fork
Branch: https://github.com/zachcp/colorama-feedstock/tree/recipe-v1"

# 6. Update main repo with submodule
cd ../..
git add packages/colorama-feedstock .gitmodules
git commit -m "Add colorama-feedstock as submodule"
git push origin main

# 7. Document lessons and close
bd comment forge-fix-962 "Lessons learned:
- Pure Python packages are straightforward to convert
- No issues with noarch builds in rattler-build
- About section field renames: home→homepage, doc_url→documentation, dev_url→repository"

bd update forge-fix-962 --status closed --set-labels migration,python,noarch,success

# 8. Create solution template for reuse
bd create "Solution: Pure Python noarch template" --labels solution,python,template

bd comment forge-fix-quw "Template based on successful colorama migration.
Verified packages: colorama
Pattern works for: simple Python packages with hatchling/setuptools"

bd sync
```

**Result:** 
- Fork: https://github.com/zachcp/colorama-feedstock
- Branch: https://github.com/zachcp/colorama-feedstock/tree/recipe-v1
- Ready for PR: https://github.com/conda-forge/colorama-feedstock/compare/main...zachcp:colorama-feedstock:recipe-v1

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
3. Create PR to upstream feedstock
4. Update Beads issue with PR link:
   ```bash
   bd comment <id> "PR submitted: https://github.com/conda-forge/xxx-feedstock/pull/123"
   bd update <id> --labels upstream-pr
   ```
5. Track PR status in comments

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

**Last Updated**: 2025-01-20
**Tracking System**: Beads (`.beads/`)
**Maintainers**: AI Agents working on conda-forge migration
**Questions**: Search Beads issues or create new issue for discussion