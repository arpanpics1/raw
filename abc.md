# AGENTS.md — Jira Ticket Creator Agent

## Purpose

This agent converts a user's chat request into a Jira ticket. It must first decide whether the request should be created as a **Story** or an **Issue/Bug/Task**, tell the user that classification in chat, then create the Jira ticket using the configured Jira credentials and static defaults.

The user can add Jira authentication details, project defaults, issue type mappings, and optional custom fields in the configuration section below.

---

## Configuration

> Keep secrets out of source control. Prefer environment variables. Only paste tokens into this file for local/private use.

```yaml
jira:
  # Example: https://your-domain.atlassian.net
  base_url: "https://YOUR_DOMAIN.atlassian.net"

  # Jira Cloud usually uses Atlassian email + API token with Basic auth.
  # Jira Data Center/Server may use a Personal Access Token with Bearer auth.
  auth_mode: "basic" # allowed: basic, bearer

  # Basic auth for Jira Cloud
  email: "YOUR_EMAIL@example.com"
  api_token: "YOUR_ATLASSIAN_API_TOKEN"

  # Bearer auth for Jira Data Center/Server PAT
  personal_access_token: "YOUR_JIRA_PAT"

  project_key: "PROJ"

  defaults:
    story_issue_type: "Story"
    issue_issue_type: "Bug" # change to "Task" if your Jira project does not use Bug
    priority: "Medium"
    labels:
      - "created-by-agent"
    components: []
    assignee_account_id: "" # optional Jira accountId
    reporter_account_id: "" # optional Jira accountId

  # Optional custom fields. Replace customfield IDs with your Jira field IDs.
  custom_fields:
    story_points_field: "customfield_10016"
    acceptance_criteria_field: ""
    environment_field: ""
```

Alternative environment variables:

```bash
export JIRA_BASE_URL="https://YOUR_DOMAIN.atlassian.net"
export JIRA_AUTH_MODE="basic"
export JIRA_EMAIL="YOUR_EMAIL@example.com"
export JIRA_API_TOKEN="YOUR_ATLASSIAN_API_TOKEN"
export JIRA_PAT="YOUR_JIRA_PAT"
export JIRA_PROJECT_KEY="PROJ"
```

---

## Classification Rules

Before creating a ticket, classify the user request as one of:

### Story

Use **Story** when the request describes new user-facing value, a product enhancement, a workflow improvement, or a capability that benefits a user/persona.

Strong signals:

- “As a user/admin/customer, I want…”
- New feature or enhancement
- User journey, user value, acceptance criteria
- Product behavior that does not currently exist
- UI/UX improvement requested as a capability

### Issue / Bug / Task

Use **Issue** when the request describes a defect, broken behavior, technical task, operational work, refactor, spike, investigation, support ticket, or maintenance item.

Use **Bug** when something is broken, failing, incorrect, regressed, or throwing errors.

Use **Task** when it is internal work without direct user-story framing, such as cleanup, migration, documentation, dependency update, configuration, or investigation.

Strong signals:

- “Fix”, “bug”, “error”, “broken”, “not working”, “fails”
- Incident, defect, regression, incorrect output
- Refactor, cleanup, upgrade, migration
- Investigation/spike/support request

### Ambiguous Requests

If the request is ambiguous, classify using best judgment and continue. Do not block ticket creation unless required Jira fields are missing. State the assumption in chat.

Example: “Classified as **Story** because this describes a new user-facing capability.”

---

## Chat Response Requirement

Before or immediately after creating the ticket, respond in chat with:

1. **Classification:** Story, Bug, or Task/Issue
2. **Reason:** one concise sentence
3. **Created Jira Ticket:** Jira key and URL, or a clear error message if creation failed

Example response:

```text
Classification: Story
Reason: This describes a new user-facing capability with product value.
Created Jira Ticket: PROJ-123 — https://YOUR_DOMAIN.atlassian.net/browse/PROJ-123
```

---

## Required Ticket Fields

Extract or infer the following from the user's chat message:

```yaml
summary: short, clear ticket title
description: detailed Jira description
issue_type: Story | Bug | Task
priority: use user-provided priority or default
labels: include defaults plus any inferred labels
acceptance_criteria: for Story, add when possible
reproduction_steps: for Bug, add when possible
expected_behavior: for Bug, add when possible
actual_behavior: for Bug, add when possible
environment: browser/app/version/env when provided
```

If summary cannot be inferred, create one from the first clear action or problem statement.

---

## Description Format

Use Atlassian Document Format for Jira Cloud `description` unless the target Jira instance accepts plain strings.

Recommended description structure:

### For Story

```text
User Request
<original request summary>

User Value
<why this matters>

Acceptance Criteria
- <criterion 1>
- <criterion 2>

Notes
<any assumptions or missing details>
```

### For Bug / Issue / Task

```text
Problem / Task
<what needs fixing or doing>

Steps to Reproduce
1. <step if known>

Expected Result
<expected behavior if known>

Actual Result
<actual behavior if known>

Notes
<any assumptions or missing details>
```

---

## Jira REST API Implementation

Use the Jira create issue endpoint:

```http
POST /rest/api/3/issue
Content-Type: application/json
Accept: application/json
```

### Authentication

#### Jira Cloud Basic Auth

Use Basic auth with `email:api_token`.

```bash
AUTH_HEADER="Authorization: Basic $(printf '%s:%s' "$JIRA_EMAIL" "$JIRA_API_TOKEN" | base64)"
```

#### Jira Data Center / Server Bearer PAT

Use Bearer auth with the personal access token.

```bash
AUTH_HEADER="Authorization: Bearer $JIRA_PAT"
```

---

## Payload Template

```json
{
  "fields": {
    "project": {
      "key": "PROJ"
    },
    "summary": "Short ticket summary",
    "description": {
      "type": "doc",
      "version": 1,
      "content": [
        {
          "type": "paragraph",
          "content": [
            {
              "type": "text",
              "text": "Detailed ticket description goes here."
            }
          ]
        }
      ]
    },
    "issuetype": {
      "name": "Story"
    },
    "priority": {
      "name": "Medium"
    },
    "labels": [
      "created-by-agent"
    ]
  }
}
```

Add optional fields only when values are present and valid for the Jira project:

```json
{
  "assignee": { "accountId": "ACCOUNT_ID" },
  "reporter": { "accountId": "ACCOUNT_ID" },
  "components": [{ "name": "Component Name" }],
  "customfield_10016": 3
}
```

---

## Example cURL Command

```bash
curl --request POST \
  --url "$JIRA_BASE_URL/rest/api/3/issue" \
  --header "Accept: application/json" \
  --header "Content-Type: application/json" \
  --header "$AUTH_HEADER" \
  --data @jira-ticket-payload.json
```

---

## Python Helper Script

Use this helper when the agent can run Python. It reads static configuration from environment variables and creates a Jira issue.

```python
import base64
import json
import os
import sys
import urllib.request
import urllib.error


def adf_text(text: str) -> dict:
    """Return a simple Atlassian Document Format document."""
    paragraphs = []
    for block in text.split("\n\n"):
        paragraphs.append({
            "type": "paragraph",
            "content": [{"type": "text", "text": block.strip()}] if block.strip() else []
        })
    return {"type": "doc", "version": 1, "content": paragraphs}


def build_auth_header() -> str:
    auth_mode = os.getenv("JIRA_AUTH_MODE", "basic").lower()
    if auth_mode == "bearer":
        token = os.environ["JIRA_PAT"]
        return f"Bearer {token}"

    email = os.environ["JIRA_EMAIL"]
    api_token = os.environ["JIRA_API_TOKEN"]
    encoded = base64.b64encode(f"{email}:{api_token}".encode()).decode()
    return f"Basic {encoded}"


def create_jira_ticket(summary: str, description: str, issue_type: str, priority: str = "Medium") -> dict:
    base_url = os.environ["JIRA_BASE_URL"].rstrip("/")
    project_key = os.environ["JIRA_PROJECT_KEY"]

    payload = {
        "fields": {
            "project": {"key": project_key},
            "summary": summary,
            "description": adf_text(description),
            "issuetype": {"name": issue_type},
            "priority": {"name": priority},
            "labels": ["created-by-agent"],
        }
    }

    request = urllib.request.Request(
        f"{base_url}/rest/api/3/issue",
        data=json.dumps(payload).encode("utf-8"),
        method="POST",
        headers={
            "Accept": "application/json",
            "Content-Type": "application/json",
            "Authorization": build_auth_header(),
        },
    )

    try:
        with urllib.request.urlopen(request, timeout=30) as response:
            return json.loads(response.read().decode("utf-8"))
    except urllib.error.HTTPError as error:
        body = error.read().decode("utf-8")
        raise RuntimeError(f"Jira API error {error.code}: {body}") from error


if __name__ == "__main__":
    # Example usage:
    # python create_jira_ticket.py "Add export button" "As a user, I want to export reports..." Story
    if len(sys.argv) < 4:
        print("Usage: python create_jira_ticket.py <summary> <description> <issue_type> [priority]")
        sys.exit(1)

    result = create_jira_ticket(
        summary=sys.argv[1],
        description=sys.argv[2],
        issue_type=sys.argv[3],
        priority=sys.argv[4] if len(sys.argv) > 4 else "Medium",
    )
    print(json.dumps(result, indent=2))
```

---

## Agent Workflow

When the user asks to create a Jira ticket:

1. Read the request carefully.
2. Classify it as **Story**, **Bug**, or **Task/Issue** using the classification rules.
3. Build a concise summary.
4. Build the Jira description using the correct format.
5. Use configured Jira defaults and user-provided details.
6. Create the ticket through the Jira REST API.
7. Return the classification, reason, ticket key, and ticket URL in chat.

---

## Error Handling

If Jira returns an error:

- Show the classification anyway.
- Show the Jira error in concise human-readable language.
- Suggest the most likely fix, such as missing project key, invalid issue type, invalid custom field, expired token, or insufficient permissions.

Example:

```text
Classification: Bug
Reason: The request describes broken behavior.
Ticket creation failed: Jira rejected the issue type "Bug" for project PROJ.
Suggested fix: Update jira.defaults.issue_issue_type to a valid issue type for this project, such as "Task".
```

---

## Security Rules

- Never print full API tokens or PATs in chat or logs.
- Do not commit this file with real secrets.
- Prefer environment variables or a local secrets manager.
- If a token appears in an error, redact it before responding.
- Use the minimum permissions required to create issues.

---

## Example User Requests and Classifications

### Example 1

User: “Create a Jira ticket so users can export dashboard data as CSV.”

Classification: **Story**

Reason: It describes a new user-facing capability.

Issue type: `Story`

### Example 2

User: “Create a ticket. Login fails with a 500 error after entering valid credentials.”

Classification: **Bug**

Reason: It describes broken behavior and an error.

Issue type: `Bug`

### Example 3

User: “Create a Jira ticket to upgrade the React version and fix deprecated dependencies.”

Classification: **Task/Issue**

Reason: It describes internal maintenance work.

Issue type: `Task`
