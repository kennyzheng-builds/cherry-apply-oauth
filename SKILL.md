---
name: apply-oauth
description: End-to-end workflow to add Claude OAuth support to Cherry Studio. Applies code changes, reviews, pushes to private GitHub repo, and builds macOS app. Run on a fresh upstream checkout or after git pull.
---

# Apply Claude OAuth to Cherry Studio

End-to-end skill: patch code -> review -> push to GitHub -> build macOS app.

Uses Claude Code's OAuth credentials from macOS Keychain so users with a Claude Max subscription can use Cherry Studio (chat + agents) without API keys.

## Full Workflow

Execute these phases in order. Stop and report if any phase fails.

### Phase 1: Apply Code Changes

Apply all 8 file changes described in the [Changes Reference](#changes-reference) section below.

- For **full rewrite** files: replace the entire file content
- For **targeted edit** files: find the matching code pattern and apply the edit. If the exact pattern doesn't match (upstream changed), use your understanding of the intent to adapt the edit to the current code structure.

### Phase 2: Verify

Run these commands and ensure they all pass:

```bash
pnpm typecheck
pnpm lint
pnpm test
```

- If `typecheck` fails: fix type errors (usually import order from eslint auto-fix)
- If `lint` has auto-fixable issues: run `pnpm format` then re-check
- If `test` fails: investigate whether the failure is related to our changes or pre-existing

### Phase 3: Commit & Push

```bash
# Stage only our changed files
git add \
  electron-builder.yml \
  src/main/services/AnthropicService.ts \
  src/main/services/agents/BaseService.ts \
  src/main/services/agents/services/claudecode/index.ts \
  src/renderer/src/aiCore/provider/providerConfig.ts \
  src/renderer/src/config/providers.ts \
  src/renderer/src/pages/settings/ProviderSettings/AnthropicSettings.tsx \
  src/renderer/src/pages/settings/ProviderSettings/ProviderSetting.tsx

# Commit
git commit --signoff -m "feat: add Claude OAuth support via macOS Keychain"

# Ensure personal remote exists
git remote get-url personal 2>/dev/null || \
  git remote add personal git@github.com:kennyzheng-builds/cherry-studio-personal.git

# Push current branch as main to personal repo
git push personal HEAD:main --force
```

### Phase 4: Build macOS App

```bash
pnpm build:mac
```

After build completes, report the output file paths (check `dist/` for `.dmg` files).
The DMG filenames should contain `-claude-oauth-` to distinguish from official builds.

### Phase 5: Prevent Spotlight Indexing of Build Output

macOS Spotlight will index `.app` bundles inside `dist/`, causing duplicate entries in app search.
Prevent this by placing a `.metadata_never_index` marker file in the `dist/` directory:

```bash
touch dist/.metadata_never_index
```

This tells macOS to skip indexing the `dist/` directory, so only the installed copy in `/Applications/` appears in Spotlight search results.

### Phase 6: Sync Built Packages to Personal Repo

After a successful build, create a GitHub release on the personal repo with the built DMG files:

```bash
# Get version from package.json
VERSION=$(node -p "require('./package.json').version")

# Create a git tag on the personal repo
git tag -f "v${VERSION}-claude-oauth" HEAD
git push personal "v${VERSION}-claude-oauth" --force

# Delete existing release if any, then create new one
gh release delete "v${VERSION}-claude-oauth" \
  --repo kennyzheng-builds/cherry-studio-personal \
  --yes 2>/dev/null || true

# Create GitHub release and upload DMG files
gh release create "v${VERSION}-claude-oauth" \
  --repo kennyzheng-builds/cherry-studio-personal \
  --title "Cherry Studio v${VERSION} (Claude OAuth)" \
  --notes "Personal build with Claude OAuth support via macOS Keychain" \
  --latest \
  dist/*-claude-oauth-*.dmg
```

---

## Changes Reference

### File 1: TARGETED EDIT — `electron-builder.yml`

**Intent:** Append `-claude-oauth` to the macOS DMG filename to distinguish from official builds.

Find the `mac:` section's `artifactName`:

**Before:**
```yaml
  artifactName: ${productName}-${version}-${arch}.${ext}
```

**After:**
```yaml
  artifactName: ${productName}-${version}-claude-oauth-${arch}.${ext}
```

> Only change the `mac:` section's artifactName. Do NOT modify Windows or Linux artifactNames.

### File 2: FULL REWRITE — `src/main/services/AnthropicService.ts`

Replace the entire file. This service reads OAuth tokens from macOS Keychain (where Claude Code CLI stores them), caches in memory, and auto-refreshes expired tokens.

```typescript
/**
 * AnthropicService - Reads Claude Code's OAuth credentials from macOS Keychain.
 *
 * Claude Code CLI stores OAuth tokens in macOS Keychain under:
 *   Service: "Claude Code-credentials"
 *   Account: <os username>
 *
 * This service reads those tokens directly, so users with a Claude Max subscription
 * can use Cherry Studio without a separate API key or OAuth flow.
 */
import { execFile } from 'node:child_process'
import { userInfo } from 'node:os'
import { promisify } from 'node:util'

import { loggerService } from '@logger'
import { net } from 'electron'

const execFileAsync = promisify(execFile)
const logger = loggerService.withContext('AnthropicOAuth')

const KEYCHAIN_SERVICE = 'Claude Code-credentials'
const KEYCHAIN_ACCOUNT = userInfo().username

// OAuth client ID used by Claude Code CLI
const CLIENT_ID = '9d1c250a-e61b-44d9-88ed-5944d1962f5e'

interface KeychainCredentials {
  claudeAiOauth?: {
    accessToken: string
    refreshToken: string
    expiresAt: number
    scopes?: string[]
    subscriptionType?: string
    rateLimitTier?: string
  }
}

interface Credentials {
  access_token: string
  refresh_token: string
  expires_at: number
}

class AnthropicService {
  private cachedCreds: Credentials | null = null

  /**
   * Read Claude Code's OAuth credentials from macOS Keychain
   */
  private async readKeychainCredentials(): Promise<Credentials | null> {
    try {
      const { stdout } = await execFileAsync('security', [
        'find-generic-password',
        '-s',
        KEYCHAIN_SERVICE,
        '-a',
        KEYCHAIN_ACCOUNT,
        '-w'
      ])

      const data: KeychainCredentials = JSON.parse(stdout.trim())
      if (!data.claudeAiOauth) {
        logger.warn('Keychain entry found but no claudeAiOauth data')
        return null
      }

      const oauth = data.claudeAiOauth
      logger.info('Read credentials from Keychain', {
        subscriptionType: oauth.subscriptionType,
        hasAccessToken: !!oauth.accessToken,
        expiresAt: new Date(oauth.expiresAt).toISOString()
      })

      return {
        access_token: oauth.accessToken,
        refresh_token: oauth.refreshToken,
        expires_at: oauth.expiresAt
      }
    } catch (error) {
      logger.debug('No Claude Code credentials found in Keychain', { error: (error as Error).message })
      return null
    }
  }

  /**
   * Refresh an expired access token using the refresh token
   */
  private async refreshAccessToken(refreshToken: string): Promise<Credentials> {
    const response = await net.fetch('https://console.anthropic.com/v1/oauth/token', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        grant_type: 'refresh_token',
        refresh_token: refreshToken,
        client_id: CLIENT_ID
      })
    })

    if (!response.ok) {
      const errorBody = await response.text()
      logger.error('Token refresh failed', { status: response.status, body: errorBody })
      throw new Error(`Token refresh failed (${response.status}): ${errorBody}`)
    }

    const data = await response.json()
    return {
      access_token: data.access_token,
      refresh_token: data.refresh_token,
      expires_at: Date.now() + data.expires_in * 1000
    }
  }

  /**
   * Write updated credentials back to macOS Keychain
   */
  private async writeKeychainCredentials(creds: Credentials): Promise<void> {
    try {
      // Read the full keychain entry first to preserve other fields
      const { stdout } = await execFileAsync('security', [
        'find-generic-password',
        '-s',
        KEYCHAIN_SERVICE,
        '-a',
        KEYCHAIN_ACCOUNT,
        '-w'
      ])

      const data: KeychainCredentials = JSON.parse(stdout.trim())
      if (data.claudeAiOauth) {
        data.claudeAiOauth.accessToken = creds.access_token
        data.claudeAiOauth.refreshToken = creds.refresh_token
        data.claudeAiOauth.expiresAt = creds.expires_at
      }

      const jsonStr = JSON.stringify(data)

      // Delete old entry and add new one
      await execFileAsync('security', [
        'delete-generic-password',
        '-s',
        KEYCHAIN_SERVICE,
        '-a',
        KEYCHAIN_ACCOUNT
      ]).catch(() => {})

      await execFileAsync('security', [
        'add-generic-password',
        '-s',
        KEYCHAIN_SERVICE,
        '-a',
        KEYCHAIN_ACCOUNT,
        '-w',
        jsonStr,
        '-U'
      ])

      logger.info('Updated credentials in Keychain')
    } catch (error) {
      logger.warn('Failed to write back to Keychain, credentials will expire', { error: (error as Error).message })
    }
  }

  /**
   * Get a valid access token - reads from Keychain, refreshes if expired
   */
  public async getValidAccessToken(): Promise<string | null> {
    // Try cache first
    if (this.cachedCreds && this.cachedCreds.expires_at > Date.now() + 60000) {
      return this.cachedCreds.access_token
    }

    // Read from Keychain
    const creds = await this.readKeychainCredentials()
    if (!creds) return null

    // Token still valid
    if (creds.expires_at > Date.now() + 60000) {
      this.cachedCreds = creds
      return creds.access_token
    }

    // Refresh expired token
    try {
      logger.info('Access token expired, refreshing...')
      const newCreds = await this.refreshAccessToken(creds.refresh_token)
      this.cachedCreds = newCreds
      await this.writeKeychainCredentials(newCreds)
      return newCreds.access_token
    } catch (error) {
      logger.error('Failed to refresh token', { error: (error as Error).message })
      return null
    }
  }

  /**
   * Check if Claude Code credentials exist in Keychain
   */
  public async hasCredentials(): Promise<boolean> {
    const creds = await this.readKeychainCredentials()
    return creds !== null
  }

  /**
   * Start OAuth flow - for keychain mode, just check if credentials exist
   * If they do, return the token. If not, show instructions.
   */
  public async startOAuthFlow(): Promise<string> {
    const token = await this.getValidAccessToken()
    if (token) return token
    throw new Error('No Claude Code credentials found. Please run "claude auth login" in your terminal first.')
  }

  /**
   * Complete OAuth flow - not needed for keychain mode
   */
  public async completeOAuthWithCode(_code: string): Promise<string> {
    const token = await this.getValidAccessToken()
    if (token) return token
    throw new Error('No Claude Code credentials found. Please run "claude auth login" in your terminal first.')
  }

  /**
   * Cancel OAuth flow - no-op for keychain mode
   */
  public async cancelOAuthFlow(): Promise<void> {
    // no-op
  }

  /**
   * Clear credentials - clears cache only, does not touch Keychain
   */
  public async clearCredentials(): Promise<void> {
    this.cachedCreds = null
    logger.info('Credentials cache cleared')
  }
}

export default new AnthropicService()
```

### File 3: FULL REWRITE — `src/renderer/src/pages/settings/ProviderSettings/AnthropicSettings.tsx`

Replace the entire file. Simplified from multi-step OAuth flow to Keychain credential detection.

```tsx
import { CheckCircleOutlined, ExclamationCircleOutlined } from '@ant-design/icons'
import { loggerService } from '@logger'
import { Alert, Button } from 'antd'
import { useEffect, useState } from 'react'
import { useTranslation } from 'react-i18next'
import styled from 'styled-components'

const logger = loggerService.withContext('AnthropicSettings')

const AnthropicSettings = () => {
  const { t } = useTranslation()
  const [hasCredentials, setHasCredentials] = useState<boolean | null>(null)
  const [loading, setLoading] = useState(false)

  const checkCredentials = async () => {
    try {
      setLoading(true)
      const result = await window.api.anthropic_oauth.hasCredentials()
      setHasCredentials(result)
    } catch (error) {
      logger.error('Failed to check credentials:', error as Error)
      setHasCredentials(false)
    } finally {
      setLoading(false)
    }
  }

  useEffect(() => {
    void checkCredentials()
  }, [])

  if (hasCredentials === null) {
    return null
  }

  if (hasCredentials) {
    return (
      <Container>
        <Alert
          type="success"
          message={t('settings.provider.anthropic.authenticated')}
          description="Using Claude Code credentials from macOS Keychain"
          showIcon
          icon={<CheckCircleOutlined />}
          action={
            <Button size="small" onClick={checkCredentials} loading={loading}>
              {t('common.refresh')}
            </Button>
          }
        />
      </Container>
    )
  }

  return (
    <Container>
      <Alert
        type="warning"
        message="Claude Code credentials not found"
        description='Please run "claude auth login" in your terminal first, then click Refresh.'
        showIcon
        icon={<ExclamationCircleOutlined />}
        action={
          <Button type="primary" size="small" onClick={checkCredentials} loading={loading}>
            {t('common.refresh')}
          </Button>
        }
      />
    </Container>
  )
}

const Container = styled.div`
  padding-top: 5px;
  margin-bottom: 10px;
`

export default AnthropicSettings
```

### File 4: TARGETED EDIT — `src/main/services/agents/BaseService.ts`

**Intent:** Skip API key validation for Anthropic provider (uses OAuth from Keychain).

Find the API key validation block (search for `const requiresApiKey`):

**Before** (upstream pattern):
```typescript
const requiresApiKey = !localProvidersWithoutApiKey.includes(validation.provider.id)

if (!validation.provider.apiKey) {
  if (requiresApiKey) {
    throw new AgentModelValidationError(
```

**After:**
```typescript
const requiresApiKey = !localProvidersWithoutApiKey.includes(validation.provider.id)
const isOAuthProvider = validation.provider.authType === 'oauth'

if (!validation.provider.apiKey) {
  if (isOAuthProvider || validation.provider.id === 'anthropic') {
    // OAuth providers (or Anthropic with keychain auth) don't need an API key
    validation.provider.apiKey = 'oauth-placeholder'
  } else if (requiresApiKey) {
    throw new AgentModelValidationError(
```

### File 5: TARGETED EDIT — `src/main/services/agents/services/claudecode/index.ts`

**Intent:** For Anthropic provider, read OAuth token from Keychain and pass via `CLAUDE_CODE_OAUTH_TOKEN` env var (triggers SDK's native OAuth headers). Use `~/.claude` config dir for OAuth.

#### 5a. Add import (near other `@main` imports):
```typescript
import anthropicService from '@main/services/AnthropicService'
```

#### 5b. Add OAuth token retrieval (after the `provider.apiKey = provider.id` fallback block, before `const apiConfig`):
```typescript
// For personal build: Anthropic always uses OAuth from Claude Code keychain
const isOAuthProvider = provider.id === 'anthropic'

// For OAuth: read token from keychain and pass via CLAUDE_CODE_OAUTH_TOKEN
// This triggers the SDK's native OAuth code path with proper headers.
let oauthToken: string | null = null
if (isOAuthProvider) {
  oauthToken = await anthropicService.getValidAccessToken()
  if (!oauthToken) {
    aiStream.emit('data', {
      type: 'error',
      error: new Error(
        'No Claude Code credentials found in Keychain. Please run "claude auth login" in your terminal first.'
      )
    })
    return aiStream
  }
  logger.info('Using OAuth token from Keychain for Claude Agent SDK')
}
```

#### 5c. Replace CLAUDE_CONFIG_DIR (find `CLAUDE_CONFIG_DIR: path.join(app.getPath('userData'), '.claude')`):

**Before:**
```typescript
      // Set CLAUDE_CONFIG_DIR to app's userData directory to avoid path encoding issues
      // on Windows when the username contains non-ASCII characters (e.g., Chinese characters)
      // This prevents the SDK from using the user's home directory which may have encoding problems
      CLAUDE_CONFIG_DIR: path.join(app.getPath('userData'), '.claude'),
```

**After:**
```typescript
      // For OAuth: use ~/.claude so the SDK subprocess can find .claude.json for account metadata
      // For non-OAuth: use app's userData directory to avoid path encoding issues on Windows
      CLAUDE_CONFIG_DIR: isOAuthProvider
        ? path.join(process.env.HOME || os.homedir(), '.claude')
        : path.join(app.getPath('userData'), '.claude'),
```

> Note: `os` is already imported in this file as `import os from 'node:os'`.

#### 5d. Replace API key env vars (find `ANTHROPIC_API_KEY:` and `ANTHROPIC_AUTH_TOKEN:` lines):

**Before:**
```typescript
      ANTHROPIC_API_KEY: provider.apiKey,
      ANTHROPIC_AUTH_TOKEN: provider.apiKey,
```

**After:**
```typescript
      // For OAuth: use CLAUDE_CODE_OAUTH_TOKEN which triggers the SDK's native OAuth
      // code path with proper headers (anthropic-beta: oauth-2025-04-20, etc.)
      // ANTHROPIC_AUTH_TOKEN does NOT trigger OAuth headers and gets rejected.
      ...(isOAuthProvider
        ? {
            CLAUDE_CODE_OAUTH_TOKEN: oauthToken!
          }
        : {
            ANTHROPIC_API_KEY: provider.apiKey,
            ANTHROPIC_AUTH_TOKEN: provider.apiKey
          }),
```

### File 6: TARGETED EDIT — `src/renderer/src/aiCore/provider/providerConfig.ts`

**Intent:** Add Claude Code impersonation headers so Anthropic API accepts OAuth tokens for chat.

Find the `buildAnthropicConfig` function's `headers` object. Add these headers before the `Authorization` line:

```typescript
'anthropic-beta':
  'oauth-2025-04-20,claude-code-20250219,interleaved-thinking-2025-05-14,fine-grained-tool-streaming-2025-05-14',
'anthropic-dangerous-direct-browser-access': 'true',
'user-agent': 'claude-cli/1.0.118 (external, sdk-ts)',
'x-app': 'cli',
```

> **If upstream removes `buildAnthropicConfig`:** find wherever the Anthropic provider config is built and ensure these headers are included when using OAuth auth.

### File 7: TARGETED EDIT — `src/renderer/src/config/providers.ts`

**Intent:** Mark Anthropic system provider as OAuth-based.

Find the Anthropic entry in `SYSTEM_PROVIDERS_CONFIG` (search for `id: 'anthropic'`). Add `authType: 'oauth'` to the config object:

```typescript
authType: 'oauth'
```

---

### File 8: TARGETED EDIT — `src/renderer/src/pages/settings/ProviderSettings/ProviderSetting.tsx`

**Intent:** For personal build, Anthropic always uses OAuth. Remove the auth method dropdown and always show `<AnthropicSettings />`.

#### 8a. Find `isAnthropicOAuth` function (search for `const isAnthropicOAuth`):

**Before:**
```typescript
const isAnthropicOAuth = () => provider.id === 'anthropic' && provider.authType === 'oauth'
```

**After:**
```typescript
// Personal build: Anthropic always uses OAuth from Claude Code keychain
const isAnthropicOAuth = () => provider.id === 'anthropic'
```

#### 8b. Find the Anthropic auth method selector block (search for `provider.id === 'anthropic'` in the JSX return):

**Before:**
```tsx
{provider.id === 'anthropic' && (
  <>
    <SettingSubtitle style={{ marginTop: 5 }}>{t('settings.provider.anthropic.auth_method')}</SettingSubtitle>
    <Select
      style={{ width: '40%', marginTop: 5, marginBottom: 10 }}
      value={provider.authType || 'apiKey'}
      onChange={(value) => updateProvider({ authType: value })}
      options={[
        { value: 'apiKey', label: t('settings.provider.anthropic.apikey') },
        { value: 'oauth', label: t('settings.provider.anthropic.oauth') }
      ]}
    />
    {provider.authType === 'oauth' && <AnthropicSettings />}
  </>
)}
```

**After:**
```tsx
{provider.id === 'anthropic' && <AnthropicSettings />}
```

> This removes the auth method dropdown entirely and always renders the OAuth Keychain detection UI for Anthropic.

#### 8c. Clean up unused import

After removing the `<Select>` usage for Anthropic, check if `Select` is still used elsewhere in the file. If not, remove it from the antd import:

**Before:**
```typescript
import { Button, Divider, Flex, Input, Select, Space, Switch, Tooltip } from 'antd'
```

**After:**
```typescript
import { Button, Divider, Flex, Input, Space, Switch, Tooltip } from 'antd'
```

> Only remove `Select` if no other usage exists in the file. Search the file first.

---

## Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| 401 "OAuth authentication is currently not supported" | Missing Claude Code impersonation headers in chat path | Check File 6 headers |
| "Not logged in - Please run /login" | Agent SDK can't find OAuth token | Check File 5d uses `CLAUDE_CODE_OAUTH_TOKEN` not `ANTHROPIC_AUTH_TOKEN` |
| "Provider 'anthropic' is missing an API key" | BaseService blocks empty apiKey | Check File 4 OAuth bypass |
| Token expired / null | Keychain credentials missing or expired | Run `claude auth login` in terminal |
| Multiple Cherry Studio in Spotlight | Build output `.app` indexed by macOS | Check Phase 5 `.metadata_never_index` |
| DMG filename same as official build | Missing `-claude-oauth` in artifactName | Check File 1 electron-builder.yml |

## Key Technical Decisions

- **`CLAUDE_CODE_OAUTH_TOKEN`** not `ANTHROPIC_AUTH_TOKEN`: Only `CLAUDE_CODE_OAUTH_TOKEN` triggers the Claude Agent SDK's native OAuth code path which sends `anthropic-beta: oauth-2025-04-20` and other required headers.
- **macOS Keychain**: Claude Code CLI stores credentials there. We read directly, avoiding duplication.
- **`CLAUDE_CONFIG_DIR: ~/.claude`** for OAuth: The SDK subprocess needs `.claude.json` for account metadata.
- **`-claude-oauth` in DMG filename**: Distinguishes personal OAuth builds from official Cherry Studio releases.
- **`.metadata_never_index`**: Prevents macOS Spotlight from indexing `.app` bundles in `dist/`, avoiding duplicate entries in app search.
- **Personal repo**: `git@github.com:kennyzheng-builds/cherry-studio-personal.git` (remote name: `personal`). Code is force-pushed to `main`, and built DMGs are uploaded as GitHub releases.
