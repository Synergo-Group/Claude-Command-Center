# Command Center plugin for Cowork

A persistent dashboard that lives in your Cowork sidebar and surfaces priority items from Gmail, Slack DMs, and Jira — with one-click "draft reply", "snooze", and "mark done" actions.

Works at any company that uses Gmail + Slack + Atlassian/Jira. The setup skill auto-detects your Atlassian site host and your internal email domain — no code changes needed per company.

---

## What you get

A Cowork artifact called **Command Center** that:

- Pulls live every time you open it: Gmail (last 7 days, inbox), Slack DMs (last 14 days, "to:me"), Jira (your assigned, non-Done issues).
- Buckets items into **Priority — needs you today**, **Needs your call — this week**, and **Stale — risk of going cold**.
- Lets you click "Draft reply to [name]" — Claude writes a draft, you edit, send to Gmail Drafts or Slack Drafts.
- Lets you snooze, mark done, or reopen — state persists across reloads.
- Friendly time-aware greeting and a metric strip (Open / Snoozed / Done today / Stale > 14d).

## Prerequisites

You need three connectors active in Cowork before installing:

- **Gmail** (the connector that exposes `search_threads` + `create_draft`)
- **Slack** (the connector that exposes `slack_search_public_and_private` + `slack_send_message_draft`)
- **Atlassian / Jira** (the connector that exposes `atlassianUserInfo`, `getAccessibleAtlassianResources`, `searchJiraIssuesUsingJql`)

If any of those aren't connected, the setup will tell you which one is missing and stop. Connect it via Cowork's connector settings, then re-run.

## Installing

### Option A — From a private GitHub repo (recommended for teams)

1. Push this folder to a private GitHub repo (e.g. `synergo/command-center-plugin`).
2. In Cowork, install the plugin pointing at that repo. (Path varies slightly by Cowork version — typically Settings → Plugins → Install from URL/repo.)
3. Restart Cowork or reload the conversation so it picks up the new skill.

### Option B — Local sideload

1. Copy the entire `command-center-plugin/` folder onto your machine.
2. Drop it into your local Cowork plugins directory. On Windows that's usually:
   ```
   C:\Users\<you>\AppData\Roaming\Claude\plugins\
   ```
3. Restart Cowork.

## Running it

In Cowork, just say:

> set up my command center

Claude will:

1. Verify your Gmail, Slack, and Jira connectors are present.
2. Auto-detect your Atlassian site host (from `getAccessibleAtlassianResources`) and your internal email domain (from `atlassianUserInfo`). If you have multiple Atlassian sites or use a generic email domain like gmail.com, Claude will ask which one to use.
3. Create the artifact in your Cowork sidebar.

First load takes 5–15 seconds while it pulls live data from each source. After that, it's instant.

## Daily use

- Click **Draft reply to [name]** on any item → edit the draft → save it to Gmail or Slack drafts.
- Click the **clock** icon on a card to snooze it.
- Click the green **check** to mark done.
- Reload the artifact any time you want fresh data — the list refetches; your done/snoozed history persists.

## State persistence

Done/snoozed status persists in `localStorage`. The diagnostic line under the greeting confirms how many items are remembered.

## Troubleshooting

- **"localStorage ✗" in the persistence banner**: your Cowork build is sandboxing storage. Done/snooze marks won't survive reloads. (An embedded-state fallback is wired in for that case but requires `mcp__cowork__update_artifact` to be callable from artifacts, which isn't always available.)
- **Loading screen never reveals**: click "Show what loaded so far" to see which MCP call is hung. Most likely a stale connector — disconnect/reconnect that one.
- **Slack DMs missing a recent message**: the artifact dedupes per channel (one entry per Slack thread). If you have a long DM with someone, only the latest message shows up.
- **Wrong items flagged as Priority**: the heuristic uses recency + sender external/internal. Edit `bucketOf()` in `artifact-template.html` to tune it.
- **0 Jira issues but you have tickets assigned**: the auto-detected `JIRA_HOST` may have been set wrong. Open the artifact source, find `const JIRA_HOST = '...'`, and check it matches your Atlassian site (e.g. `acme.atlassian.net`). Re-run the setup skill if needed.

## Customizing

The whole UI is in `skills/command-center-setup/artifact-template.html`. After install, your local Claude can edit it for you — say something like "open the command center template and add a Calendar section." Just remember to update the artifact afterwards (`mcp__cowork__update_artifact`).

---

Built by Valentin Beres · Synergo Group · 2026
