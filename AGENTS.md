# AGENTS.md - Forge Fix Project

> **Mission:** Migrate conda-forge from meta.yaml to recipe.yaml (CEP 13/14 v1) using rattler-build.

## 🛠️ Automatic Conversion Tool: conda-recipe-manager

### Recommended Starting Point

The fastest way to start a conversion is using the automatic conversion utility:

```bash
# Install the tool
pixi global install conda-recipe-manager
# or
conda install -c conda-forge conda-recipe-manager

# Convert meta.yaml to recipe.yaml
cd <package>-feedstock
conda-recipe-manager convert recipe/meta.yaml > recipe/recipe.yaml
```

**Note:** The automatic conversion is a great starting point but **requires manual review**:
- Verify key ordering matches schema requirements
- Check that selectors converted properly (`# [unix]` → `if: unix` + `then:`)
- Validate test section structure (now a list of tests)
- Ensure build/run requirements are correct
- Update field names (`home` → `homepage`, etc.)
- Check variant key usage in build scripts

**Our Workflow:** Use `conda-recipe-manager` to generate the initial recipe, then:
1. Review and fix any conversion issues
2. Test locally with rattler-build
3. Commit only recipe files (not rerendered files)
4. Let conda-forge bot handle CI regeneration

## ⚠️ Known Blocker

**C-extension packages:** Rattler-build CI support in conda-smithy is incomplete. When `conda_build_tool: rattler-build` is set, rerender deletes .ci_support files but doesn't generate replacement CI workflows. **Focus on noarch Python packages only** until tooling matures. See issue: forge-fix-l3s

## Quick Reference

### Key Learnings from Recent Conversions

#### 1. **Context Variables**
- Define name and version in `context:` section (not hardcoded)
- `python_min` should NOT be in context - it's injected by conda-forge CI
- Use `${{ variable }}` syntax (note the `$` prefix for rattler-build)

Example:
```yaml
schema_version: 1
context:
  name: mypackage
  version: 1.0.0
package:
  name: ${{ name }}
  version: ${{ version }}
```

#### 2. **Source URLs with Variables**
- Always use variables for dynamic URL construction
- Pattern: `https://pypi.org/packages/source/${{ name[0] }}/${{ name }}/...`

```yaml
source:
  url: https://pypi.org/packages/source/${{ name[0] }}/${{ name }}/${{ name }}-${{ version }}.tar.gz
```

For packages where source tarball filename differs from package name, adapt accordingly:
```yaml
source:
  url: https://pypi.org/packages/source/${{ name[0] }}/${{ name }}/package_slug-${{ version }}.tar.gz
```

#### 3. **Python Version in Tests**
- Test section requires `python_version: ${{ python_min }}.*` (injected by CI)
- Local testing: pass `--variant python_min=3.8` to rattler-build
- Do NOT hardcode python_min in context

#### 4. **pip install Script**
- Standard: `python -m pip install . --no-deps -vv`
- Use `--no-build-isolation` only for packages with non-setuptools build backends (poetry, hatchling)
- Never include `--ignore-installed` in conda recipes

### Syntax Changes (Key Conversions)

| Old (meta.yaml) | New (recipe.yaml) |
|-----------------|-------------------|
| `{{ var }}` | `${{ var }}` (note the `$` prefix) |
| `{% set name = "pkg" %}` | `context: { name: pkg }` |
| `# [unix]` selector | `if: unix` + `then:` or `${{ "value" if unix }}` |
| `home`, `doc_url`, `dev_url` | `homepage`, `documentation`, `repository` |
| `build.run_exports` | `requirements.run_exports` |
| `requirements.run_constrained` | `requirements.run_constraints` |
| `build.ignore_run_exports` | `requirements.ignore_run_exports.by_name` |
| `build.ignore_run_exports_from` | `requirements.ignore_run_exports.from_package` |
| `bld.bat` | `build.bat` |
| `test:` (single) | `tests:` (list of tests) |
| `git_url`, `git_rev` | `git:`, `tag:` or `rev:` |

### Selector Conversion Examples

**meta.yaml:**
```yaml
requirements:
  host:
    - pywin32  # [win]
    - libffi   # [not win]
```

**recipe.yaml (two options):**
```yaml
requirements:
  host:
    # Option 1: Inline jinja (preferred for simple cases)
    - ${{ "pywin32" if win }}
    - ${{ "libffi" if not win }}
    
    # Option 2: if/then map (better for multiple items)
    - if: win
      then:
        - pywin32
    - if: not win
      then:
        - libffi
```

### Test Section Conversion

**meta.yaml (single test):**
```yaml
test:
  imports:
    - mypackage
  commands:
    - mypackage --version
```

**recipe.yaml (list of independent tests):**
```yaml
tests:
  - script:
      - mypackage --version
  - python:
      imports:
        - mypackage
      pip_check: true  # default, can disable
```

Each test runs in its own environment!

### Key Order (Required)

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

## Recipe Patterns

### Python noarch

```yaml
requirements:
  host:
    - python ${{ python_min }}.*
    - pip
    - <build-backend>  # setuptools, hatchling, flit-core, poetry-core
  run:
    - python >=${{ python_min }}

tests:
  - python:
      python_version: ${{ python_min }}.*  # REQUIRED
      imports: [package]
      pip_check: true
```

**Local test:** `rattler-build build --recipe recipe/recipe.yaml --variant python_min=3.10`

### Python C-Extension

```yaml
build:
  skip:
    - if: python_version < "3.7"
      then: true

requirements:
  build:
    - if: build_platform != target_platform
      then:
        - python
        - cross-python_${{ target_platform }}
    - ${{ compiler('c') }}
    - ${{ stdlib('c') }}
  host:
    - python
    - pip
    - <build-backend>
  run:
    - python

tests:
  - python:
      imports: [package]
      pip_check: true
```

**Key differences:** 
- MUST have compiler/stdlib in build
- NOT noarch
- NO python_version in tests

**Local test:** `rattler-build build --recipe recipe/recipe.yaml --variant python=3.11`

## Workflow

### Setup

```bash
gh repo fork conda-forge/<package>-feedstock --clone=false
cd packages
git submodule add https://github.com/YOUR_USERNAME/<package>-feedstock.git
cd <package>-feedstock && git checkout -b recipe-v1
```

### Convert

1. Check build number: `grep "number:" recipe/meta.yaml`
2. Create `recipe.yaml`:
   - **Recommended:** `conda-recipe-manager convert recipe/meta.yaml > recipe/recipe.yaml`
   - **Alternative:** Manual creation from scratch (best for learning)
   - **Always:** Review and fix key ordering, selectors, test structure
3. **Bump build number** (if version unchanged)
4. Update `conda-forge.yml` (add these lines):
   ```yaml
   conda_build_tool: rattler-build
   conda_install_tool: pixi
   ```
5. Add `license_file: LICENSE` to `about:` section in recipe.yaml
6. Update python_min in context to a modern version (3.8+, NOT <3.6)

### Test & Submit

```bash
# Test locally first
rattler-build build --recipe recipe/recipe.yaml --variant python_min=3.10

# Commit ONLY the recipe files (NOT rerendered files)
git rm recipe/meta.yaml
git add recipe/recipe.yaml conda-forge.yml
git commit -m "Migrate to recipe.yaml (CEP 13/14)"
git push origin recipe-v1

# Create PR (bot will rerender automatically)
gh pr create --draft \
  --repo conda-forge/<package>-feedstock \
  --base main \
  --head YOUR_USERNAME:recipe-v1 \
  --title "Migrate to recipe.yaml (CEP 13/14)" \
  --body "Migrating to recipe.yaml format per CEP 13/14.

@conda-forge-admin, please rerender

Checklist:
- [x] Builds locally with rattler-build
- [x] Build number bumped
- [x] conda-forge.yml updated
- [x] Only recipe files committed (not rerendered files)"
```

### 🎯 Git Workflow Best Practices

**IMPORTANT:** Only commit recipe-related files, let the conda-forge bot handle rerendering:

**DO commit:**
- ✅ `recipe/recipe.yaml` (new recipe)
- ✅ `conda-forge.yml` (build tool config)
- ✅ Removal of `recipe/meta.yaml` (old recipe)

**DON'T commit:**
- ❌ `.ci_support/*` files (bot will regenerate)
- ❌ `.scripts/*` files (bot will regenerate)
- ❌ `.azure-pipelines/*` files (bot will regenerate)
- ❌ `README.md` (bot will update)
- ❌ Any other auto-generated files

**Why?** The conda-forge bot (`@conda-forge-admin, please rerender`) will automatically:
1. Generate correct CI configuration for rattler-build
2. Update all support files
3. Ensure consistency across the feedstock
4. Avoid merge conflicts with ongoing changes

**Workflow:**
1. Make recipe changes (recipe.yaml + conda-forge.yml)
2. Commit only those changes
3. Push to your fork
4. Create PR with `@conda-forge-admin, please rerender` in description
5. Bot handles all CI files automatically

### Triggering Bot Rerender

After pushing your changes, comment on the PR:
```
@conda-forge-admin, please rerender
```

The bot will:
- Generate all CI files for rattler-build
- Update `.ci_support/` configurations
- Create proper build scripts in `.scripts/`
- Update README and other metadata
- Commit everything back to your PR branch

**Note:** Wait for the bot before pushing additional commits to avoid conflicts.

## Compiling Native Code

### Patches
- Patches are **critical** - always verify they apply correctly
- Check patched files in build directory: `find output/bld -name "file.ext" -exec grep "patch marker" {} \;`
- Patches syntax: list of paths relative to recipe directory
  ```yaml
  source:
    patches:
      - patches/0001-fix.patch
      - patches/0002-other.patch
      - if: osx
        then:
          - patches/0003-apple.patch
  ```

### C-stdlib Variants
- Local builds may fail with "undefined" for `${{ stdlib('c') }}`
- **macOS:** `--variant c_stdlib=macosx_deployment_target --variant c_stdlib_version=11.0`
- **Linux:** `--variant c_stdlib=sysroot --variant c_stdlib_version=2.17`
- Or comment out stdlib line for local testing only

### Linux Testing (Docker)
- Use `mambaorg/micromamba:latest` for clean environment
- On ARM64 Mac, add `--platform linux/amd64` flag
  ```bash
  docker run --rm --platform linux/amd64 \
    -v "$(pwd)/packages/<pkg>-feedstock:/work" \
    mambaorg/micromamba:latest bash -c "
    micromamba install -y -n base -c conda-forge rattler-build && \
    cd /work && \
    rattler-build build --recipe recipe/recipe.yaml \
      --variant python=3.11 \
      --variant c_stdlib=sysroot \
      --variant c_stdlib_version=2.17
  "
  ```

### Platform-Specific Dependencies
- Use `if: not win` / `if: win` for platform-specific deps (libffi, patch tools)
- Cross-compilation needs `python` and `cross-python_${{ target_platform }}` in build section

## Troubleshooting

| Issue | Fix |
|-------|-----|
| `python_min` not substituted | Add `--variant python_min=3.10` locally |
| "undefined" in build (stdlib) | Provide `--variant c_stdlib=...` or comment for local test |
| Build number not bumped | Increment from meta.yaml value |
| Skip not working locally | Comment out for local testing |

## Resources

### Official Documentation
- [Rattler-Build Conversion Guide](https://rattler-build.prefix.dev/latest/converting_from_conda_build/) - **Official conversion documentation**
- [Progress Tracker](https://tdejager.github.io/are-we-recipe-v1-yet/) - Community progress
- [CEP 13](https://github.com/conda/ceps/blob/main/cep-0013.md) - YAML Syntax | [CEP 14](https://github.com/conda/ceps/blob/main/cep-0014.md) - Schema
- [Rattler-Build](https://github.com/prefix-dev/rattler-build) - Main repository

### Conversion Tools
- [conda-recipe-manager](https://github.com/conda-incubator/conda-recipe-manager) - Automatic conversion utility (recommended)

## Examples

**colorama** (noarch): Pure Python, straightforward conversion  
**wrapt** (C-ext): ⚠️ Blocked - Recipe works locally but CI infrastructure incomplete  
**cffi** (C-ext): FFI package with patches, compiler deps, platform-specific requirements

## Completed Migrations

✅ **pathspec** (forge-fix-bfc): Path pattern matching library  
✅ **wcwidth** (forge-fix-fix): Unicode display width calculation  
✅ **cloudpickle** (forge-fix-vj3): Extended pickle for distributed computing  
✅ **toolz** (forge-fix-bf7): Functional utilities  
✅ **semantic_version** (forge-fix-syr): Semantic versioning utilities (PR pending review)

## Next Targets

Simple noarch packages to practice:
- **python-dateutil** (forge-fix-eca): Date/time parsing
- **flake8** (forge-fix-dqu): Code linting
- **black** (forge-fix-7c4): Code formatter

---

**Last Updated:** 2025-12-05  
**Major Updates:**
- Added automatic conversion tools (conda-recipe-manager, feedrattler)
- Clarified git workflow: commit only recipe files, let bot handle rerendering
- Expanded syntax conversion reference with selectors and test examples
- Added official rattler-build conversion guide reference