# Troubleshooting: CI/CD Variables Not Expanding

## Problem

CI pipeline fails with error:
```
CLAUDE_CODE_OAUTH_TOKEN: UNEXPANDED ("$CLAUDE_CODE_OAUTH_TOKEN") - Variable not set in CI/CD settings!
GitbeakerRequestError: Unauthorized
```

The variable appears as a literal string `"$CLAUDE_CODE_OAUTH_TOKEN"` instead of being expanded to the actual token value.

## Root Cause

The most common cause is having the CI/CD variable marked as **"Protected"** in GitLab settings.

Protected variables are **ONLY** available on:
- Protected branches (like `main`, `master`)
- Pipelines running directly on those branches

Protected variables are **NOT** available on:
- Feature branches
- Merge request pipelines from non-protected branches
- Fork pipelines

Since Claude Code is typically triggered on merge request comments, and MRs are usually from feature branches, marking the variable as "Protected" will cause it to not be available.

## Solution

### Step 1: Update Variable Settings in GitLab

1. Navigate to your project or group:
   - **Project**: `https://git.agenticlab.tech/YOUR_GROUP/YOUR_PROJECT/-/settings/ci_cd`
   - **Group**: `https://git.agenticlab.tech/groups/YOUR_GROUP/-/settings/ci_cd`

2. Expand **"Variables"** section

3. Find `CLAUDE_CODE_OAUTH_TOKEN` and click **Edit**

4. Configure these settings:
   - ❌ **Protected**: **UNCHECK THIS** (this is the critical fix!)
   - ✅ **Masked**: Keep checked (for security)
   - ✅ **Expand variable reference**: Keep checked
   - **Environment scope**: `*` (All - default)

5. Click **Update variable**

### Step 2: Verify in CI Pipeline

After updating the variable, trigger a new pipeline on your merge request:

1. Add a new comment on the MR: `@claude hello`
2. Check the pipeline logs
3. Look for the debug output:
   ```
   === GitLab Environment Variables Debug ===
   CLAUDE_CODE_OAUTH_TOKEN: Set (length: XX, prefix: "clot-...")
   ```

If it now shows "Set" instead of "UNEXPANDED", the fix worked!

## Why Not Use Protected Variables?

While protected variables seem more secure, they're not suitable for Claude Code integration because:

1. **MR pipelines need access**: Claude is typically invoked on MRs from feature branches
2. **Masked provides security**: The "Masked" flag prevents the token from appearing in logs
3. **Access control is at project/group level**: Only users with appropriate permissions can modify variables

## Alternative Solutions

If you absolutely must use protected variables (e.g., company security policy), you have two options:

### Option A: Protect Your Feature Branches

Mark your feature branches as protected:
1. Go to **Settings → Repository → Protected branches**
2. Add a wildcard pattern like `claude/*`
3. Configure appropriate push/merge permissions

This allows protected variables to be available on those branches.

### Option B: Use Protected Branches for Claude Work

1. Only use Claude on commits to `main` branch
2. Set up rules to only trigger Claude on protected branches
3. Update the CI configuration:
   ```yaml
   rules:
     - if: '$CI_COMMIT_BRANCH == "main"'
       when: always
   ```

**Note**: Option A or B significantly limits Claude Code's usefulness for collaborative development.

## Other Possible Causes

If unchecking "Protected" doesn't fix the issue, check these:

1. **Variable name typo**: Ensure it's exactly `CLAUDE_CODE_OAUTH_TOKEN` (case-sensitive)
2. **Variable scope**: If set at group level, ensure group-level variables are enabled for the project
3. **Expand variable reference**: Must be checked (should be by default)
4. **Variable value**: Ensure the actual token value is set, not a placeholder

## Verification Commands

Add this to your CI script to debug variable issues:

```yaml
script:
  # Debug: Check if variable is set and expanded
  - |
    if [ -z "$CLAUDE_CODE_OAUTH_TOKEN" ]; then
      echo "ERROR: CLAUDE_CODE_OAUTH_TOKEN is not set"
    elif [ "$CLAUDE_CODE_OAUTH_TOKEN" = '$CLAUDE_CODE_OAUTH_TOKEN' ]; then
      echo "ERROR: CLAUDE_CODE_OAUTH_TOKEN is not expanded (Protected variable on non-protected branch?)"
    else
      echo "SUCCESS: CLAUDE_CODE_OAUTH_TOKEN is set (length: ${#CLAUDE_CODE_OAUTH_TOKEN})"
    fi
```

## Related Documentation

- [GitLab Protected Variables](https://docs.gitlab.com/ee/ci/variables/#protect-a-cicd-variable)
- [GitLab Masked Variables](https://docs.gitlab.com/ee/ci/variables/#mask-a-cicd-variable)
- [AGENTICLAB_SETUP.md](./AGENTICLAB_SETUP.md) - Complete setup guide
- [CI_ERROR_FIX.md](./CI_ERROR_FIX.md) - Proxy configuration troubleshooting
