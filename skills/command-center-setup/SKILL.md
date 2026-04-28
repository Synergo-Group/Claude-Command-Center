---
name: command-center-setup
description: Set up the user's personal Command Center artifact in Cowork — a live dashboard pulling priority items from their Gmail inbox, Slack DMs, and assigned Jira tickets, with quick-action buttons for drafting replies, snoozing, and marking done. Trigger this skill whenever the user says any of "set up my command center", "install command center", "build me a command center", "I want a daily dashboard", "create my morning dashboard", or asks for a single place to triage their inbox + Slack + Jira together. Also trigger if they mention they got this skill from a teammate and want to install it.
---

# Command Center setup

You are setting up a persistent Cowork artifact called "Command Center" — a single dashboard that pulls live data from the user's Gmail inbox, Slack direct messages, and Jira issues, organized into Priority / This Week / Stale buckets, with buttons for drafting replies, snoozing, and marking done.

## What you need to do

1. **Verify connectors are present.** Check the available MCP tools in this session. You need:
   - A Gmail MCP that exposes `search_threads` and `create_draft` (the tool names will look like `mcp__<uuid>__search_threads`).
   - A Slack MCP that exposes `slack_search_public_and_private` and `slack_send_message_draft`.
   - An Atlassian/Jira MCP that exposes `atlassianUserInfo`, `getAccessibleAtlassianResources`, and `searchJiraIssuesUsingJql`.

   If any of those connectors aren't connected, tell the user which one is missing and stop. They need to connect it in Cowork's connector settings before you can proceed. Don't try to substitute, don't fall back to mock data.

2. **Capture the exact MCP tool names** from the available tool list. The UUIDs in the tool names are unique per user, so you must use whatever names appear in *this* user's session — do not copy hard-coded UUIDs from anywhere.

3. **Read the artifact template.** It's at `artifact-template.html` in this skill's directory. The template has placeholder tokens like `{{GMAIL_SEARCH}}` that you need to replace.

4. **Substitute the MCP tool placeholders** with the exact MCP tool names from step 2:

   | Placeholder              | Replace with the tool name for                         |
   |--------------------------|--------------------------------------------------------|
   | `{{ATLASSIAN_USER_INFO}}`| Atlassian's `atlassianUserInfo`                        |
   | `{{ATLASSIAN_RESOURCES}}`| Atlassian's `getAccessibleAtlassianResources`          |
   | `{{JIRA_SEARCH}}`        | Atlassian's `searchJiraIssuesUsingJql`                 |
   | `{{GMAIL_SEARCH}}`       | Gmail's `search_threads`                               |
   | `{{GMAIL_DRAFT}}`        | Gmail's `create_draft`                                 |
   | `{{SLACK_SEARCH}}`       | Slack's `slack_search_public_and_private`              |
   | `{{SLACK_DRAFT}}`        | Slack's `slack_send_message_draft`                     |

   Each token should be replaced with the FULL tool name including the `mcp__<uuid>__` prefix. Do not invent UUIDs — pull them from the actual tool names you see in your available tool list.

5. **Detect the user's Atlassian site host** and substitute it for `{{JIRA_HOST}}`. Call `getAccessibleAtlassianResources` once. The response is an array; take `result[0].url` (e.g. `https://acme.atlassian.net`) and use just the hostname (`acme.atlassian.net`) — strip the `https://` prefix and any trailing slash. If the user has multiple sites in the response, ask them which one they want the dashboard pointed at. This value is used both as a `cloudId` fallback for the JQL search and to build "Open in Jira" links, so it has to be exact.

6. **Detect the user's internal email domain keyword** and substitute it for `{{INTERNAL_DOMAIN_KEYWORD}}`. Call `atlassianUserInfo` to get the user's email. Take the domain part (everything after `@`), drop the TLD (`.com`, `.group`, `.io`, `.co.uk`, etc.), and lowercase the result. Examples:
   - `valentin@synergo.group` → `synergo`
   - `jane@acme-corp.com` → `acme-corp`
   - `bob@example.co.uk` → `example`

   If the email lookup fails or the user has a generic gmail.com / outlook.com address, ask them: "What's the lowercase keyword that identifies emails from your team? (e.g. `acme` for everyone at acme.com)". The artifact uses this as a case-insensitive substring check, so it's forgiving — a partial match is fine.

7. **Create the artifact** by calling `mcp__cowork__create_artifact` with:
   - `id: "command-center"` (kebab-case slug)
   - `html`: the substituted template body, complete and self-contained
   - `description`: "Live priority dashboard pulling from Gmail, Slack DMs, and Jira"
   - `mcp_tools`: the array of fully-qualified MCP tool names you used

8. **Verify no placeholders survived** before passing the HTML to `create_artifact`. Search the substituted string for `{{` — if any survive, you missed a substitution and the artifact will silently break. The full set you must replace is: `{{ATLASSIAN_USER_INFO}}`, `{{ATLASSIAN_RESOURCES}}`, `{{JIRA_SEARCH}}`, `{{GMAIL_SEARCH}}`, `{{GMAIL_DRAFT}}`, `{{SLACK_SEARCH}}`, `{{SLACK_DRAFT}}`, `{{JIRA_HOST}}`, `{{INTERNAL_DOMAIN_KEYWORD}}`.

9. **Tell the user it's ready** — one or two short sentences. Mention that the first load will take 5–15 seconds while it pulls live data from each source, and that done/snoozed items persist across reloads via localStorage.

## Common pitfalls

- **Don't hard-code the UUIDs.** They're per-user. The whole point of substitution is that this skill works for any teammate who installs it.
- **Don't skip the connector check.** If Slack isn't connected and you create the artifact anyway, the loader will time out and the user will think the artifact is broken.
- **Don't read the template, edit it inline in your reasoning, and paste it back unchanged.** Use a proper string replacement so the output HTML actually has the user's MCP tool names baked in.
- **The artifact is large** (~700 lines of HTML). Pass it to `create_artifact` as a single complete document including the `<!doctype html>` line.

## If the user already has a Command Center

Call `mcp__cowork__list_artifacts` first. If `command-center` is already there, ask whether they want to overwrite it (use `update_artifact`) or rename this one (use a different `id`). Default to overwriting if they want a fresh install.
