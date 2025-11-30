# AGENTS.md - Forge Fix Project

> **Mission:** Migrate conda-forge from meta.yaml to recipe.yaml (CEP 13/14 v1) using rattler-build.

## ⚠️ Known Blocker

**C-extension packages:** Rattler-build CI support in conda-smithy is incomplete. When `conda_build_tool: rattler-build` is set, rerender deletes .ci_support files but doesn't generate replacement CI workflows. **Focus on noarch Python packages only** until tooling matures. See issue: forge-fix-l3s

## Quick Reference

### Syntax Changes

| Old (meta.yaml) | New (recipe.yaml) |
|-----------------|-------------------|
| `{{ var }}` | `${{ var }}` |
| `{% set name = "pkg" %}` | `context: { name: pkg }` |
| `# [unix]` selector | `if: unix` + `then:` |
| `home`, `doc_url`, `dev_url` | `homepage`, `documentation`, `repository` |

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
2. Create `recipe.yaml` with correct key order
3. **Bump build number** (if version unchanged)
4. Update `conda-forge.yml`:
   ```yaml
   conda_build_tool: rattler-build
   conda_install_tool: pixi
   ```

### Test & Submit

```bash
conda-smithy lint recipe
rattler-build build --recipe recipe/recipe.yaml --variant python_min=3.10  # or python=3.11

git rm recipe/meta.yaml
conda-smithy rerender --commit auto
git add recipe/recipe.yaml conda-forge.yml
git commit -m "Migrate to recipe.yaml (CEP 13/14)"
git push origin recipe-v1

# Draft PR for review
gh pr create --draft \
  --repo conda-forge/<package>-feedstock \
  --base main \
  --head YOUR_USERNAME:recipe-v1 \
  --title "[WIP] Migrate to recipe.yaml (CEP 13/14)" \
  --body "Draft PR for recipe.yaml migration. Feedback welcome!"
```

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

- [Progress Tracker](https://tdejager.github.io/are-we-recipe-v1-yet/)
- [CEP 13](https://github.com/conda/ceps/blob/main/cep-0013.md) | [CEP 14](https://github.com/conda/ceps/blob/main/cep-0014.md)
- [Rattler-Build](https://github.com/prefix-dev/rattler-build)

## Examples

**colorama** (noarch): Pure Python, straightforward conversion  
**wrapt** (C-ext): ⚠️ Blocked - Recipe works locally but CI infrastructure incomplete  
**cffi** (C-ext): FFI package with patches, compiler deps, platform-specific requirements

## Next Targets

Simple noarch packages to practice:
- **semantic-version** (forge-fix-syr): Semantic versioning utilities
- **cloudpickle** (forge-fix-vj3): Extended pickle for distributed computing
- **toolz** (forge-fix-bf7): Functional utilities
- **python-dateutil** (forge-fix-eca): Date/time parsing

---

**Last Updated:** 2025-01-30