# Skill: GitHub API Integration

## Purpose
Automate GitHub PR creation using REST API instead of CLI.

## When to Use
After branch is pushed and ready for PR creation. Useful when GitHub CLI isn't available.

## GitHub API: Create Pull Request

### Endpoint
```
POST /repos/{owner}/{repo}/pulls
```

### Authentication
Requires GitHub Personal Access Token with `repo` scope.

### Request Body
```json
{
  "title": "string",
  "body": "string",
  "head": "string (branch name)",
  "base": "string (target branch)",
  "draft": false,
  "maintainer_can_modify": true
}
```

### Example for Issue #1
```json
{
  "title": "[DOCS] Improve documentation for System::total_memory() (#1)",
  "body": "## Description\nClarify that `total_memory()` returns the TOTAL system memory (RAM).\n\n## Related Issue\nCloses #1\n\n## Type of Change\n- [x] Documentation update\n\n## Changes Made\n- Updated doc comment for `System::total_memory()` method\n- Changed: \"Returns the RAM size in bytes.\"\n- To: \"Returns the total system memory (RAM) in bytes.\"",
  "head": "fix/1-improve-total-memory-doc",
  "base": "main",
  "draft": false,
  "maintainer_can_modify": true
}
```

## Using Invoke-WebRequest (PowerShell)

```powershell
$token = "github_pat_YOUR_TOKEN_HERE"
$headers = @{
    Authorization = "Bearer $token"
    "X-GitHub-Api-Version" = "2022-11-28"
    Accept = "application/vnd.github+json"
}

$body = @{
    title = "[DOCS] Improve documentation for System::total_memory() (#1)"
    body = "Clarify that total_memory() returns TOTAL system memory (RAM).`n`nCloses #1"
    head = "fix/1-improve-total-memory-doc"
    base = "main"
    draft = $false
    maintainer_can_modify = $true
} | ConvertTo-Json

$response = Invoke-WebRequest `
    -Uri "https://api.github.com/repos/ssiddarthapexon/sysinfo/pulls" `
    -Method POST `
    -Headers $headers `
    -Body $body `
    -ContentType "application/json"

if ($response.StatusCode -eq 201) {
    $pr = $response.Content | ConvertFrom-Json
    Write-Host "✓ PR #$($pr.number) created: $($pr.html_url)"
} else {
    Write-Host "✗ Error: $($response.StatusCode)"
}
```

## How to Get GitHub Token
1. Go to: https://github.com/settings/tokens
2. Click "Generate new token (classic)"
3. Scopes: check `repo` only
4. Copy token and save securely

## API Response
Success (201):
```json
{
  "url": "https://api.github.com/repos/ssiddarthapexon/sysinfo/pulls/2",
  "id": 12345,
  "number": 2,
  "title": "[DOCS] Improve documentation...",
  "html_url": "https://github.com/ssiddarthapexon/sysinfo/pull/2",
  "state": "open"
}
```

## Alternative: GitHub Actions Workflow
Create `.github/workflows/auto-pr.yml` to trigger from chat.

See: `.github/workflows/auto-pr.yml`
