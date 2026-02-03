# gh gist

## Create

```bash
# Create a gist from a file
gh gist create myfile.go --desc "Example code"

# Create public gist
gh gist create myfile.go --public

# Create from stdin
echo "code snippet" | gh gist create --filename snippet.go
```

## Multi-File Gists

```bash
# Create a gist with multiple files
gh gist create file1.go file2.go file3_test.go --desc "Multi-file example"

# Create multi-file gist from different directories
gh gist create src/main.go src/util.go README.md --public --desc "Project snippet"
```

## List

```bash
# List your gists
gh gist list

# List only public gists
gh gist list --public

# List only secret (private) gists
gh gist list --secret

# Limit results
gh gist list --limit 10
```

## View

```bash
# View a gist
gh gist view <gist-id>

# View a specific file in a gist
gh gist view <gist-id> --filename main.go

# View raw content (no decoration)
gh gist view <gist-id> --raw

# Open gist in browser
gh gist view <gist-id> --web
```

## Edit

```bash
# Edit a gist (opens editor)
gh gist edit <gist-id>

# Add a file to an existing gist
gh gist edit <gist-id> --add newfile.go

# Update a specific file in a gist from local file
gh gist edit <gist-id> --filename main.go mylocal.go

# Update gist description
gh gist edit <gist-id> --desc "Updated description"
```

## Delete

```bash
# Delete a gist
gh gist delete <gist-id>
```

## Clone

```bash
# Clone a gist to local directory
gh gist clone <gist-id>

# Clone to a specific directory
gh gist clone <gist-id> my-gist-dir
```
