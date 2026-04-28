# APM {VERSION} - MCP Registry

This registry lists MCPs known to the APM framework with canonical installation commands and authentication flows. When the Planner captures an MCP requirement in the Spec or the Manager presents setup guidance to the User, they resolve entries here.

Entries include installation commands for Claude Code's `claude mcp add` syntax. Other platforms (Cursor, Codex, Gemini CLI, etc.) have their own MCP configuration mechanisms - the Manager translates the intent into the active platform's syntax when presenting to the User.

---

## Playwright

**Purpose:** Browser automation for web E2E validation. Navigate pages, interact with DOM, capture screenshots and Playwright traces.

**Install:**

```bash
claude mcp add playwright --transport stdio -- npx @playwright/mcp@latest
```

**Authentication:** None.

**Troubleshooting:**

- If `claude mcp list` shows playwright as installed but invocations fail, restart Claude Code.
- First invocation downloads Chromium and other browsers - expect a 1-2 minute delay on the first run.
- On macOS, Gatekeeper may prompt for permission the first time a browser launches; approve it.

---

## Supabase

**Purpose:** Database migrations, schema inspection, seeding test data for E2E scenarios, Row Level Security rule verification.

**Install:**

```bash
claude mcp add supabase --transport stdio -- npx @supabase/mcp-server-supabase@latest
```

**Authentication:** Service role key.

**User flow:**

1. Open Supabase dashboard: `https://supabase.com/dashboard`
2. Select the project.
3. Navigate to Project Settings → API.
4. Copy the `service_role` key (NOT the `anon` key - the service role has elevated privileges for schema and data operations).
5. Set it as `SUPABASE_SERVICE_ROLE_KEY` environment variable in the shell where Claude Code runs, or pass it via MCP server args depending on the server version. Check `claude mcp list` output for exact configuration requirements.

**Security note:** The service role key bypasses Row Level Security policies. Never commit it to version control. Use local environment variables or a secret manager; `.env.local` should be in `.gitignore`.

---

## Vercel

**Purpose:** Deployment status inspection, environment variable management, preview URL retrieval, build log access.

**Install:**

```bash
claude mcp add vercel --transport stdio -- npx @vercel/mcp@latest
```

**Authentication:** Vercel API token.

**User flow:**

1. Open Vercel dashboard: `https://vercel.com/account/tokens`
2. Create a new token scoped to the target project (or full-account scope if multiple projects).
3. Export as `VERCEL_TOKEN` in the shell where Claude Code runs, or pass via MCP config.

---

## Chrome DevTools

**Purpose:** Real Chrome DevTools protocol access - network inspection, console logs, performance profiling, coverage.

**Install:**

```bash
claude mcp add chrome-devtools --transport stdio -- npx @modelcontextprotocol/server-chrome-devtools@latest
```

**Authentication:** None. Requires Chrome launched with remote debugging enabled.

**User flow:**

1. Close all Chrome instances (important - remote debugging conflicts with existing sessions).
2. Launch Chrome with `--remote-debugging-port=9222`:
   - macOS: `open -a "Google Chrome" --args --remote-debugging-port=9222`
   - Linux: `google-chrome --remote-debugging-port=9222 &`
   - Windows: `"C:\Program Files\Google\Chrome\Application\chrome.exe" --remote-debugging-port=9222`
3. The MCP connects to port 9222 automatically.

---

## Expo EAS

**Purpose:** Expo Application Services interactions - build status, over-the-air update submissions, build queue inspection.

**Install:**

```bash
claude mcp add expo-eas --transport stdio -- npx @expo/mcp@latest
```

**Authentication:** Expo access token (stored locally by Expo CLI).

**User flow:**

1. Run `eas login` locally and authenticate with Expo credentials.
2. Run `eas whoami` to confirm the session.
3. The MCP reads the local Expo credentials file - no separate token configuration needed.

---

## GitHub

**Purpose:** Issue and pull request management, repository inspection, workflow triggering, release creation.

**Install:**

```bash
claude mcp add github --transport stdio -- npx @modelcontextprotocol/server-github@latest
```

**Authentication:** GitHub personal access token.

**User flow:**

1. Open `https://github.com/settings/tokens` (or fine-grained tokens page).
2. Generate a new token with scopes: `repo`, `workflow`, `read:org`.
3. Export as `GITHUB_TOKEN` in the shell where Claude Code runs.

**Security note:** Use fine-grained tokens scoped to specific repositories when possible. Rotate tokens periodically.

---

## Firebase

**Purpose:** Firestore queries, Firebase Auth user management, Cloud Storage inspection, Cloud Functions invocation.

**Install:**

```bash
claude mcp add firebase --transport stdio -- npx firebase-tools-mcp@latest
```

**Authentication:** Firebase service account JSON.

**User flow:**

1. Firebase Console → Project Settings → Service Accounts → Generate new private key.
2. Save the downloaded JSON to a secure location (e.g., `~/.firebase/service-account-<project>.json`).
3. Export as `GOOGLE_APPLICATION_CREDENTIALS` with the absolute path to the JSON file.

**Security note:** Never commit the service account JSON. Add its path to `.gitignore` if stored inside the project directory.

---

## Maestro

**Purpose:** React Native E2E flow execution (iOS and Android). Not technically an MCP at the time of writing - invoked as a CLI from within subagent execution. Listed here for completeness when the registry is consulted.

**Install:**

```bash
# macOS
curl -Ls "https://get.maestro.mobile.dev" | bash

# Verify
maestro --version
```

**Authentication:** None.

**Usage:** The Worker invokes `maestro test <flow.yaml>` or generates flow YAML inline during E2E scenario execution. Flow definitions live in the artifact path for review, not in the project source tree.

---

## Adding New MCPs

When a project requires an MCP not listed here, the Planner captures the following during Context Gathering:

- MCP name and primary purpose in the project
- Installation command (`claude mcp add ...` or platform equivalent)
- Authentication requirements: none, API key, service account, OAuth flow
- User-facing setup steps if authentication is needed
- Any scope guidance (project vs global) that matters for this MCP

The Planner writes these into the Spec's MCP Dependencies section in the same table format as registry entries. When multiple projects in an organization use the same custom MCP, contributors may extend this registry in their APM custom repository fork - the registry is intentionally separate from `SKILL.md` to enable this.

---

**End of Registry**
