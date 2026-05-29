# Release Pipeline Setup Guide

How the release flow works for csvscope, and where each file belongs in your repo.

---

## File placement

```
csvscope/
├── version.txt                              ← single source of truth for version
├── pyproject.toml                           ← keep in sync with version.txt
├── CHANGELOG.md                             ← auto-updated by release workflow
├── .github/
│   ├── workflows/
│   │   ├── ci.yml                           ← runs on push/PR (tests, lint, type-check)
│   │   ├── release-trigger.yml              ← runs when version.txt changes, creates tag
│   │   ├── release-github.yml               ← runs on tag, creates GH release
│   │   └── release-pypi.yml                 ← runs on tag, publishes to PyPI
│   └── release-notes/
│       ├── README.md                        ← explains the directory
│       ├── _template.md                     ← copy this when adding a new release
│       └── v0.1.0.md                        ← one file per release
└── ...
```

---

## How a release happens (the flow)

```
1. Developer writes release notes
   └→ .github/release-notes/v0.2.0.md

2. Developer bumps version
   ├→ version.txt:        0.2.0
   └→ pyproject.toml:     version = "0.2.0"

3. Commit + push to main

4. release-trigger.yml fires (version.txt changed)
   ├→ Reads version.txt
   ├→ Validates pyproject.toml matches
   ├→ Validates .github/release-notes/v0.2.0.md exists and is non-empty
   ├→ Checks tag doesn't already exist
   └→ Creates and pushes tag v0.2.0

5. Tag push triggers TWO parallel workflows:

   ├─ release-github.yml
   │  ├→ Reads .github/release-notes/v0.2.0.md
   │  ├→ Creates source archive (.tar.gz, .zip)
   │  ├→ Creates GitHub Release with notes body
   │  ├→ Inserts notes into CHANGELOG.md under "## [Unreleased]"
   │  └→ Commits CHANGELOG update back to main
   │
   └─ release-pypi.yml
      ├→ Verifies tag matches version.txt and pyproject.toml
      ├→ Builds sdist and wheel
      ├→ Validates distributions with twine
      └→ Publishes to PyPI via trusted publishing
```

The two release workflows are independent. If PyPI fails, GitHub release still succeeds (and vice versa).

---

## Initial repo setup (one-time)

### 1. Create the directory structure

```bash
mkdir -p .github/workflows .github/release-notes
```

### 2. Place the workflow files

```bash
mv ci.yml                   .github/workflows/ci.yml
mv release-trigger.yml      .github/workflows/release-trigger.yml
mv release-github.yml       .github/workflows/release-github.yml
mv release-pypi.yml         .github/workflows/release-pypi.yml
```

### 3. Place release notes files

```bash
mv release-notes-README.md  .github/release-notes/README.md
mv _template.md             .github/release-notes/_template.md
mv v0.1.0.md                .github/release-notes/v0.1.0.md
```

### 4. Place top-level files

```bash
mv version.txt   version.txt
mv CHANGELOG.md  CHANGELOG.md
mv AGENT.md     AGENT.md
```

### 5. Configure PyPI trusted publishing

1. Go to https://pypi.org/manage/account/publishing/
2. Click "Add a new pending publisher"
3. Fill in:
   - PyPI Project Name: `csvscope`
   - Owner: your GitHub username/org
   - Repository name: `csvscope`
   - Workflow name: `release-pypi.yml`
   - Environment name: `pypi`

No API tokens needed — PyPI trusts GitHub Actions via OIDC.

### 6. Create the `pypi` environment in GitHub

1. Go to repo Settings → Environments
2. Click "New environment", name it `pypi`
3. Optional: add protection rules
   - Required reviewers (someone must approve PyPI publishes)
   - Wait timer (delay before publishing — useful for "oops, that was wrong")
   - Deployment branches: restrict to `main` only

### 7. (Optional) Set up branch protection on main

1. Go to repo Settings → Branches → Branch protection rules
2. Add rule for `main`:
   - Require pull request reviews before merging
   - Require status checks to pass (select CI workflow jobs)
   - Restrict who can push to matching branches

---

## How to cut your first release

```bash
# 1. Make sure you're on main and up to date
git checkout main
git pull

# 2. Write release notes for v0.1.0 (already done if you used the template)
# File: .github/release-notes/v0.1.0.md

# 3. Verify version.txt and pyproject.toml are 0.1.0
cat version.txt
grep "^version" pyproject.toml

# 4. Commit and push
git add version.txt pyproject.toml .github/release-notes/v0.1.0.md
git commit -m "release: v0.1.0"
git push origin main

# 5. Watch the workflows run
# Go to Actions tab on GitHub
```

After ~2 minutes you should see:
- Tag `v0.1.0` created
- GitHub release published at `releases/tag/v0.1.0`
- Package published to `pypi.org/project/csvscope`
- `CHANGELOG.md` updated with the v0.1.0 entry

---

## How to cut subsequent releases

For a v0.2.0 release:

```bash
# 1. Create release notes file
cp .github/release-notes/_template.md .github/release-notes/v0.2.0.md
# Edit v0.2.0.md to describe what changed

# 2. Bump version
echo "0.2.0" > version.txt
# Edit pyproject.toml to set version = "0.2.0"

# 3. Commit and push
git add version.txt pyproject.toml .github/release-notes/v0.2.0.md
git commit -m "release: v0.2.0"
git push origin main
```

That's it. Workflows handle the rest.

---

## Troubleshooting

**"Release notes file does not exist"**

You bumped `version.txt` without creating `.github/release-notes/v<version>.md` first. Create the file and push another commit.

**"version.txt does not match pyproject.toml"**

The two are out of sync. Fix one to match the other and push again.

**"Tag already exists"**

Someone (or a previous workflow run) already created the tag. Either:
- Bump to the next version
- Delete the tag manually: `git push --delete origin v<version>` (use with care)

**PyPI publish fails with "trusted publisher not configured"**

You haven't set up PyPI trusted publishing yet. See step 5 above.

**PyPI publish fails with "filename already exists"**

PyPI doesn't allow re-uploading the same version. You must bump the version and release again.

**CHANGELOG.md commit fails to push**

Branch protection might require PR review. Either:
- Add `github-actions[bot]` to bypass list
- Manually merge the CHANGELOG update

---

## Versioning rules

This project uses [Semantic Versioning](https://semver.org/):

- **MAJOR.MINOR.PATCH** (e.g., `1.2.3`)
- **PATCH** (1.2.3 → 1.2.4): bug fixes, no API changes
- **MINOR** (1.2.3 → 1.3.0): new features, backwards-compatible
- **MAJOR** (1.2.3 → 2.0.0): breaking changes

While in `0.x.y`:
- Anything goes; API is not stable
- Bump MINOR for new features OR breaking changes
- Bump PATCH for fixes only

Pre-releases:
- `1.0.0-rc.1`, `1.0.0-beta.1`, `1.0.0-alpha.1`
- Workflow auto-detects `-` in tag and marks as prerelease on GitHub
