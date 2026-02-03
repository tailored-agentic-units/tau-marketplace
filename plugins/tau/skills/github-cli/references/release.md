# gh release

## Create

```bash
# Create release from tag
gh release create v1.0.0 --title "v1.0.0" --notes "Initial release"

# Generate release notes automatically
gh release create v1.0.0 --generate-notes

# Draft release
gh release create v1.0.0 --draft --title "v1.0.0 Release Candidate"

# With HEREDOC notes
gh release create v1.0.0 --title "v1.0.0" --notes "$(cat <<'EOF'
## What's New
- Feature A
- Feature B

## Bug Fixes
- Fixed issue #42
EOF
)"

# Upload release assets
gh release create v1.0.0 ./dist/binary-linux ./dist/binary-darwin
```

## List

```bash
# List releases
gh release list

# JSON output
gh release list --json tagName,publishedAt,isPrerelease
```

## View

```bash
gh release view v1.0.0

# View the latest release
gh release view --json tagName,publishedAt
```

## Delete

```bash
# Delete a release
gh release delete v1.0.0 --yes

# Delete release and its git tag
gh release delete v1.0.0 --yes --cleanup-tag
```
