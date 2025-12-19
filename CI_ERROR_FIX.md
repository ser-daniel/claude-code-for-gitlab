# CI Error Fix Summary

## Problem

The GitLab CI pipeline was failing with the following error:

```
TypeError: undefined is not a constructor (evaluating 'new undici_1.ProxyAgent...')
  at _getProxyAgentDispatcher (/tmp/claude-code/node_modules/@actions/http-client/lib/index.js:563:22)
```

## Root Cause

1. **Global proxy configuration**: The GitLab runner was configured with global `HTTP_PROXY` environment variables
2. **Library incompatibility**: The `@actions/http-client` library (used by `@actions/github`) automatically detects proxy environment variables and tries to use `undici.ProxyAgent`
3. **Bun runtime issue**: The Bun runtime doesn't fully support `undici.ProxyAgent`, causing it to be undefined
4. **Unnecessary proxy usage**: The proxy was being applied to all steps, including the prepare phase which only makes GitLab API calls (not Anthropic API calls)

## Solution

### 1. Updated Runner Configuration

**File**: `/srv/gitlab-ce/gitlab-runner-claude/config/config.toml`

**Change**: Removed global proxy environment variables from the runner configuration

**Before**:
```toml
environment = ["HTTP_PROXY=http://172.17.0.1:801", "HTTPS_PROXY=http://172.17.0.1:801", ...]
```

**After**:
```toml
# No global environment variables
```

### 2. Updated CI Configuration

**File**: `/srv/claude-code-for-gitlab/examples/gitlab/agenticlab-setup.gitlab-ci.yml`

**Changes**:
1. Fixed repository URL: `https://github.com/RealMikeChong/claude-code-for-gitlab.git` → `https://github.com/ser-daniel/claude-code-for-gitlab.git`
2. Changed entrypoint: `prepare.ts` → `gitlab_entrypoint.ts` (unified entrypoint that handles all phases)
3. Added proxy configuration in the script section (only applies during Claude Code execution)
4. Added Node.js installation (required for some dependencies)

**Key configuration**:
```yaml
script:
  # Step 1: Run prepare script (WITHOUT proxy to avoid undici issues)
  - echo "Step 1 - Preparing Claude Code action..."
  - cd /tmp/claude-code && bun run src/entrypoints/prepare.ts

  # Step 2: Install Claude Code globally
  - bun install -g @anthropic-ai/claude-code@1.0.60

  # Step 3: Install base-action dependencies
  - cd /tmp/claude-code/base-action && bun install && cd /tmp/claude-code

  # Step 4: Run Claude Code via base-action (WITH proxy for Anthropic API)
  - |
    export HTTP_PROXY=http://172.17.0.1:801
    export HTTPS_PROXY=http://172.17.0.1:801
    # ... other proxy vars ...
    cd /tmp/claude-code && bun run base-action/src/index.ts

  # Step 5: Update comment with results
  - cd /tmp/claude-code && bun run src/entrypoints/update-comment-gitlab.ts || true
```

### 3. Restarted Runner

```bash
cd /srv/gitlab-ce && docker compose restart gitlab-runner-claude
```

## Benefits of This Approach

1. **No compatibility issues**: Proxy is only set for Step 4 (base-action), avoiding Bun/undici incompatibilities in earlier steps
2. **Cleaner separation**: GitLab API calls (Steps 1-3) don't go through proxy, only Anthropic API calls (Step 4) do
3. **Multi-step workflow**: Clear separation between prepare, install, execute, and update phases
4. **Easier debugging**: Each step is independent and can be debugged separately

## Verification

### 1. Check Proxy Service
```bash
systemctl status xray
ss -tlnp | grep :801
```

Both should show the proxy is running and listening on port 801.

### 2. Check Runner
```bash
docker logs gitlab-runner-claude -f
```

Runner should start jobs without proxy environment variables in the initial environment.

### 3. Test CI Pipeline

Create a test merge request and comment `@claude` to trigger the pipeline. The job should:
1. Clone the repository successfully
2. Install dependencies without proxy errors
3. Run prepare phase without proxy issues
4. Apply proxy settings before running Claude Code
5. Successfully call Anthropic API through the proxy

## What's Different Now

| Aspect | Before | After |
|--------|--------|-------|
| **Proxy config** | Global (runner level) | Per-job (script level) |
| **Workflow** | Single `prepare.ts` call | Multi-step: prepare → install → execute → update |
| **Repository** | `RealMikeChong/claude-code-for-gitlab` | `ser-daniel/claude-code-for-gitlab` |
| **Phases** | Only prepare (incomplete) | All phases (complete) |
| **Dependencies** | Basic Alpine packages | Added Node.js for compatibility |

## Next Steps

1. **Configure webhook**: Set up GitLab webhook to trigger pipelines on `@claude` mentions
2. **Add GitLab token**: Update `/srv/claude-code-for-gitlab/gitlab-app/.env` with your GitLab personal access token
3. **Add Claude credentials**: Add `CLAUDE_CODE_OAUTH_TOKEN` or `ANTHROPIC_API_KEY` as CI/CD variables
4. **Test end-to-end**: Create a test MR and mention `@claude` to verify the complete workflow

## Files Modified

1. `/srv/gitlab-ce/gitlab-runner-claude/config/config.toml` - Removed global proxy config
2. `/srv/claude-code-for-gitlab/examples/gitlab/agenticlab-setup.gitlab-ci.yml` - Updated CI config
3. `/srv/claude-code-for-gitlab/AGENTICLAB_SETUP.md` - Updated documentation

## References

- Unified entrypoint implementation: `/srv/claude-code-for-gitlab/src/entrypoints/gitlab_entrypoint.ts`
- Webhook-triggered example: `/srv/claude-code-for-gitlab/examples/gitlab/webhook-triggered.gitlab-ci.yml`
- Xray proxy config: `/usr/local/etc/xray/config.json`
