# Discord Event Notifier v2

An n8n automation that watches a Google Sheet for event submissions and automatically:
- Assigns a unique tracking ID to new events
- Posts formatted announcements to Discord based on event status (Scheduled / Draft / Cancelled)
- Emails registered participants when an event is scheduled or cancelled

Built for a community/club (e.g. a learning club) that manages events through a Google Form → Google Sheet pipeline and wants zero-touch Discord + email notifications whenever an organizer updates an event's status.

---

## How it works

**Trigger:** Polls a Google Sheet ("Form Responses 1" tab) every minute for new or updated rows.

**Flow:**

1. **Google Sheets Trigger** — detects a new/changed row in the events sheet.
2. **Code in JavaScript** — parses the raw `Event Time` field (format: `date | start - end`) into separate `date`, `startTime`, `endTime`, `startDateTime`, and `endDateTime` fields.
3. **If — Has Event ID?**
   - **No (new row):** → **Edit Fields** generates a unique tracking ID (`EVT-MM-HHmmss`) → **Update row in sheet** writes the ID and parsed date/time fields back to the sheet.
   - **Yes (already tracked):** → routes to the **Switch** node based on the row's `Status` column.
4. **Switch — Status**
   - **`Scheduled`** → **If1** confirms status → in parallel:
     - **Scheduled Events Notifs** posts a rich `@everyone` Discord announcement (date, time, venue, speaker, description, registration link, tracking ID).
     - **Get row(s) in sheet1** looks up matching participants in the "Participants" tab → **Send a message** emails each participant a "your event has been scheduled" notice.
   - **`Draft`** → **Drafted Event Notifs** posts an internal-only Discord log (visible to organizers, not `@everyone`) noting a new draft event was created and needs to be manually set to "Scheduled" to go live.
   - **`Cancelled`** → **If2** confirms status → in parallel:
     - **Cancellation Notif** posts a Discord cancellation announcement.
     - **Get row(s) in sheet** looks up matching participants → **Send a message1** emails each participant a cancellation notice.

---

## Prerequisites

| Service | Used for | Node(s) |
|---|---|---|
| **Google Sheets API** (OAuth2) | Reading/writing the events sheet & looking up participants | `Google Sheets Trigger`, `Update row in sheet`, `Get row(s) in sheet`, `Get row(s) in sheet1` |
| **Discord Webhook** | Posting announcements to a channel | `Scheduled Events Notifs`, `Drafted Event Notifs`, `Cancellation Notif` |
| **Gmail API** (OAuth2) | Emailing participants | `Send a message`, `Send a message1` |

You'll also need a Google Sheet with (at minimum) these two tabs:

- **Form Responses 1** — one row per event, with columns including: `Event Time` (format `M/D/YYYY | 3:00 PM - 5:00 PM`), ` Event Name` *(note the leading space)*, `Status` (`Draft` / `Scheduled` / `Cancelled`), `Created By`, `Venue`, `Speaker`, `Event Description`, `Registration Link`, `Event ID` (auto-filled by the workflow).
- **Participants** — one row per attendee, with columns including `Event Name`, `Name`, `Gmail`.

> ⚠️ Note the column name ` Event Name` has a leading space in several nodes — this must match exactly or the lookups/filters will silently fail.

---

## Setup

1. **Import the workflow**
   - In n8n: **Workflows → Import from File** → select `Discord_Event_Notifier_v2.json`.

2. **Replace the placeholders** — this file has been sanitized of all personal credentials/links. Search for each of the following inside the workflow JSON or n8n's node UI and fill in your own values:

   | Placeholder | What to put there |
   |---|---|
   | `YOUR_GOOGLE_SHEET_ID` | Your Google Sheet's ID (the long string in its URL between `/d/` and `/edit`) |
   | `[INSERT YOUR GOOGLE SHEET SHARE LINK HERE]` | Your sheet's shareable link (used inside the "Drafted Event Notifs" Discord message so organizers can jump straight to it) |
   | `YOUR_GOOGLE_SHEETS_TRIGGER_CREDENTIAL_ID` | Your own Google Sheets Trigger OAuth2 credential (create this in n8n's Credentials panel, then reassign it on the `Google Sheets Trigger` node) |
   | `YOUR_GOOGLE_SHEETS_CREDENTIAL_ID` | Your own Google Sheets OAuth2 credential (reassign on `Update row in sheet`, `Get row(s) in sheet`, `Get row(s) in sheet1`) |
   | `YOUR_GMAIL_CREDENTIAL_ID` | Your own Gmail OAuth2 credential (reassign on both `Send a message` nodes) |
   | `YOUR_DISCORD_WEBHOOK_CREDENTIAL_ID` | Your own Discord Webhook credential (reassign on `Scheduled Events Notifs`, `Drafted Event Notifs`, `Cancellation Notif`) |
   | `YOUR_N8N_INSTANCE_ID` | Not required to change — n8n generates this automatically on import |

   In practice, the easiest path is: import the workflow, then click into each node showing a credential error, and select/create the correct credential from n8n's UI — n8n will handle re-linking it for you rather than requiring manual JSON edits.

3. **Set the Google Sheet reference** on each Google Sheets node: open the node, and in the **Document** dropdown pick your sheet, and in **Sheet** pick the correct tab (`Form Responses 1` or `Participants`). This will auto-populate the ID/URL fields.

4. **Point the Discord Webhook credential** at your own channel webhook URL (created in Discord under **Channel Settings → Integrations → Webhooks**).

5. **Activate the workflow.**

---

## Example: Event row → Discord announcement

**Sheet input:**
| Event Time | Event Name | Status | Venue | Speaker |
|---|---|---|---|---|
| `8/12/2026 \| 3:00 PM - 5:00 PM` | AWS re:Cap Session | Scheduled | Zoom | Jane Doe |

**Discord output (abridged):**
```
📅 NEW EVENT SCHEDULED
📢 AWS re:Cap Session
📅 Date: 8/12/2026
🕒 Time: 3:00 PM - 5:00 PM
📍 Venue: Zoom
🎤 Speaker: Jane Doe
🆔 Tracking ID: EVT-08-150322
```

---

## Notes / Limitations

- The workflow polls every minute — for larger sheets or heavier use, consider switching to a Google Sheets **onChange** trigger or increasing the polling interval to reduce API quota usage.
- Status matching is case-insensitive but expects exact values: `Draft`, `Scheduled`, `Cancelled`.
- Participant emails are matched by exact `Event Name` string — trailing/leading whitespace in the sheet will break the lookup.

---

# How to Use — Discord Event Notifier v2 (n8n)

A step-by-step walkthrough for getting this workflow running on your own n8n instance (cloud or self-hosted).

---

## 1. Prerequisites checklist

Before importing, make sure you have:

- [ ] An n8n instance (cloud, self-hosted, or Docker) you can log into
- [ ] A Google account with a Sheet set up for events (see structure below)
- [ ] A Discord server where you have permission to create a webhook
- [ ] A Gmail account you're okay sending notification emails from

---

## 2. Set up your Google Sheet

Create a Google Sheet with two tabs:

**Tab 1: `Form Responses 1`** (one row per event)

| Column | Purpose |
|---|---|
| `Event Time` | Format: `M/D/YYYY \| 3:00 PM - 5:00 PM` |
| ` Event Name` | ⚠️ has a leading space — must match exactly |
| `Status` | `Draft`, `Scheduled`, or `Cancelled` |
| `Created By` | Organizer name |
| `Venue` | Location or link |
| `Speaker` | Speaker name |
| `Event Description` | Short blurb |
| `Registration Link` | Signup URL |
| `Event ID` | Leave blank — the workflow fills this in automatically |

**Tab 2: `Participants`**

| Column | Purpose |
|---|---|
| `Event Name` | Must match ` Event Name` from Tab 1 (minus the leading space handling is done in the lookup, but keep names consistent) |
| `Name` | Participant's name |
| `Gmail` | Participant's email address |

Tip: connect a Google Form to `Form Responses 1` so organizers submit events through a form instead of editing the sheet directly.

---

## 3. Create a Discord webhook

1. In Discord, go to **Server Settings → Integrations → Webhooks**.
2. Click **New Webhook**, pick the channel you want announcements posted to.
3. Copy the **Webhook URL** — you'll paste this into n8n in step 5.

---

## 4. Import the workflow into n8n

1. Open n8n.
2. Go to **Workflows** → click **Import from File** (or **⋮ menu → Import from File**).
3. Select `Discord_Event_Notifier_v2.json`.
4. The workflow will load with all nodes, but several will show a **credential error (red exclamation icon)** — that's expected, since the credentials were stripped out before sharing.

---

## 5. Reconnect credentials

Click into each flagged node and set up its credential:

**Google Sheets nodes** (`Google Sheets Trigger`, `Update row in sheet`, `Get row(s) in sheet`, `Get row(s) in sheet1`):
1. Open the node → **Credential** dropdown → **Create New Credential**.
2. Choose **Google Sheets OAuth2 API**, sign in with your Google account, and grant access.
3. In the same node, set the **Document** field to your event sheet (search by name or paste the URL), and set **Sheet** to the correct tab (`Form Responses 1` or `Participants`).

**Discord nodes** (`Scheduled Events Notifs`, `Drafted Event Notifs`, `Cancellation Notif`):
1. Open the node → **Credential** dropdown → **Create New Credential**.
2. Choose **Discord Webhook API** and paste the webhook URL from step 3.

**Gmail nodes** (`Send a message`, `Send a message1`):
1. Open the node → **Credential** dropdown → **Create New Credential**.
2. Choose **Gmail OAuth2 API**, sign in, and grant access.

> Once you create a credential for one node (e.g. Google Sheets), you can reuse it on the other nodes of the same type — no need to recreate it each time.

---

## 6. Fix the hardcoded sheet link in the "Drafted Event Notifs" message

This node's Discord message includes a line pointing organizers back to the sheet:

```
To change status, go to Google Sheet Link
[INSERT YOUR GOOGLE SHEET SHARE LINK HERE]
```

Open the `Drafted Event Notifs` node, find that placeholder in the **Content** field, and replace it with your sheet's shareable link (**Share → Copy link** in Google Sheets).

---

## 7. Test it

1. Click **Execute Workflow** (top right) to run it manually once, or add a test row to your sheet and wait up to 60 seconds for the trigger to pick it up.
2. Add a new row in `Form Responses 1` with `Status` = `Draft` and no `Event ID`.
   - Expect: the workflow fills in `Event ID` and posts an internal log to Discord.
3. Change that row's `Status` to `Scheduled`.
   - Expect: a public `@everyone` Discord announcement, and emails sent to matching participants.
4. Change the same row's `Status` to `Cancelled`.
   - Expect: a Discord cancellation notice, and cancellation emails sent.

---

## 8. Activate

Once tests pass, toggle the workflow to **Active** (top right switch). It will now poll your sheet every minute automatically — no manual runs needed.

---

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| Nothing happens after changing Status | Trigger polling interval (1 min) hasn't elapsed yet, or the sheet/tab selected in the trigger doesn't match your actual sheet |
| Participant emails not sent | `Event Name` in `Participants` tab doesn't exactly match ` Event Name` (check for typos/extra spaces) |
| Discord message doesn't post | Webhook credential invalid, or the webhook was deleted/regenerated in Discord |
| Event ID never gets assigned | Row's `Event ID` column isn't actually empty (e.g. contains a space) — the `If` node checks for a truly empty value |
