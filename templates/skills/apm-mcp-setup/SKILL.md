---
name: apm-mcp-setup
description: Standards and registry for coordinating MCP installation with the User during Manager initialization and at runtime.
---

# APM {VERSION} - MCP Setup Skill

## 1. Overview

**Reading Agents:** Planner, Manager

This skill defines how MCP (Model Context Protocol) dependencies are declared, coordinated, and installed throughout an APM session. The Planner consults it when capturing external service needs during Context Gathering and writing the Spec's MCP Dependencies section during Work Breakdown. The Manager consults it during First Initiation to guide the User through installation and at runtime when a Worker reports a missing MCP.

### 1.1 Outputs

- *Spec's MCP Dependencies section:* Written by the Planner during Work Breakdown.
- *User-facing setup instructions:* Rendered by the Manager during First Initiation and on runtime missing-MCP events.

---

## 2. Declaration

The Planner writes the Spec's `## MCP Dependencies` section during Work Breakdown when the project requires MCPs. Each entry follows this format:

```markdown
## MCP Dependencies

| MCP | Purpose | Install | Auth |
|-----|---------|---------|------|
| playwright | E2E validation for web flows | `claude mcp add playwright --transport stdio -- npx @playwright/mcp@latest` | None |
| supabase | Database seeding for E2E data setup | `claude mcp add supabase --transport stdio -- npx @supabase/mcp-server-supabase@latest` | Service role key from Supabase dashboard |
```

Entries reference `mcp-registry.md` alongside this skill for canonical install commands and authentication flows. When a project requires an MCP not in the registry, the Planner asks the User during Context Gathering for installation and authentication details and documents them in the table.

**Scope decision.** Declare an MCP in Dependencies only when the project actively needs it for execution - E2E validation, data setup, deployment coordination. Do not list MCPs the User has installed globally for convenience but that no Task actually requires.

---

## 3. Manager-Owned Setup Flow

During First Initiation (after the understanding summary and Version Control conventions are approved, before first dispatch), the Manager reads the Spec's MCP Dependencies section and presents setup guidance to the User.

**Presentation pattern:**

```
Before the first Task dispatches, these MCPs need to be installed:

1. Playwright MCP - required for E2E validation on web flows.
   Install: `claude mcp add playwright --transport stdio -- npx @playwright/mcp@latest`
   No authentication needed.

2. Supabase MCP - required for database seeding during E2E scenarios.
   Install: `claude mcp add supabase --transport stdio -- npx @supabase/mcp-server-supabase@latest`
   Authentication: open Supabase dashboard → Project Settings → API, copy the
   service_role key (not the anon key), then set it as SUPABASE_SERVICE_ROLE_KEY
   or pass via the MCP server args.

Once each MCP is installed and authenticated, confirm and I'll proceed with the first dispatch.
```

**Scope guidance to include when relevant.** When an MCP supports project-scoped installation (e.g., Claude Code's `--scope project`), the Manager offers both options and lets the User choose. Project scope is typical when credentials are project-specific (Supabase, Vercel, Firebase); global scope is typical for generic tools (Playwright, Chrome DevTools).

The Manager pauses after presenting and waits for User confirmation before dispatching the first Task. If the User reports installation issues, the Manager provides troubleshooting hints from the registry; if the issue is unresolved, the Manager suggests proceeding without the MCP and flagging affected Tasks as Partial at runtime for retry after resolution.

---

## 4. Runtime Missing-MCP Handling

When a Worker reports that a required MCP is missing (E2E execution returned `blocked`, or the Worker detected the MCP unavailable during execution), the Manager:

1. Reads the Task Log's Issues section for the missing MCP name.
2. Re-presents setup guidance for that specific MCP from the registry, formatted as §3 Presentation pattern.
3. Pauses until the User confirms installation.
4. Dispatches a follow-up Task Prompt using the original `log_path` per `{GUIDE_PATH:task-assignment}` §3.4 Follow-Up Task Prompt Construction.

Workers never install MCPs themselves; the Manager owns all installation coordination. This keeps Worker scope clean and ensures the User consistently encounters setup guidance from one authoritative voice.

---

## 5. Registry

The canonical list of known MCPs with installation commands and authentication flows lives in `mcp-registry.md` alongside this skill. When the Planner captures an MCP requirement or the Manager presents setup guidance, both resolve entries from the registry.

The registry is intentionally kept in a separate file so it can be updated without touching this skill's standards. Projects using APM custom repositories may extend the registry for organization-specific MCPs by adding entries to their forked copy.

---

**End of Skill**
