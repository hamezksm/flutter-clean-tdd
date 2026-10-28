# flutter-clean-tdd

A Claude skill for building Flutter apps with Clean Architecture and strict TDD.

**Stack:** Riverpod (state + DI) · flutter_hooks (no StatefulWidgets) · ObjectBox · Dio · go_router + go_router_builder · freezed + json_serializable · mocktail

## Install

### Claude Code (personal, all projects)
```bash
git clone https://github.com/hamezksm/flutter-clean-tdd.git
mkdir -p ~/.claude/skills
cp -r flutter-clean-tdd/skills/flutter-clean-tdd ~/.claude/skills/
```

### Claude Code (one project, shared with the team via git)
```bash
mkdir -p .claude/skills
cp -r path/to/flutter-clean-tdd/skills/flutter-clean-tdd .claude/skills/
```

### skills CLI
```bash
npx skills add https://github.com/hamezksm/flutter-clean-tdd --skill flutter-clean-tdd
```

### Claude.ai
Zip the `skills/flutter-clean-tdd` folder and upload it under Settings → Capabilities → Skills.

## Recommended companion
For Apple-style UI principles (the skill translates them to Flutter):
```bash
npx skills add https://github.com/emilkowalski/skills --skill apple-design
```

## Structure
```
skills/flutter-clean-tdd/
  SKILL.md                 # rules + workflow
  references/              # loaded on demand
    architecture.md  state-and-widgets.md  models.md  data-layer.md
    routing.md  testing.md  dependencies.md  platform-setup.md  ui.md
```
