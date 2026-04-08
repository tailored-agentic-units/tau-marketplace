# gh workflow / gh run

## Workflows

```bash
# List workflows
gh workflow list

# Trigger a workflow
gh workflow run <workflow-name>
```

## Trigger with Inputs

```bash
# Trigger workflow with input parameters
gh workflow run <workflow-name> -f param1=value1 -f param2=value2

# Trigger on a specific branch
gh workflow run <workflow-name> --ref feature-branch

# Trigger with JSON inputs
gh workflow run <workflow-name> --json <<< '{"param1":"value1","param2":"value2"}'
```

## Runs

```bash
# View recent runs
gh run list --limit 10

# View specific run
gh run view <run-id>

# Watch a run in progress
gh run watch <run-id>

# Download run artifacts
gh run download <run-id>
```

## View Workflow Logs

```bash
# View logs for a specific run
gh run view <run-id> --log

# View logs for a failed run only
gh run view <run-id> --log-failed

# View logs for a specific job within a run
gh run view <run-id> --job <job-id> --log
```

## Rerun Failed Jobs

```bash
# Rerun all failed jobs in a run
gh run rerun <run-id> --failed

# Rerun an entire run (all jobs)
gh run rerun <run-id>

# Rerun with debug logging enabled
gh run rerun <run-id> --debug
```

## Cancel Running Workflow

```bash
# Cancel a running workflow
gh run cancel <run-id>
```

## List Runs with Filters

```bash
# List runs for a specific workflow
gh run list --workflow <workflow-name>

# List only failed runs
gh run list --status failure --limit 10

# List runs on a specific branch
gh run list --branch main --limit 10

# List runs with JSON output
gh run list --json databaseId,status,conclusion,headBranch,createdAt --limit 10

# List runs filtered by event type
gh run list --event push --limit 10

# List runs for a specific user
gh run list --user username --limit 10
```
