# Release Checklist

> Check every item before each release. Version source of truth: the VERSION file must match the latest release version in CHANGELOG.md (automatically validated by the CI ersion-check job; a mismatch turns the run red).

## Release Steps

1. [ ] Confirm the [Unreleased] section of CHANGELOG.md is complete (Keep a Changelog groups: Added / Fixed / Security / Removed)
2. [ ] Change `## [Unreleased]` to `## [x.y.z] — YYYY-MM-DD` and update the compare links from `...HEAD` to the new tag
3. [ ] Update the VERSION file to x.y.z in sync
4. [ ] For milestone releases (e.g., v1.0.0 / v1.1.0), update docs/RELEASE_NOTES_v<x.y.z>.md in sync
5. [ ] Tag it: git tag v<x.y.z> + git push --tags
6. [ ] After pushing, confirm CI is fully green (routing 173-case baseline + coherence + pin gate + version-check)

## Metadata Sync (Release Housekeeping)

- Adding/removing bootstrap capabilities → sync the capability lists in RULES.md / skills/SKILL.md (with skills/scripts/bootstrap-manifest.json as the single source of truth, currently 25 items)
- Adding field-journal entries → update the three parts of skills/field-journal/_index.md (scenario categories / high-frequency patterns / entity inverted index) and the statistics
- Routing rule changes → modify only skills/config/routing.json (docs are maintained by the generation script, or at least kept consistent)

> Note: the <!-- [进化统计] --> cumulative comment at the bottom of journal entries is no longer maintained by hand (removed on 2026-08-10; the numbers cannot be reliably maintained); project counts follow _index.md.
