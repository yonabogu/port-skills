# Port GitHub Integration — Reference

## Pull Request Mapping with itemsToParse

If you want each PR label as a separate entity:

```yaml
- kind: pull-request
  port:
    itemsToParse: .labels
    entity:
      mappings:
        identifier: '.base.repo.name + "-label-" + .item.name'
        title: .item.name
        blueprint: '"prLabel"'
        properties:
          color: .item.color
          repo: .base.repo.name
          pr_number: .number
```

## Required GitHub Permissions (OAuth App / GitHub App)

Port's GitHub integration needs:
- `repo` — read repos, branches, contents
- `pull_requests` — read PRs
- `actions` — read/trigger workflows (for action backend)
- `members` — read org teams

## Useful mapping transforms

```yaml
# Normalize PR state
status: 'if .state == "open" then "open" elif .merged_at != null then "merged" else "closed" end'

# Count changed files
changed_files: .changed_files

# PR age in days (Port JQ is UTC)
# Use a calculation property for this — mapper doesn't support now easily
```
