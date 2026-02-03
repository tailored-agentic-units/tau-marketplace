# gh secret

Secrets are encrypted values for GitHub Actions, Dependabot, or Codespaces.

## Set

```bash
# Set from prompt (interactive)
gh secret set MYSECRET

# Set from value
gh secret set MYSECRET --body "$ENV_VALUE"

# Set from file
gh secret set MYSECRET < credentials.txt

# Set for a deployment environment
gh secret set MYSECRET --env production --body "$VALUE"

# Set organization secret visible to all repos
gh secret set MYSECRET --org my-org --visibility all

# Set organization secret for specific repos
gh secret set MYSECRET --org my-org --repos repo1,repo2 --visibility selected

# Set for Dependabot
gh secret set MYSECRET --app dependabot --body "$VALUE"

# Bulk set from .env file
gh secret set -f .env
```

## List

```bash
# List repo secrets
gh secret list

# List environment secrets
gh secret list --env production

# List organization secrets
gh secret list --org my-org

# List Dependabot secrets
gh secret list --app dependabot
```

## Delete

```bash
# Delete repo secret
gh secret delete MYSECRET

# Delete environment secret
gh secret delete MYSECRET --env production

# Delete organization secret
gh secret delete MYSECRET --org my-org
```
