```bash
Edit release notes → Bump version.txt + pyproject.toml → Push to main
                          │
                          ▼
              release-trigger.yml creates tag
                          │
                ┌─────────┴─────────┐
                ▼                   ▼
      release-github.yml      release-pypi.yml
      (GitHub release +       (build + publish
       CHANGELOG update)       to PyPI)
```
