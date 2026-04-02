# cherry-apply-oauth

A [Claude Code](https://claude.com/claude-code) skill that patches [Cherry Studio](https://github.com/CherryHQ/cherry-studio) to use your **Claude Max subscription** via OAuth — no API keys needed.

## What it does

This skill modifies 7 files in Cherry Studio to:

- **Read OAuth tokens from macOS Keychain** — where Claude Code CLI stores them after `claude auth login`
- **Enable chat** — adds Claude Code impersonation headers so the Anthropic API accepts OAuth tokens
- **Enable agents** — passes tokens via `CLAUDE_CODE_OAUTH_TOKEN` env var to trigger the SDK's native OAuth path
- **Simplify the UI** — replaces the API key form with a Keychain credential detection panel

After applying, both **chat** and **agents** work with your Claude Max subscription.

## Prerequisites

- macOS with [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code) installed
- Authenticated via `claude auth login` (credentials stored in macOS Keychain)
- A [Cherry Studio](https://github.com/CherryHQ/cherry-studio) checkout with `pnpm install` completed

## Usage

### Install the skill

```bash
cd /path/to/cherry-studio
mkdir -p .agents/skills/apply-oauth
curl -o .agents/skills/apply-oauth/SKILL.md \
  https://raw.githubusercontent.com/kennyzheng-builds/cherry-apply-oauth/main/SKILL.md
```

### Run the skill

In Claude Code, inside the Cherry Studio directory:

```
/apply-oauth
```

The skill will automatically:

1. **Apply code changes** — 2 full file rewrites + 5 targeted edits
2. **Verify** — `pnpm typecheck && pnpm lint && pnpm test`
3. **Commit & push** — to your private GitHub repo
4. **Build** — `pnpm build:mac` to produce a `.dmg` installer

### Update after upstream releases

When Cherry Studio releases a new version:

```bash
git fetch origin main
git rebase origin/main    # rebase your OAuth commits on top
# If conflicts are too many, just run /apply-oauth on the fresh code
```

## Files modified

| File | Change |
|---|---|
| `src/main/services/AnthropicService.ts` | Full rewrite — reads OAuth tokens from macOS Keychain |
| `src/renderer/src/pages/settings/ProviderSettings/AnthropicSettings.tsx` | Full rewrite — Keychain credential detection UI |
| `src/main/services/agents/BaseService.ts` | Skip API key validation for Anthropic |
| `src/main/services/agents/services/claudecode/index.ts` | Pass OAuth token via `CLAUDE_CODE_OAUTH_TOKEN` |
| `src/renderer/src/aiCore/provider/providerConfig.ts` | Add Claude Code impersonation headers for chat |
| `src/renderer/src/config/providers.ts` | Set `authType: 'oauth'` on Anthropic provider |
| `src/renderer/src/pages/settings/ProviderSettings/ProviderSetting.tsx` | Always show OAuth UI for Anthropic |

## How it works

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│  Claude Code CLI │────>│  macOS Keychain   │<────│  Cherry Studio  │
│  claude auth     │     │  OAuth tokens     │     │  (this patch)   │
│  login           │     │  access + refresh │     │                 │
└─────────────────┘     └──────────────────┘     └────────┬────────┘
                                                          │
                                              ┌───────────┴───────────┐
                                              │                       │
                                        ┌─────┴─────┐         ┌──────┴──────┐
                                        │   Chat     │         │   Agents    │
                                        │ AI SDK +   │         │ Claude Code │
                                        │ OAuth hdrs │         │ SDK + env   │
                                        └───────────┘         └─────────────┘
```

- **Chat path**: Reads token from Keychain → sends with Claude Code impersonation headers (`anthropic-beta: oauth-2025-04-20`, `user-agent: claude-cli/...`, `x-app: cli`)
- **Agent path**: Reads token from Keychain → passes via `CLAUDE_CODE_OAUTH_TOKEN` env var → SDK's native OAuth flow handles headers automatically

## License

MIT
