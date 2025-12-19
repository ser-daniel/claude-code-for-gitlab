# Claude Code Integration Setup for git.agenticlab.tech

Complete setup guide for integrating Claude Code with your self-hosted GitLab instance.

## Overview

Your setup includes:
- **GitLab CE** at `https://git.agenticlab.tech` (Docker-based)
- **Dedicated Claude CI Runner** with xray proxy routing (`172.17.0.1:801`)
- **Webhook Service** for `@claude` mention handling
- **Nginx reverse proxy** with Let's Encrypt SSL

---

## 1. CI Runner Setup ✅ COMPLETED

A dedicated GitLab runner has been configured with:
- Tag: `claude-code` (use this in your CI jobs)
- Network: Connected to GitLab's internal network
- Runner ID: 6
- **Note**: Proxy is configured per-job, not globally, to avoid compatibility issues

The runner is active and visible in GitLab Admin → Runners.

---

## 2. GitLab Configuration ✅ COMPLETED

Added to `/srv/gitlab-ce/data/config/gitlab.rb`:
- OAuth support enabled
- API rate limits increased (3000 req/min)
- Webhook timeout extended (60s)

Changes applied via `gitlab-ctl reconfigure`.

---

## 3. Webhook Service Setup 🔧 IN PROGRESS

### Step 1: Configure Environment

Edit `/srv/claude-code-for-gitlab/gitlab-app/.env`:

```bash
cd /srv/claude-code-for-gitlab/gitlab-app
nano .env
```

**Required changes:**

```env
# Replace with your GitLab personal access token (needs 'api' scope)
GITLAB_TOKEN=glpat-YOUR_TOKEN_HERE

# Set a secure webhook secret (generated):
WEBHOOK_SECRET=aa9c4084ebaa3f7e41f9b6e6192fdb9a920d208e2911c50728cc2446da6dc66c
```

**To create a GitLab token:**
1. Go to https://git.agenticlab.tech/-/user_settings/personal_access_tokens
2. Create token with scopes: `api`, `read_repository`, `write_repository`
3. Copy the token and paste into `.env`

### Step 2: Build and Start Webhook Service

```bash
cd /srv/claude-code-for-gitlab/gitlab-app
docker compose -f docker-compose.agenticlab.yml build
docker compose -f docker-compose.agenticlab.yml up -d
```

Verify it's running:
```bash
docker logs claude-webhook
curl http://localhost:3001/health  # Should return OK
```

---

## 4. Nginx Configuration 🔧 IN PROGRESS

Choose one option:

### Option A: Add to Existing Domain (Simpler)

Add webhook endpoint to `git.agenticlab.tech`:

```bash
sudo nano /etc/nginx/sites-enabled/git.agenticlab.tech
```

Add this **inside** the `server { listen 443 ssl; }` block:

```nginx
    location /claude-webhook {
        proxy_pass http://localhost:3001;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-GitLab-Event $http_x_gitlab_event;
        proxy_set_header X-GitLab-Token $http_x_gitlab_token;
        proxy_read_timeout 120s;
        proxy_connect_timeout 120s;
    }
```

Test and reload:
```bash
sudo nginx -t
sudo systemctl reload nginx
```

**Webhook URL will be:** `https://git.agenticlab.tech/claude-webhook`

### Option B: Separate Subdomain (Recommended)

Create new subdomain `claude-webhook.agenticlab.tech`:

```bash
sudo nano /etc/nginx/sites-available/claude-webhook.agenticlab.tech
```

Paste the configuration from `/tmp/claude-webhook-nginx.conf` (Option 2 section).

Enable and get SSL certificate:
```bash
sudo ln -s /etc/nginx/sites-available/claude-webhook.agenticlab.tech \
            /etc/nginx/sites-enabled/

# Get SSL certificate
sudo certbot --nginx -d claude-webhook.agenticlab.tech

sudo systemctl reload nginx
```

**Webhook URL will be:** `https://claude-webhook.agenticlab.tech/webhook`

---

## 5. GitLab Webhook Configuration 📋 NEXT STEPS

For each project or group where you want Claude Code:

### Group-Level Webhook (Recommended)

1. Go to your group: https://git.agenticlab.tech/groups/YOUR_GROUP
2. Navigate to **Settings → Webhooks**
3. Click **Add new webhook**
4. Configure:
   - **URL:** `https://git.agenticlab.tech/claude-webhook` (or your chosen URL)
   - **Secret token:** `aa9c4084ebaa3f7e41f9b6e6192fdb9a920d208e2911c50728cc2446da6dc66c`
   - **Trigger:** ✅ Comments
   - **SSL verification:** ✅ Enable SSL verification
5. Click **Add webhook**

### Project-Level Webhook

Same process but in **Project → Settings → Webhooks**

---

## 6. Project CI Configuration 📄 NEXT STEPS

Add to your project's `.gitlab-ci.yml` (or copy from `/srv/claude-code-for-gitlab/examples/gitlab/agenticlab-setup.gitlab-ci.yml`):

```yaml
variables:
  CLAUDE_TRIGGER_PHRASE: "@claude"
  CLAUDE_MODEL: "opus"
  CLAUDE_BRANCH_PREFIX: "claude/"
  BASE_BRANCH: "main"
  GIT_STRATEGY: clone
  GIT_DEPTH: 0

claude_assistant:
  image: oven/bun:1.1.29-alpine
  stage: build
  tags:
    - claude-code  # Important: Use the dedicated runner

  before_script:
    - apk add --no-cache git openssh-client ca-certificates curl nodejs npm
    - git config --global user.name "Claude Assistant"
    - git config --global user.email "claude@agenticlab.tech"
    - git clone https://github.com/ser-daniel/claude-code-for-gitlab.git /tmp/claude-code
    - cd /tmp/claude-code && bun install --frozen-lockfile
    - cd $CI_PROJECT_DIR

  script:
    # Step 1: Run prepare script (WITHOUT proxy to avoid undici issues)
    - echo "Step 1 - Preparing Claude Code action..."
    - cd /tmp/claude-code && bun run src/entrypoints/prepare.ts

    # Step 2: Install Claude Code globally
    - echo "Step 2 - Installing Claude Code..."
    - bun install -g @anthropic-ai/claude-code@1.0.60

    # Step 3: Install base-action dependencies
    - echo "Step 3 - Installing base-action dependencies..."
    - cd /tmp/claude-code/base-action && bun install && cd /tmp/claude-code

    # Step 4: Run Claude Code via base-action (WITH proxy for Anthropic API)
    - echo "Step 4 - Running Claude Code with proxy..."
    - |
      export HTTP_PROXY=http://172.17.0.1:801
      export HTTPS_PROXY=http://172.17.0.1:801
      export http_proxy=http://172.17.0.1:801
      export https_proxy=http://172.17.0.1:801
      export NO_PROXY=gitlab,localhost,127.0.0.1,172.17.0.0/16,git.agenticlab.tech
      export no_proxy=gitlab,localhost,127.0.0.1,172.17.0.0/16,git.agenticlab.tech
      export CLAUDE_CODE_ACTION="1"
      export INPUT_PROMPT_FILE="/tmp/claude-prompts/claude-prompt.txt"
      export ANTHROPIC_MODEL="${CLAUDE_MODEL:-opus}"
      cd /tmp/claude-code && bun run base-action/src/index.ts

    # Step 5: Update comment with results
    - cd /tmp/claude-code && bun run src/entrypoints/update-comment-gitlab.ts || true

  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      when: always
    - if: '$CLAUDE_TRIGGER == "true"'  # Triggered by webhook
      when: always

  variables:
    CLAUDE_CODE_OAUTH_TOKEN: $CLAUDE_CODE_OAUTH_TOKEN
    CI_SERVER_URL: "https://git.agenticlab.tech"
```

### Add CI/CD Variables

In your project or group settings (**Settings → CI/CD → Variables**):

| Variable | Value | Protected | Masked | Description |
|----------|-------|-----------|---------|-------------|
| `CLAUDE_CODE_GL_ACCESS_TOKEN` | `glpat-Ykn9BA...` | ❌ **NO** | ✅ | GitLab Personal Access Token (for GitLab API) |
| `CLAUDE_CODE_OAUTH_TOKEN` | Your Claude token | ❌ **NO** | ✅ | Claude Code OAuth token (for Anthropic API) |

**How to get these values:**

1. **CLAUDE_CODE_GL_ACCESS_TOKEN**: GitLab Personal Access Token
   - Go to https://git.agenticlab.tech/-/user_settings/personal_access_tokens
   - Create token with scopes: `api`, `read_repository`, `write_repository`
   - **You already have one:** `glpat-Ykn9BATOykcTLu6IDLpJ1m86MQp1OjMH.01.0w1pbijb5` (from `.env`)

2. **CLAUDE_CODE_OAUTH_TOKEN**: Claude Code OAuth token
   - Get from [Claude Code settings](https://claude.ai/settings)
   - Or use `ANTHROPIC_API_KEY` as alternative

**Note:** The GitLab OAuth app credentials (`GITLAB_APP_ID` and `GITLAB_APP_SECRET`) are **NOT** used in CI/CD pipelines. They're only used by the webhook service for OAuth authentication flows.

**IMPORTANT**: Do **NOT** mark these variables as "Protected"! Protected variables are only available on protected branches, which will prevent Claude Code from working on merge request pipelines from feature branches.

**Security Note**: While not marked as protected, these variables should be:
- ✅ **Masked**: Yes for secrets (hides values in job logs)
- ✅ **Expand variable reference**: Yes (allows variable expansion)
- ✅ **Environment scope**: All (default)

---

## 7. Testing the Integration 🧪

### Test 1: Webhook Service Health

```bash
curl https://git.agenticlab.tech/claude-webhook/health
# Should return: {"status":"ok"}
```

### Test 2: Create a Test MR

1. Create a new branch in any project
2. Make a small change and push
3. Create a merge request
4. Comment: `@claude please review this change`
5. Check:
   - Webhook delivery in **Project → Settings → Webhooks → Edit → Recent events**
   - Pipeline triggered in **CI/CD → Pipelines**
   - Runner picked up job (look for `claude-code` tag)

### Test 3: Check Logs

```bash
# Webhook service logs
docker logs claude-webhook -f

# GitLab runner logs
docker logs gitlab-runner-claude -f

# Check proxy is working (in CI job output)
# Should see HTTP_PROXY=http://172.17.0.1:801
```

---

## Troubleshooting

### Webhook not triggering

1. Check webhook recent events in GitLab
2. Verify webhook secret matches
3. Check webhook service logs: `docker logs claude-webhook`
4. Test webhook endpoint: `curl -I https://git.agenticlab.tech/claude-webhook/health`

### CI job proxy issues

**Note**: Proxy configuration has been updated to avoid compatibility issues.

The proxy is now configured **per-job** in the CI script, not globally in the runner config. This is because:
- Global proxy variables cause issues with the `@actions/http-client` library during the prepare phase
- The proxy is only needed when actually calling the Anthropic API, not for all GitLab API calls
- Bun runtime has compatibility issues with `undici.ProxyAgent` when proxy is set globally

Current configuration:
1. Runner config: No global proxy environment variables
2. CI job: Proxy set in `script:` section before running `gitlab_entrypoint.ts`
3. Proxy applies to: Anthropic API calls only

If proxy isn't working:
1. Check that the script section exports proxy variables correctly
2. Verify xray proxy is running: `systemctl status xray`
3. Test proxy from CI job: `curl --proxy http://172.17.0.1:801 https://api.anthropic.com/v1/messages`

### Rate limiting issues

Webhook service implements rate limiting:
- 3 triggers per user per resource per 15 minutes
- Check logs for "Rate limit exceeded" messages
- Adjust in `.env`: `RATE_LIMIT_MAX` and `RATE_LIMIT_WINDOW`

### Proxy connection issues

Test proxy from runner container:
```bash
docker run --rm --network gitlab-ce_gitlab --add-host host.docker.internal:172.17.0.1 \
  -e HTTPS_PROXY=http://172.17.0.1:801 \
  alpine sh -c "apk add curl && curl -I https://api.anthropic.com"
```

---

## Architecture Diagram

```
GitLab Comment (@claude)
    ↓
Webhook → https://git.agenticlab.tech/claude-webhook
    ↓
Webhook Service (localhost:3000)
    ↓ (triggers pipeline)
GitLab CI/CD
    ↓
Claude Runner (with tag: claude-code)
    ↓ (all traffic via proxy)
Xray Proxy (172.17.0.1:801)
    ↓
Claude API (api.anthropic.com)
```

---

## File Locations

- **GitLab data:** `/srv/gitlab-ce/data/`
- **GitLab config:** `/srv/gitlab-ce/data/config/gitlab.rb`
- **Main runner config:** `/srv/gitlab-ce/gitlab-runner/config/config.toml`
- **Claude runner config:** `/srv/gitlab-ce/gitlab-runner-claude/config/config.toml`
- **Webhook service:** `/srv/claude-code-for-gitlab/gitlab-app/`
- **Webhook .env:** `/srv/claude-code-for-gitlab/gitlab-app/.env`
- **Example CI:** `/srv/claude-code-for-gitlab/examples/gitlab/agenticlab-setup.gitlab-ci.yml`

---

## Next Actions Required

1. [ ] Add GitLab personal access token to `/srv/claude-code-for-gitlab/gitlab-app/.env`
2. [ ] Build and start webhook service
3. [ ] Configure nginx (choose Option A or B)
4. [ ] Set up group or project webhooks in GitLab
5. [ ] Add CI/CD variables (CLAUDE_CODE_OAUTH_TOKEN or ANTHROPIC_API_KEY)
6. [ ] Test with a sample merge request

---

## Security Notes

- Webhook secret is 64 characters (secure)
- Webhook service only exposed via reverse proxy with SSL
- GitLab token has minimal required scopes
- Rate limiting prevents abuse
- Proxy routing keeps Claude API key on trusted network
- Runner isolated with dedicated tag

---

## Support

For issues:
- **GitLab runner:** Check logs with `docker logs gitlab-runner-claude`
- **Webhook service:** Check logs with `docker logs claude-webhook`
- **Redis:** Check logs with `docker logs claude-webhook-redis`
- **Nginx:** Check logs with `sudo tail -f /var/log/nginx/error.log`
