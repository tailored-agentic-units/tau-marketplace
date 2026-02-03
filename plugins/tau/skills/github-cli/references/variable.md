# gh variable

Variables are plaintext configuration values for GitHub Actions or Dependabot.

## Set

```bash
# Set from value
gh variable set MYVAR --body "some-value"

# Set for a deployment environment
gh variable set MYVAR --env staging --body "staging-value"

# Set organization variable visible to all repos
gh variable set MYVAR --org my-org --visibility all

# Set organization variable for specific repos
gh variable set MYVAR --org my-org --repos repo1,repo2 --visibility selected

# Bulk set from .env file
gh variable set -f .env
```

## Get

```bash
# Get repo variable
gh variable get MYVAR

# Get environment variable
gh variable get MYVAR --env staging

# JSON output
gh variable get MYVAR --json name,value,updatedAt
```

## List

```bash
# List repo variables
gh variable list

# List environment variables
gh variable list --env staging

# List organization variables
gh variable list --org my-org
```

## Delete

```bash
# Delete repo variable
gh variable delete MYVAR

# Delete environment variable
gh variable delete MYVAR --env staging

# Delete organization variable
gh variable delete MYVAR --org my-org
```
