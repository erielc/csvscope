# Release Notes

This directory holds per-release notes that the release workflow consumes when creating GitHub releases.

## Workflow

1. Before bumping `version.txt`, create a new file here: `v<NEW_VERSION>.md`
2. Write the release notes following the template in `_template.md`
3. Update `version.txt` AND `pyproject.toml` to match the new version (both must agree)
4. Commit and push to `main`
5. The release-trigger workflow detects the `version.txt` change, validates everything, and creates a `v<NEW_VERSION>` tag
6. The tag push triggers two parallel workflows:
   - **GitHub Release**: creates a release using the notes file and attaches source archives
   - **PyPI Release**: builds and publishes to PyPI
7. The GitHub Release workflow also auto-updates `CHANGELOG.md` with the notes content

## Naming convention

- File name MUST match `v<version>.md` exactly (e.g., `v0.2.1.md`, `v1.0.0.md`)
- Pre-release versions: `v1.0.0-rc1.md`, `v1.0.0-beta.1.md`
- Version in filename must match `version.txt` exactly

## Format

Use markdown. Common sections:

- `## Highlights` — 1-3 sentence summary at top
- `## Added` — new features
- `## Changed` — non-breaking changes
- `## Fixed` — bug fixes
- `## Deprecated` — features marked for removal
- `## Removed` — features removed
- `## Security` — security fixes
- `## Breaking Changes` — for major releases
- `## Notes` — anything else important (Python version, migration steps)

See `_template.md` for a copyable starting point.

## What NOT to do

- Don't include the `## [version] - date` header — that's added by the workflow when updating CHANGELOG.md
- Don't reference internal PR numbers without links — users won't be able to find them easily
- Don't delete or rename files after release — they're the historical record
