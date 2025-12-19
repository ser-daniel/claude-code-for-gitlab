# Claude Code for GitLab - Authentication Guide

Complete guide to understanding and configuring authentication for Claude Code in GitLab CI/CD.

## Overview

Claude Code for GitLab requires **TWO separate authentication tokens** for different purposes:

1. **GitLab API Access** - To read/write code, comments, and repository data
2. **Claude/Anthropic API Access** - To get AI responses from Claude

## Required CI/CD Variables

### 1. CLAUDE_CODE_GL_ACCESS_TOKEN (GitLab Personal Access Token)

**Purpose:** Authenticates with GitLab API for repository operations

**What it's used for:**
- Reading repository code and files
- Creating/updating merge requests
- Posting comments on issues and MRs
- Pushing changes to branches
- Checking user permissions

**How to get it:**
1. Go to GitLab → User Settings → Access Tokens
   - For agenticlab.tech: https://git.agenticlab.tech/-/user_settings/personal_access_tokens
2. Create a new token with these scopes:
   - ✅ `api` - Full API access
   - ✅ `read_repository` - Read repository content
   - ✅ `write_repository` - Write repository content
3. Copy the token (starts with `glpat-`)

**Example value:** `glpat-Ykn9BATOykcTLu6IDLpJ1m86MQp1OjMH.01.0w1pbijb5`

**Add to GitLab CI/CD:**
- Key: `CLAUDE_CODE_GL_ACCESS_TOKEN`
- Value: Your GitLab Personal Access Token
- Protected: ❌ NO (must work on MR pipelines!)
- Masked: ✅ YES
- Expand variable reference: ✅ YES

---

### 2. CLAUDE_CODE_OAUTH_TOKEN (Claude/Anthropic Token)

**Purpose:** Authenticates with Anthropic API to get Claude AI responses

**What it's used for:**
- Calling Claude AI API to generate responses
- Processing user requests with AI
- Getting code suggestions and reviews

**How to get it:**
1. Go to https://claude.ai/settings
2. Find "API Keys" or "OAuth Tokens" section
3. Create a new token for GitLab integration
4. Copy the token (starts with `sk-ant-`)

**Alternative:** Use `ANTHROPIC_API_KEY` instead if you have an Anthropic API key

**Add to GitLab CI/CD:**
- Key: `CLAUDE_CODE_OAUTH_TOKEN`
- Value: Your Claude Code OAuth token
- Protected: ❌ NO (must work on MR pipelines!)
- Masked: ✅ YES
- Expand variable reference: ✅ YES

---

## Authentication Flow in CI/CD

When a CI/CD pipeline runs, the authentication works like this:

```
┌─────────────────────────────────────────────────────┐
│ GitLab CI/CD Pipeline Starts                        │
└─────────────────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────┐
│ Step 1: prepare.ts                                  │
│ • Uses: CLAUDE_CODE_GL_ACCESS_TOKEN                 │
│ • Purpose: Check permissions, create tracking       │
│            comment on MR/issue                      │
│ • API: GitLab API                                   │
└─────────────────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────┐
│ Step 4: base-action (Claude Code execution)        │
│ • Uses: CLAUDE_CODE_OAUTH_TOKEN                     │
│ • Purpose: Call Claude AI to process request        │
│ • API: Anthropic API (through proxy)                │
└─────────────────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────┐
│ Step 5: update-comment-gitlab.ts                    │
│ • Uses: CLAUDE_CODE_GL_ACCESS_TOKEN                 │
│ • Purpose: Update comment with results              │
│ • API: GitLab API                                   │
└─────────────────────────────────────────────────────┘
```

## GitLab Token Priority Order

The code checks for GitLab authentication in this priority order:

1. **CLAUDE_CODE_GL_ACCESS_TOKEN** ← **Recommended** (highest priority)
2. **CLAUDE_CODE_OAUTH_TOKEN** (fallback - should NOT be used for GitLab!)
3. **GITLAB_TOKEN** (fallback)
4. **CI_JOB_TOKEN** (built-in, but has limited permissions)

**Important:** Always use `CLAUDE_CODE_GL_ACCESS_TOKEN` to avoid confusion!

## What About GitLab OAuth App Credentials?

You may have created a GitLab OAuth application with these credentials:
- `GITLAB_APP_ID` (e.g., `0096ebff89d9...`)
- `GITLAB_APP_SECRET` (e.g., `gloas-61d52ba1...`)

**These are NOT used in CI/CD pipelines!**

GitLab OAuth app credentials are only used by:
- The webhook service for OAuth authentication flows
- Web-based applications that need users to authenticate via OAuth

They should be set in the webhook service `.env` file, not in GitLab CI/CD variables.

## Common Mistakes

### ❌ Wrong: Using OAuth app credentials in CI/CD
```yaml
variables:
  GITLAB_OAUTH_APP_ID: $GITLAB_OAUTH_APP_ID
  GITLAB_OAUTH_APP_SECRET: $GITLAB_OAUTH_APP_SECRET
```
These don't work in CI/CD pipelines!

### ❌ Wrong: Using Claude token for GitLab auth
```yaml
variables:
  GITLAB_TOKEN: $CLAUDE_CODE_OAUTH_TOKEN
```
Claude token is for Anthropic API, not GitLab!

### ✅ Correct: Using proper tokens
```yaml
variables:
  CLAUDE_CODE_GL_ACCESS_TOKEN: $CLAUDE_CODE_GL_ACCESS_TOKEN  # GitLab API
  CLAUDE_CODE_OAUTH_TOKEN: $CLAUDE_CODE_OAUTH_TOKEN          # Anthropic API
```

## Troubleshooting

### Error: "Unauthorized" from GitLab API

**Symptom:**
```
GitbeakerRequestError: Unauthorized
prepare_error=Actor does not have write permissions to the repository
```

**Cause:** Missing or invalid `CLAUDE_CODE_GL_ACCESS_TOKEN`

**Fix:**
1. Verify the token is set in GitLab CI/CD variables
2. Check the token has the required scopes (api, read_repository, write_repository)
3. Make sure the token hasn't expired
4. Ensure "Protected" is unchecked (or branch is protected)

### Error: "Using Claude Code OAuth token for GitLab authentication"

**Symptom:**
```
Using Claude Code OAuth token for GitLab authentication
Token prefix: sk-ant-o...
GitbeakerRequestError: Unauthorized
```

**Cause:** `CLAUDE_CODE_GL_ACCESS_TOKEN` is not set, so the code falls back to using `CLAUDE_CODE_OAUTH_TOKEN` for GitLab auth (which fails because it's an Anthropic token)

**Fix:** Add `CLAUDE_CODE_GL_ACCESS_TOKEN` with a GitLab Personal Access Token

### Debug Output

When `prepare.ts` runs, it shows which tokens are set:

```
=== GitLab Environment Variables Debug ===
CLAUDE_CODE_GL_ACCESS_TOKEN: Set (length: 42, prefix: "glpat-XX...")  ✅ Good!
CLAUDE_CODE_OAUTH_TOKEN: Set (length: 108, prefix: "sk-ant-o...")     ✅ Good!
GITLAB_TOKEN: Not set                                                  ℹ️ Optional
CI_JOB_TOKEN: Set (length: 772, prefix: "glcbt-...")                  ℹ️ Built-in
=========================================
```

Look for this output in your CI job logs to verify tokens are set correctly.

## Security Best Practices

1. **Never commit tokens to git**
   - Use GitLab CI/CD variables, not hardcoded values
   - Check `.env` files are in `.gitignore`

2. **Use masked variables**
   - Both tokens should have "Masked" enabled in GitLab

3. **Don't mark as Protected**
   - Protected variables only work on protected branches
   - Claude Code needs to work on MR pipelines from any branch

4. **Regular token rotation**
   - Rotate GitLab Personal Access Tokens periodically
   - Update CI/CD variables when rotating

5. **Minimal scopes**
   - Only grant necessary scopes to tokens
   - GitLab token: `api`, `read_repository`, `write_repository`

## Quick Setup Checklist

For agenticlab.tech setup:

- [ ] Create GitLab Personal Access Token
  - Go to: https://git.agenticlab.tech/-/user_settings/personal_access_tokens
  - Scopes: api, read_repository, write_repository
  - Copy token: `glpat-...`

- [ ] Get Claude Code OAuth token
  - Go to: https://claude.ai/settings
  - Create token for GitLab integration
  - Copy token: `sk-ant-...`

- [ ] Add to GitLab CI/CD variables
  - Go to: https://git.agenticlab.tech/insurance-assistant/aegra-langgraph/-/settings/ci_cd
  - Add `CLAUDE_CODE_GL_ACCESS_TOKEN` = `glpat-...` (Protected: NO, Masked: YES)
  - Add `CLAUDE_CODE_OAUTH_TOKEN` = `sk-ant-...` (Protected: NO, Masked: YES)

- [ ] Test pipeline
  - Comment `@claude hello` on a merge request
  - Check job logs for successful authentication
  - Verify both tokens show as "Set" in debug output

## Related Documentation

- [AGENTICLAB_SETUP.md](./AGENTICLAB_SETUP.md) - Complete setup guide
- [TROUBLESHOOTING_VARS.md](./TROUBLESHOOTING_VARS.md) - Variable troubleshooting
- [docs/GITLAB_CLAUDE_EXECUTION_GUIDE.md](./docs/GITLAB_CLAUDE_EXECUTION_GUIDE.md) - Execution details
- [docs/GITLAB_TOKEN_TROUBLESHOOTING.md](./docs/GITLAB_TOKEN_TROUBLESHOOTING.md) - Token issues
