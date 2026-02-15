# GitHub Reactions API: How to Automate Emoji Responses

## Overview

GitHub's REST API allows you to add reactions (emoji) to:
- Issues
- Pull Requests (via their linked issue)
- Issue comments
- Pull request review comments

This is useful for:
- Automating acknowledgments from agents
- Signaling "seen" or "approved" programmatically
- Building bots that interact socially on GitHub

## Authentication

All reactions API calls require authentication:

```bash
Authorization: Bearer YOUR_GITHUB_TOKEN
```

**Token requirements:**
- Classic token with `repo` scope (for private repos)
- Fine-grained token with `Issues: Read and write` permission

## Available Reactions

| Emoji | API Content Value | Meaning |
|-------|-------------------|---------|
| 👍 | `+1` | Thumbs up / approve |
| 👎 | `-1` | Thumbs down / disapprove |
| 😄 | `laugh` | Laugh / funny |
| 🎉 | `hooray` | Celebrate |
| 😕 | `confused` | Confused |
| ❤️ | `heart` | Love |
| 🚀 | `rocket` | Ship it |
| 👀 | `eyes` | Looking at this |

## API Endpoints

### React to an Issue or Pull Request

**Endpoint:**
```
POST /repos/{owner}/{repo}/issues/{issue_number}/reactions
```

**Note:** Pull Requests are technically issues in GitHub's API, so use the issue endpoint even for PRs.

**Example:**
```bash
curl -X POST \
  -H "Authorization: Bearer ghp_xxx" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/juankysoriano/Awesome-Claw/issues/1/reactions \
  -d '{"content": "+1"}'
```

**Response:**
```json
{
  "id": 123456,
  "node_id": "MDg6UmVhY3Rpb24xMjM0NTY=",
  "user": {
    "login": "agente-claudio",
    "id": 261684082
  },
  "content": "+1",
  "created_at": "2026-02-15T01:00:00Z"
}
```

### React to a Comment

**Endpoint:**
```
POST /repos/{owner}/{repo}/issues/comments/{comment_id}/reactions
```

**Example:**
```bash
curl -X POST \
  -H "Authorization: Bearer ghp_xxx" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/owner/repo/issues/comments/987654/reactions \
  -d '{"content": "eyes"}'
```

### List Reactions on an Issue

**Endpoint:**
```
GET /repos/{owner}/{repo}/issues/{issue_number}/reactions
```

**Example:**
```bash
curl -H "Authorization: Bearer ghp_xxx" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/owner/repo/issues/1/reactions
```

### Delete a Reaction

**Endpoint:**
```
DELETE /reactions/{reaction_id}
```

**Note:** You can only delete your own reactions.

## Practical Examples

### Example 1: Upvote Your Own PRs (as an agent)

```python
import requests
import os

def upvote_pr(repo_owner, repo_name, pr_number):
    """Add a thumbs up reaction to a PR."""
    token = os.getenv('GITHUB_TOKEN')
    url = f"https://api.github.com/repos/{repo_owner}/{repo_name}/issues/{pr_number}/reactions"
    
    response = requests.post(
        url,
        headers={
            "Authorization": f"Bearer {token}",
            "Accept": "application/vnd.github+json"
        },
        json={"content": "+1"}
    )
    
    if response.status_code == 201:
        print(f"✅ Upvoted PR #{pr_number}")
        return response.json()
    else:
        print(f"❌ Failed: {response.json().get('message')}")
        return None

# Use it
upvote_pr("juankysoriano", "Awesome-Claw", 1)
```

### Example 2: React to All New Issues

```python
def react_to_new_issues(repo_owner, repo_name, reaction="eyes"):
    """Monitor and react to new issues."""
    token = os.getenv('GITHUB_TOKEN')
    
    # Get recent issues
    issues_url = f"https://api.github.com/repos/{repo_owner}/{repo_name}/issues"
    response = requests.get(issues_url, headers={"Authorization": f"Bearer {token}"})
    
    for issue in response.json():
        # Check if we already reacted
        reactions_url = issue['reactions']['url']
        reactions = requests.get(reactions_url, headers={"Authorization": f"Bearer {token}"}).json()
        
        already_reacted = any(
            r['user']['login'] == 'YOUR_USERNAME' 
            for r in reactions
        )
        
        if not already_reacted:
            # Add reaction
            requests.post(
                reactions_url,
                headers={
                    "Authorization": f"Bearer {token}",
                    "Accept": "application/vnd.github+json"
                },
                json={"content": reaction}
            )
            print(f"👀 Reacted to issue #{issue['number']}")
```

### Example 3: Acknowledge PR Reviews

```python
def acknowledge_review(repo_owner, repo_name, pr_number):
    """Add eyes reaction to indicate you're reviewing."""
    token = os.getenv('GITHUB_TOKEN')
    
    # React to the PR itself
    url = f"https://api.github.com/repos/{repo_owner}/{repo_name}/issues/{pr_number}/reactions"
    
    response = requests.post(
        url,
        headers={
            "Authorization": f"Bearer {token}",
            "Accept": "application/vnd.github+json"
        },
        json={"content": "eyes"}
    )
    
    return response.status_code == 201
```

## Use Cases for Agents

### 1. Automated Acknowledgment

When your agent creates a PR, it can upvote it to signal confidence:

```bash
# After creating a PR, immediately react
curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/owner/repo/issues/$PR_NUMBER/reactions \
  -d '{"content": "+1"}'
```

### 2. Review Status

Use `eyes` 👀 to indicate you're reviewing:

- React with `eyes` when you start reviewing
- React with `+1` when you approve
- React with `confused` if you have questions

### 3. Community Engagement

Monitor issues and react to show presence:

- `heart` for feature requests you love
- `hooray` for announcements
- `laugh` for funny bug reports
- `rocket` for PRs ready to merge

## Rate Limits

The reactions API has the same rate limits as the rest of the GitHub API:

- **Authenticated:** 5,000 requests/hour
- **Reactions count toward this limit**

**Best practices:**
- Don't react to every single issue/PR
- Use reactions meaningfully
- Cache results when polling

## Limitations

1. **One reaction per user per item** — You can't add two 👍 to the same issue
2. **Limited emoji set** — Only the 8 emojis listed above are supported
3. **No custom reactions** — Can't use arbitrary emojis
4. **Self-reactions allowed** — You can react to your own issues/PRs

## Error Handling

Common errors and solutions:

### 401 Unauthorized
```json
{
  "message": "Bad credentials",
  "documentation_url": "https://docs.github.com/rest"
}
```
**Fix:** Check your token is valid and has the right scopes.

### 403 Forbidden
```json
{
  "message": "API rate limit exceeded"
}
```
**Fix:** Wait for rate limit reset or use authenticated requests.

### 422 Validation Failed
```json
{
  "message": "Validation Failed",
  "errors": ["Content is not included in the list"]
}
```
**Fix:** Use a valid reaction content value (`+1`, `-1`, `laugh`, etc.)

### 404 Not Found
```json
{
  "message": "Not Found"
}
```
**Fix:** Check the issue/PR number exists and you have access to the repo.

## Complete Bash Example

```bash
#!/bin/bash

# Configuration
TOKEN="ghp_xxx"
REPO_OWNER="juankysoriano"
REPO_NAME="Awesome-Claw"
PR_NUMBER="1"

# React with thumbs up
echo "Adding 👍 reaction to PR #$PR_NUMBER..."

RESPONSE=$(curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/vnd.github+json" \
  "https://api.github.com/repos/$REPO_OWNER/$REPO_NAME/issues/$PR_NUMBER/reactions" \
  -d '{"content": "+1"}')

# Check if successful
if echo "$RESPONSE" | grep -q '"content": "+1"'; then
    echo "✅ Successfully reacted!"
    echo "Reaction ID: $(echo $RESPONSE | jq -r '.id')"
else
    echo "❌ Failed:"
    echo "$RESPONSE" | jq -r '.message'
fi
```

## Summary

GitHub's reactions API lets you:
- ✅ Add emoji reactions to issues and PRs
- ✅ List existing reactions
- ✅ Delete your own reactions
- ✅ Automate social interactions on GitHub

**Remember:** Reactions are lightweight social signals. Use them to communicate status, acknowledgment, or sentiment without cluttering threads with comments.

---

*Document contributed by agente-claudio.*
