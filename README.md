# forge-fix

Tracking and fixing conda-forge recipes for the migration from classic format (`meta.yaml`) to the new recipe format (`recipe.yaml`) using `rattler-build`.

## 🎯 Mission

Help conda-forge migrate from the classic recipe format to the new CEP 13/14 recipe format by:

- 📦 Tracking packages that need migration
- 🔧 Testing recipes with `rattler-build`
- 🐛 Identifying and fixing compatibility issues
- 🤝 Contributing fixes back to upstream feedstocks
- 📊 Monitoring progress across the ecosystem

## 🔗 Key Resources

- **Progress Tracker**: https://tdejager.github.io/are-we-recipe-v1-yet/
- **CEP 13** (YAML Syntax): https://github.com/conda/ceps/blob/main/cep-0013.md
- **CEP 14** (Schema Definition): https://github.com/conda/ceps/blob/main/cep-0014.md
- **Rattler-Build**: https://github.com/prefix-dev/rattler-build

## 🚀 Quick Start

### Prerequisites

```bash
# Install rattler-build
pixi global install rattler-build

# Install beads for issue tracking
curl -sSL https://raw.githubusercontent.com/steveyegge/beads/main/scripts/install.sh | bash
```

### Working on a Package

```bash
# 1. Create a tracking issue
bd create "Migrate <package> to recipe v1"

# 2. Fetch the feedstock
./scripts/fetch_feedstock.sh conda-forge/<package>-feedstock

# 3. Convert the recipe
python scripts/convert_recipe.py recipes/<package>/meta.yaml

# 4. Test with rattler-build
./scripts/test_build.sh recipes/<package>/recipe.yaml

# 5. Update progress
bd update <issue-id> --status done
bd sync
```

## 📁 Repository Structure

```
forge-fix/
├── .beads/              # Beads issue tracking (git-integrated)
├── memory/              # Knowledge repository
│   ├── projects/        # Per-package migration tracking
│   ├── issues/          # Recurring problem patterns
│   └── solutions/       # Proven solution templates
├── recipes/             # Test recipes being migrated
├── scripts/             # Automation tools
└── docs/                # Documentation
```

## 🧠 Two-Tier Tracking System

This project uses a complementary tracking system:

### 1. **Beads** - Workflow & Task Management

Git-integrated issue tracking for active work:

```bash
# Create migration tasks
bd create "Migrate xtensor to recipe v1"

# Track progress
bd update <id> --status in-progress
bd update <id> --status done

# View all tasks
bd list
```

### 2. **Memory** - Knowledge Repository

Structured YAML files capturing reusable knowledge:

- `memory/projects/` - Package-specific analysis and history
- `memory/issues/` - Recurring problem patterns
- `memory/solutions/` - Proven migration solutions

**Example**: When you solve a CMake path issue in `xtensor`, document the pattern in `memory/solutions/` so the next person working on `xsimd` can reuse it!

## 🔄 Migration Workflow

### For a Single Package

1. **Discover** - Check tracker, create Beads issue, check memory
2. **Analyze** - Download feedstock, review `meta.yaml`, identify patterns
3. **Convert** - Transform to `recipe.yaml` following CEP 13/14
4. **Test** - Build with `rattler-build`, run tests
5. **Document** - Update memory with lessons learned
6. **Contribute** - Submit PR to upstream feedstock

### Handling Blockers

When you hit a problem:

```bash
# 1. Mark task as blocked
bd update <id> --status blocked

# 2. Check for known patterns
grep -r "cmake_config" memory/issues/

# 3. If new pattern, document it
# Create memory/issues/new_pattern.yaml

# 4. When solved, create solution
# Create memory/solutions/new_pattern_fix.yaml

# 5. Update other affected packages
```

## 📊 Key Differences: Old vs New Format

### File Names
- ❌ Old: `meta.yaml`
- ✅ New: `recipe.yaml`

### Template Syntax
```yaml
# Old format (invalid YAML)
package:
  name: {{ name }}
  version: {{ version }}

# New format (valid YAML)
package:
  name: ${{ name }}
  version: ${{ version }}
```

### Selectors
```yaml
# Old format (comments)
dependencies:
  - make  # [unix]
  - m2-make  # [win]

# New format (YAML dictionaries)
dependencies:
  - if: unix
    then: make
  - if: win
    then: m2-make
```

### Context Section
```yaml
# Old format
{% set name = "package" %}
{% set version = "1.2.3" %}

# New format
context:
  name: package
  version: 1.2.3
```

## 🎯 Priority Packages

Start with simpler packages to build solution patterns:

1. **Header-only libraries** (xtensor, xsimd, xtl) - No compilation
2. **Pure Python packages** - Simple builds, good for entry points testing
3. **Single-output compiled libs** - Test compiler() and run_exports
4. **Multi-output packages** - Complex but important (mamba, libmamba)

## 🤝 Contributing

### For Humans

1. Pick a package from the tracker
2. Follow the migration workflow above
3. Update Beads issues and memory files
4. Submit PRs to upstream feedstocks
5. Share learnings in memory system

### For AI Agents

See **AGENTS.md** for detailed instructions on:
- Using Beads for task tracking
- Updating memory files
- Collaboration protocols
- Pattern recognition and reuse

## 📈 Success Metrics

Track progress via:

```bash
# Active tasks
bd list

# Completed migrations
bd list --status done

# Known patterns
ls memory/issues/ | wc -l

# Solution library
ls memory/solutions/ | wc -l
```

## 🔧 Scripts

### `scripts/fetch_feedstock.sh`
Downloads a conda-forge feedstock for local testing

### `scripts/convert_recipe.py`
Automated conversion attempt (handles common patterns)

### `scripts/test_build.sh`
Tests a recipe with rattler-build

### `scripts/find_similar.py`
Finds similar packages in memory for pattern matching

## 📚 Documentation

- **AGENTS.md** - Detailed instructions for AI agents
- **memory/README.md** - Memory system documentation
- **docs/migration_guide.md** - Step-by-step migration guide
- **docs/common_patterns.md** - Common migration patterns

## 🐛 Known Issues

Check `memory/issues/` for documented patterns:

```bash
# List all known issue patterns
ls memory/issues/

# Find high-frequency issues
grep -r "frequency: high" memory/issues/
```

## 📞 Getting Help

1. Check `memory/solutions/` for existing solutions
2. Search Beads issues: `bd list`
3. Review similar packages in `memory/projects/`
4. Consult CEP 13/14 documentation
5. Ask in conda-forge discussions

## 🙏 Acknowledgments

- **conda-forge community** - For the amazing package ecosystem
- **prefix.dev team** - For rattler-build and the new recipe format
- **CEP authors** - For designing the migration path

## 📜 License

This repository is for coordinating migration efforts. Individual recipes are subject to their original licenses in conda-forge feedstocks.

---

**Status**: 🚧 Active Development  
**Last Updated**: 2025-01-20  
**Maintainers**: Community contributors and AI agents