# Jira Update

A single Jira command for anyone in 7EDGE to check their own work, check someone else's work, or view a full board — all in one place.

---

## STEP 1 — Check for Help Request

If the user types `/PM/jira-update help` or `/PM/jira-update ?`, display the following help guide and stop — do not run any Jira queries:

---
### 📖 Help: How to Use /PM/jira-update

This command connects to Jira and gives you a live update. Here's what you can do:

| What you type | What you get |
|---|---|
| `/PM/jira-update` | Your own pending cards + comments where you've been tagged but haven't replied |
| `/PM/jira-update [person name]` | That person's pending cards + comments where they've been tagged but haven't replied |
| `/PM/jira-update [board name]` | Full board status — all active tickets, who they're assigned to, and latest comments |
| `/PM/jira-update [person name] [board name]` | That person's pending cards on that specific board + tickets where they've been tagged |
| `/PM/jira-update help` | Shows this help guide |

**Examples:**
- `/PM/jira-update` → shows your own Jira update
- `/PM/jira-update Shafana CC` → shows Shafana's cards on the CC board + where she's been tagged
- `/PM/jira-update Continuous Crusaders` → shows the full CC board status
- `/PM/jira-update Sagar` → shows all of Sagar's pending cards across all boards

**What counts as "pending"?**
Cards with status: In Queue, Ready, In Progress, In Design, In Review, Review, QA, In QA, UAT, Blocked — basically anything active that hasn't reached Done or Deployed yet.

**What counts as "unacknowledged mention"?**
A ticket where someone tagged you in a comment, and the last comment on that ticket is NOT from you — meaning you haven't replied yet. This checks all tickets across the board, not just ones assigned to you.

---

## STEP 2 — Identify the Current User

Always call the Atlassian MCP "get current user" endpoint first to get:
- Current user's display name
- Current user's account ID

Use these dynamically throughout. Never hardcode any name or ID.

---

## STEP 3 — Detect Intent from Input

Read what the user typed after `/PM/jira-update` and decide:

| Input | Mode |
|---|---|
| Nothing | **My Check** |
| `help` or `?` | **Help** (handled in Step 1) |
| A person's name only | **Person Check** |
| A board or project name only | **Board Check** |
| A person's name + a board/project name | **Person + Board Check** |

If the input is ambiguous (one word that could be a name or a project), try resolving it as both via MCP. Use whichever returns a valid result. If both match, ask the user to clarify.

---

## STEP 4 — Run the Right Check

---

### MY CHECK (no input)

**Part A — Assigned Cards**
Search for issues where:
- Assignee = current user's account ID
- Status IN: In Queue, Ready, In Progress, In Design, In Review, Review, QA, In QA, UAT, Blocked
  *(i.e. active statuses between Ready and Done — exclude Backlog, Cancelled, Done, Deployed, Closed, Released, Resolved)*

**Part B — Unacknowledged Mentions**
Search for ALL issues (across all projects) where:
- A comment contains a mention of the current user's account ID
- The ticket status is NOT Done, Deployed, Cancelled, Closed, Released, Resolved
- The last comment on the ticket was NOT written by the current user
  *(This means someone is waiting for their reply — regardless of who the ticket is assigned to)*

**Output:**
Greet the user by first name.

📌 **My Pending Cards (X items)**
For each: [ticket key with link] · summary · status · project · priority · last updated

💬 **Needs My Reply (X items)**
For each: [ticket key with link] · summary · project · who tagged me · when they tagged me
*(Include tickets assigned to others where the user has been mentioned)*

> Summary: "You have X pending cards and Y unacknowledged mentions. Most urgent: [ticket key]."

---

### PERSON CHECK ([person name])

Use MCP to resolve the person's name to their Jira account ID.

**Part A — Assigned Cards**
Search for issues where:
- Assignee = resolved person's account ID
- Status IN: In Queue, Ready, In Progress, In Design, In Review, Review, QA, In QA, UAT, Blocked

**Part B — Unacknowledged Mentions**
Search for ALL issues (across all projects) where:
- A comment mentions this person's account ID
- Ticket status is NOT Done, Deployed, Cancelled, Closed, Released, Resolved
- The last comment was NOT written by this person

**Output:**

📋 **[Person Name]'s Pending Cards (X items)**
For each: [ticket key with link] · summary · status · project · priority · last updated · latest comment (author + text)

💬 **[Person Name]'s Unacknowledged Mentions (X items)**
For each: [ticket key with link] · summary · project · assignee · who tagged them · when
*(Include tickets they don't own but have been tagged on)*

> Summary: "[Person Name] has X pending cards and Y unacknowledged mentions."

---

### BOARD CHECK ([board or project name])

Use MCP to resolve the board/project name to its project key.

Search for all issues in that project where:
- Status IN: In Queue, Ready, In Progress, In Design, In Review, Review, QA, In QA, UAT, Blocked

**Output:**

🗂️ **Board Status: [Project Name] — X active tickets**

Group by status (In Progress → In Design → Review → QA → Blocked → In Queue/Ready):

For each ticket:
```
[KEY] Summary
Assignee: Name | Status: X | Priority: Y | Updated: Date
💬 Latest comment (Author, Date): "comment text..."
```

> Dependency summary: "Tickets flagged as blocked or waiting: [list any where latest comment indicates a blocker, dependency, or unanswered question]."

---

### PERSON + BOARD CHECK ([person name] [board or project name])

Use MCP to resolve both the person's name and the board/project name.

**Part A — Assigned Cards on that Board**
Search for issues where:
- Assignee = resolved person's account ID
- Project = resolved project key
- Status IN: In Queue, Ready, In Progress, In Design, In Review, Review, QA, In QA, UAT, Blocked

**Part B — Unacknowledged Mentions on that Board**
Search for issues where:
- Project = resolved project key
- A comment mentions this person's account ID
- Ticket status is NOT Done, Deployed, Cancelled, Closed, Released, Resolved
- The last comment was NOT written by this person
- *(Include tickets on this board assigned to anyone, not just this person)*

**Output:**

📋 **[Person Name]'s Pending Cards on [Project Name] (X items)**
For each: [ticket key with link] · summary · status · priority · last updated · latest comment (author + text)

💬 **[Person Name]'s Unacknowledged Mentions on [Project Name] (X items)**
For each: [ticket key with link] · summary · assignee · who tagged them · when
*(This is where CC-4313-style tickets get caught — assigned to someone else, but this person was tagged)*

> Summary: "[Person Name] has X pending cards and Y unacknowledged mentions on [Project Name]."

---

## General Rules

- Always use Atlassian MCP — never hardcode IDs, names, or project keys
- Always include direct ticket links: https://7edge.atlassian.net/browse/[KEY]
- Unacknowledged mentions must search ALL tickets on the board/project, not just ones assigned to the person
- If a person or project cannot be resolved, say so clearly and ask for clarification
- Keep output concise and actionable
- If no results found for any section, say "✅ All clear — nothing pending here"
