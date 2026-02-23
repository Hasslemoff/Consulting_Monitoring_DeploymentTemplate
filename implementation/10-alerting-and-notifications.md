# Phase 10 — Alerting and Notifications

## Objective

Configure email, Microsoft Teams, Slack, and PagerDuty media types. Create user groups with appropriate permissions. Build escalation actions that route alerts by severity with timed escalation steps.

## Prerequisites

- [ ] Phase 6 complete — Zabbix Server running and accessible via frontend
- [ ] SMTP relay available and configured to accept mail from Zabbix Server / proxy IPs
- [ ] Microsoft Teams incoming webhook URL obtained from channel administrator
- [ ] Slack App created with bot token (or plan to create one in this phase)
- [ ] PagerDuty integration key generated for the target service
- [ ] User accounts provisioned (local or LDAP/AD integrated)

> **Note:** This phase can run in parallel with Phases 09 and 12. No dependencies exist between alerting configuration and template/agent work.

---

## Step 1 — Set Global Zabbix URL Macro

All webhook-based media types (Teams, Slack, PagerDuty) include a link to the problem in Zabbix. This link is built from the `{$ZABBIX.URL}` global macro.

1. Navigate to **Administration > Macros**
2. Add or update the macro:

| Macro | Value |
|-------|-------|
| `{$ZABBIX.URL}` | `{{ZABBIX_FRONTEND_URL}}` |

Example: `https://zabbix.example.com`

> **Important:** Do not include a trailing slash. The webhook templates append paths like `/tr_events.php?triggerid=...` directly.

---

## Step 2 — Configure Email Media Type

### 2a. Edit the Built-in Email Media Type

1. Navigate to **Alerts > Media types**
2. Click on **Email** (built-in)
3. Configure:

| Field | Value |
|-------|-------|
| Name | `Email` |
| Type | Email |
| SMTP server | `{{SMTP_SERVER}}` |
| SMTP server port | `587` |
| SMTP helo | `{{SMTP_HELO}}` |
| SMTP email | `{{SMTP_FROM_EMAIL}}` |
| Connection security | STARTTLS |
| Authentication | Username and password (if relay requires it) |

4. Update the **Message format** to HTML for richer formatting.

### 2b. Configure Email Display Name

In the email "From" field, set the display name for clarity in inboxes:

| Field | Value |
|-------|-------|
| SMTP email | `{{SMTP_FROM_EMAIL}}` |

When users receive the email, it will appear as: `{{SMTP_FROM_NAME}} <{{SMTP_FROM_EMAIL}}>`

> **Note:** The display name is configured at the user media level, not the media type level. The media type controls the SMTP envelope sender.

### 2c. Test Email Delivery

1. Click **Test** on the Email media type
2. Enter a test recipient email address
3. Click **Test**
4. Verify the email arrives in the recipient's inbox (check spam/junk folders)

If the test fails, see the Troubleshooting section below.

---

## Step 3 — Configure Microsoft Teams Webhook

### 3a. Create Teams Incoming Webhook

In Microsoft Teams:

1. Navigate to the target channel (e.g., `#zabbix-alerts`)
2. Click the **...** menu next to the channel name > **Manage channel**
3. Navigate to **Connectors** (or **Workflows** if using the new Teams)
4. Create an **Incoming Webhook**:
   - Name: `Zabbix Alerts`
   - Icon: Upload the Zabbix logo (optional)
5. Copy the generated webhook URL — this becomes `{{TEAMS_WEBHOOK_URL}}`

> **Note:** If your organization uses the new Teams Workflows instead of legacy connectors, create a "Post to a channel when a webhook request is received" workflow and use that URL instead.

### 3b. Import and Configure the Media Type

1. Navigate to **Alerts > Media types > Import**
2. Import the Teams media type file:
   ```
   media_ms_teams_workflow.yaml
   ```
   This file is available in the Zabbix integrations repository under `media/microsoft_teams/`.

3. After import, click on the **MS Teams** media type
4. Set the parameters:

| Parameter | Value |
|-----------|-------|
| `teams_endpoint` | `{{TEAMS_WEBHOOK_URL}}` |

5. Verify `{$ZABBIX.URL}` is referenced in the webhook body for problem links.

### 3c. Test Teams Webhook

1. Click **Test** on the MS Teams media type
2. Provide sample data in the test fields
3. Click **Test**
4. Verify the message appears in the target Teams channel

---

## Step 4 — Configure Slack Integration

### 4a. Create a Slack App

If a Slack App does not already exist:

1. Navigate to `https://api.slack.com/apps`
2. Click **Create New App > From scratch**
3. Configure:
   - App Name: `Zabbix Alerts`
   - Workspace: Select your workspace
4. Navigate to **OAuth & Permissions > Scopes > Bot Token Scopes**
5. Add the following scopes:

| Scope | Purpose |
|-------|---------|
| `chat:write` | Post alert messages to channels |
| `groups:read` | Read private channel list for channel selection |
| `im:read` | Read direct message channel list |
| `channels:read` | Read public channel list for channel selection |

6. Click **Install to Workspace** and approve the permissions
7. Copy the **Bot User OAuth Token** — this becomes `{{SLACK_BOT_TOKEN}}`
8. Invite the bot to the target channel:
   ```
   /invite @Zabbix Alerts
   ```

### 4b. Import and Configure the Media Type

1. Navigate to **Alerts > Media types > Import**
2. Import the Slack media type file:
   ```
   media_slack.yaml
   ```

3. After import, click on the **Slack** media type
4. Set the parameters:

| Parameter | Value |
|-----------|-------|
| `bot_token` | `{{SLACK_BOT_TOKEN}}` |
| `channel` | `{{SLACK_CHANNEL}}` |

> **Important:** Store sensitive webhook parameters as **Secret** type in Zabbix to prevent them from being visible in the frontend or API responses.

### 4c. Test Slack Integration

1. Click **Test** on the Slack media type
2. Set "Send to" to the target channel (e.g., `#zabbix-alerts`)
3. Click **Test**
4. Verify the message appears in the Slack channel

---

## Step 5 — Configure PagerDuty Integration

### 5a. Import and Configure the Media Type

1. Navigate to **Alerts > Media types > Import**
2. Import the PagerDuty media type file:
   ```
   media_pagerduty.yaml
   ```

3. After import, click on the **PagerDuty** media type
4. Set the parameters:

| Parameter | Value |
|-----------|-------|
| `token` | `{{PAGERDUTY_INTEGRATION_KEY}}` |

> **Important:** Store sensitive webhook parameters as **Secret** type in Zabbix to prevent them from being visible in the frontend or API responses.

> **Note:** The integration key is specific to a PagerDuty **service**. If you need alerts routed to different PagerDuty services, create multiple PagerDuty media types with different integration keys, or use PagerDuty Event Orchestration to route by payload content.

### 5b. Test PagerDuty Integration

1. Click **Test** on the PagerDuty media type
2. Click **Test**
3. Verify an incident is created in PagerDuty (remember to resolve/suppress the test incident afterward)

---

## Step 6 — Create User Groups

Navigate to **Users > User groups > Create user group** for each of the following groups.

### 6a. Zabbix Admins

| Field | Value |
|-------|-------|
| Group name | `Zabbix Admins` |
| Frontend access | Internal (or LDAP if AD-integrated) |
| Enabled | Yes |

**Permissions** tab:

| Host group | Permission |
|------------|------------|
| All groups | Read-write |

**Users role:** Super admin or Admin

### 6b. NOC / Operations

| Field | Value |
|-------|-------|
| Group name | `NOC Operations` |
| Frontend access | LDAP |
| Enabled | Yes |

**Permissions** tab:

| Host group | Permission |
|------------|------------|
| All groups | Read |

**Users role:** User

> **Note:** This group represents the 24/7 operations team. All members receive alert notifications.

### 6c. Application Team Groups

Create one group per application team. Example:

| Field | Value |
|-------|-------|
| Group name | `App Team - Web Services` |
| Frontend access | LDAP |
| Enabled | Yes |

**Permissions** tab — restrict to relevant host groups only:

| Host group | Permission |
|------------|------------|
| Web Servers | Read |
| Load Balancers | Read |

Repeat for other application teams (e.g., `App Team - Database`, `App Team - Middleware`), granting each team read access only to their own host groups.

### 6d. Database Admins

| Field | Value |
|-------|-------|
| Group name | `Database Admins` |
| Frontend access | LDAP |
| Enabled | Yes |

**Permissions** tab:

| Host group | Permission |
|------------|------------|
| Database Servers | Read-write |
| Zabbix Infrastructure/Database | Read-write |

### 6e. Management

| Field | Value |
|-------|-------|
| Group name | `Management` |
| Frontend access | LDAP |
| Enabled | Yes |

**Permissions** tab:

| Host group | Permission |
|------------|------------|
| All groups | Read |

**Users role:** User (dashboard and report access only)

---

## Step 7 — Assign Media to Users

For each user, navigate to **Users > Users > [username] > Media** tab and add the appropriate media types.

### Example — NOC Team Member

| Media type | Send to | When active | Severity filter |
|------------|---------|-------------|-----------------|
| Email | `user@example.com` | `1-7,00:00-24:00` | Warning, Average, High, Disaster |
| MS Teams | `{{TEAMS_WEBHOOK_URL}}` | `1-7,00:00-24:00` | Average, High, Disaster |
| PagerDuty | `{{PAGERDUTY_INTEGRATION_KEY}}` | `1-7,00:00-24:00` | High, Disaster |

### Example — Management User

| Media type | Send to | When active | Severity filter |
|------------|---------|-------------|-----------------|
| Email | `manager@example.com` | `1-5,08:00-18:00` | Disaster |

> **Tip:** For webhook-based media types (Teams, Slack, PagerDuty), the "Send to" field is used as a parameter by the webhook script. Refer to each media type's documentation for the expected format.

---

## Step 8 — Create Escalation Actions

Navigate to **Alerts > Actions > Trigger actions > Create action** for each escalation policy.

### Action 1 — Warning: Email Team

| Field | Value |
|-------|-------|
| Name | `Warning - Email Team` |
| Conditions | Trigger severity = Warning |
| Enabled | Yes |

**Operations** tab:

| Step | Start | Duration | Operation |
|------|-------|----------|-----------|
| 1 | 1 | 0 (default) | Send message to user group: `NOC Operations` via `Email` |

**Recovery operations** tab:

| Operation |
|-----------|
| Notify all involved |

### Action 2 — High: Page After 15 Minutes

| Field | Value |
|-------|-------|
| Name | `High - Page After 15 Min` |
| Conditions | Trigger severity = High |
| Enabled | Yes |

**Operations** tab:

| Step | Start | Duration | Operation |
|------|-------|----------|-----------|
| 1 | 1 | 15m | Send message to user group: `NOC Operations` via `Email` |
| 2 | 2 | 0 | Send message to user group: `NOC Operations` via `PagerDuty` |

> **Explanation:** Step 1 fires immediately and has a 15-minute duration. If the problem is not acknowledged or resolved within 15 minutes, Step 2 fires and pages via PagerDuty.

**Recovery operations** tab:

| Operation |
|-----------|
| Notify all involved |

### Action 3 — Disaster: Immediate Page + Escalation

| Field | Value |
|-------|-------|
| Name | `Disaster - Immediate Page + Escalate` |
| Conditions | Trigger severity = Disaster |
| Enabled | Yes |

**Operations** tab:

| Step | Start | Duration | Operation |
|------|-------|----------|-----------|
| 1 | 1 | 30m | Send message to user group: `NOC Operations` via `PagerDuty` |
| 1 | 1 | 30m | Send message to user group: `NOC Operations` via `Email` |
| 2 | 2 | 30m | Send message to user group: `NOC Operations` via `PagerDuty` (repeat) |
| 3 | 3 | 0 | Send message to user group: `Management` via `PagerDuty` |
| 3 | 3 | 0 | Send message to user group: `Management` via `Email` |

> **Explanation:** Step 1 fires immediately via PagerDuty and email. If still unresolved after 30 minutes, Step 2 re-pages the NOC. After another 30 minutes (60 minutes total), Step 3 escalates to Management.

**Recovery operations** tab:

| Operation |
|-----------|
| Notify all involved |

### Action 4 — Average: Email + Teams/Slack

| Field | Value |
|-------|-------|
| Name | `Average — Email + Teams/Slack` |
| Conditions | Trigger severity = Average |
| Enabled | Yes |

**Operations** tab:

| Step | Start | Duration | Operation |
|------|-------|----------|-----------|
| 1 | 1 | 0 (default) | Send message to user group: `NOC Operations` via `Email` |
| 1 | 1 | 0 (default) | Send message to user group: `NOC Operations` via `MS Teams` |

**Recovery operations** tab:

| Operation |
|-----------|
| Notify all involved |

### Action Conditions — Additional Filtering

For all actions, consider adding these optional conditions to reduce noise:

| Condition | Value | Purpose |
|-----------|-------|---------|
| Maintenance status not in | Maintenance | Suppress alerts during planned windows |
| Problem is not suppressed | Yes | Respect manual suppression |
| Tag name = `notify` value = `true` | Only if used | Tag-based opt-in alerting |

---

## Step 9 — Configure Maintenance Windows

Navigate to **Data collection > Maintenance > Create maintenance period**.

### 9a. Understanding Maintenance Types

| Type | Behavior | When to Use |
|------|----------|-------------|
| **With data collection** | Hosts continue to be monitored, but trigger actions (alerts) are suppressed | Standard patching — you want the data but no false-positive pages |
| **No data collection** | All data collection stops entirely on affected hosts | Extended outages where data would be meaningless (e.g., host rebuild) |

> **Recommendation:** Use "With data collection" for most scheduled maintenance. This preserves monitoring continuity and avoids data gaps in graphs and reports.

### 9b. Recurring Maintenance — Patch Tuesday

Create a recurring monthly maintenance window for Microsoft Patch Tuesday:

| Field | Value |
|-------|-------|
| Name | `Patch Tuesday - Windows Servers` |
| Maintenance type | With data collection |
| Active since | (current date) |
| Active till | (1 year from now) |

**Periods** tab:

| Field | Value |
|-------|-------|
| Period type | Monthly |
| Month days | Second Tuesday of each month |
| Time | `02:00` to `06:00` (adjust to your patch window) |

**Host groups** tab:

| Host group |
|------------|
| Windows Servers |

### 9c. One-Time Maintenance Window

For ad-hoc maintenance (e.g., network switch upgrade):

| Field | Value |
|-------|-------|
| Name | `Network Switch Upgrade - Building A` |
| Maintenance type | With data collection |
| Active since | (start datetime) |
| Active till | (end datetime) |

Add the specific hosts or host groups affected.

> **Tip:** Set maintenance windows with a buffer — start 15 minutes before the actual work and end 15 minutes after. This accounts for pre-work checks and post-work service recovery time.

---

## Verification Checkpoint

Complete all checks before marking this phase done.

| # | Check | How to Verify | Expected Result | Pass |
|---|-------|---------------|-----------------|------|
| 1 | Email media type configured | Alerts > Media types > Email > Test | Test email received in inbox | [ ] |
| 2 | Teams webhook configured | Alerts > Media types > MS Teams > Test | Test message appears in Teams channel | [ ] |
| 3 | Slack integration configured | Alerts > Media types > Slack > Test | Test message appears in Slack channel | [ ] |
| 4 | PagerDuty integration configured | Alerts > Media types > PagerDuty > Test | Test incident created in PagerDuty | [ ] |
| 5 | User groups created | Users > User groups | 5+ groups with correct permissions | [ ] |
| 6 | Users have media assigned | Users > Users > [user] > Media tab | At least 1 media type per NOC user | [ ] |
| 7 | Warning action fires | Create test trigger (severity=Warning) | Email received by NOC group | [ ] |
| 8 | High action escalates | Create test trigger (severity=High), wait 15 min | PagerDuty page after timeout | [ ] |
| 9 | Disaster action pages immediately | Create test trigger (severity=Disaster) | PagerDuty + Email within 60 seconds | [ ] |
| 10 | Recovery notification sent | Resolve the test trigger | "Resolved" notification to all involved | [ ] |
| 11 | Maintenance window created | Data collection > Maintenance | Patch Tuesday recurring window visible | [ ] |
| 12 | `{$ZABBIX.URL}` macro set | Administration > Macros | URL matches `{{ZABBIX_FRONTEND_URL}}` | [ ] |

---

## Troubleshooting

### Email Relay Blocked

**Symptom:** Email test returns "Connection refused" or "Relay access denied."

**Resolution:**
1. Verify the SMTP relay accepts connections from the Zabbix Server IP:
   ```bash
   # From the Zabbix Server
   nc -zv {{SMTP_HELO}} 587
   ```
2. If the relay requires authentication, verify the username and password in the media type configuration.
3. If using Office 365 direct send, ensure the Zabbix Server's public IP (or NAT IP) is added to the Exchange connector's allowed sender list.
4. Check the Zabbix Server log for SMTP errors:
   ```bash
   grep -i "smtp\|email\|mail" /var/log/zabbix/zabbix_server.log | tail -20
   ```
5. Verify SPF/DKIM/DMARC records include the Zabbix Server if sending externally.

### Teams Webhook Returns 400 or 403

**Symptom:** Teams media type test returns HTTP 400 Bad Request or 403 Forbidden.

**Resolution:**
1. Verify the webhook URL is correct and has not been regenerated (old URLs become invalid).
2. If using the new Teams Workflows instead of legacy connectors, the URL format differs. Ensure the media type YAML matches the workflow endpoint format.
3. Test the webhook URL directly from the Zabbix Server:
   ```bash
   curl -H "Content-Type: application/json" \
     -d '{"text": "Test from Zabbix"}' \
     "{{TEAMS_WEBHOOK_URL}}"
   ```
4. If the webhook was created in a channel you no longer have access to, the webhook may have been disabled. Recreate it.
5. Check if your organization's Teams policy restricts incoming webhooks — contact your Teams administrator.

### PagerDuty Events Not Triggering

**Symptom:** PagerDuty media type test succeeds but no incident appears in PagerDuty.

**Resolution:**
1. Verify the integration key type — it must be an **Events API v2** integration key, not a REST API key.
2. Check that the PagerDuty service is not in maintenance mode.
3. Verify the integration key belongs to the correct PagerDuty service:
   - PagerDuty > Services > [Service] > Integrations tab
4. Test the Events API directly:
   ```bash
   curl -X POST "https://events.pagerduty.com/v2/enqueue" \
     -H "Content-Type: application/json" \
     -d '{
       "routing_key": "{{PAGERDUTY_INTEGRATION_KEY}}",
       "event_action": "trigger",
       "payload": {
         "summary": "Test from Zabbix",
         "severity": "critical",
         "source": "zabbix-server"
       }
     }'
   ```
5. If the curl test succeeds but Zabbix does not, check the media type script parameters for formatting issues.

### Escalation Steps Not Firing

**Symptom:** Step 1 of an action fires, but Step 2 (escalation) never executes.

**Resolution:**
1. Check the **step duration** — if set to `0`, the action completes at Step 1 and does not escalate. Set the duration to the desired wait time (e.g., `900` for 15 minutes).
2. Verify the problem was not acknowledged — by default, acknowledgment stops escalation. Check the action's "Pause operations for acknowledged problems" setting.
3. Verify the problem was not resolved before the escalation timer expired.
4. Check **Alerts > Actions > Action log** for the specific action to see which steps executed and any errors.
5. Verify the user in the escalation step has the correct media type configured and active during the current time period.

### Slack Bot Cannot Post

**Symptom:** Slack media type test returns "channel_not_found" or "not_in_channel."

**Resolution:**
1. Ensure the Slack bot has been invited to the target channel:
   ```
   /invite @Zabbix Alerts
   ```
2. For private channels, the `groups:read` scope is required in addition to `chat:write`.
3. Verify the bot token starts with `xoxb-` (bot token), not `xoxp-` (user token).
4. If the channel was recently created, wait a few minutes for Slack's internal caching to update.
