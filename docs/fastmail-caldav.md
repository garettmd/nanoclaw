# Fastmail CalDAV Integration for NanoClaw

## Overview

This guide sets up CalDAV integration with Fastmail so you can read, create, and update calendar events from NanoClaw agents.

## Tools We'll Use

**vdirsyncer** - Syncs CalDAV calendars to local storage
**khal** - CLI calendar interface for reading/creating events

## Setup Steps

### 1. Get Fastmail App Password

1. Go to https://www.fastmail.com/settings/security/devicekeys
2. Create a new app password with these permissions:
   - Calendar (read/write)
3. Save the password (e.g., `your-app-password-here`)

### 2. Find Your CalDAV URL

Fastmail CalDAV URL format:
```
https://caldav.fastmail.com/dav/calendars/user/YOUR_EMAIL@fastmail.com/CALENDAR_NAME/
```

To find your calendar names:
1. Log into Fastmail web interface
2. Go to Calendar
3. Note the calendar name(s) you want to sync (e.g., "Default", "Personal", "Work")

### 3. Update Dockerfile

The Dockerfile.custom has been updated to include CalDAV tools (python3, pip, vdirsyncer, khal).

Rebuild the image:
```bash
docker build -f Dockerfile.custom -t nanoclaw-agent:custom .
```

### 4. Add Credentials to .env

Add to your `.env` file:
```bash
# Fastmail CalDAV (optional)
FASTMAIL_EMAIL=your_email@fastmail.com
FASTMAIL_APP_PASSWORD=your-app-password-here
```

### 5. Create vdirsyncer Config

Use the template in `config-examples/vdirsyncer.config` as a starting point.

The agent will need to:
1. Read this template
2. Replace FASTMAIL_EMAIL and FASTMAIL_APP_PASSWORD with actual values from env
3. Write to `~/.vdirsyncer/config`

### 6. Initial Sync

First time setup (agent should run):
```bash
# Discover calendars
vdirsyncer discover

# Initial sync
vdirsyncer sync
```

## Usage from Agents

### Sync Calendar
```bash
vdirsyncer sync
```

### Read Events (using khal)
```bash
# List today's events
khal list today

# List events for next 7 days
khal list now 7d

# Search for events
khal search "meeting"

# Show specific date
khal list 2026-03-10
```

### Create Event
```bash
khal new 2026-03-10 10:00 1h "Team Meeting" -d "Discuss project roadmap"
```

### Create Event (interactive)
```bash
khal interactive new
```

### Update Event
Need to:
1. Export event to .ics file
2. Edit the file
3. Re-import

Or use vdirsyncer + direct file manipulation:
```bash
# Events stored at ~/.calendar/fastmail/CALENDAR_NAME/*.ics
# Edit the .ics file directly, then sync
vdirsyncer sync
```

## Agent Helper Functions

You could create helper scripts for common operations:

**~/.local/bin/fastmail-create-event**
```bash
#!/bin/bash
khal new "$@"
vdirsyncer sync
```

**~/.local/bin/fastmail-list-events**
```bash
#!/bin/bash
vdirsyncer sync
khal list "$@"
```

## Automation Examples

### Daily Calendar Digest
Schedule a task that runs every morning:
```bash
vdirsyncer sync
khal list today 7d
```

### Event Reminders
Check for upcoming events and send reminders:
```bash
khal list today 2h | grep -v "No events"
```

### Create Event from Message
Parse user message and create calendar event:
```bash
# User says: "Schedule team meeting tomorrow at 2pm for 1 hour"
khal new tomorrow 14:00 1h "Team Meeting"
vdirsyncer sync
```

## Troubleshooting

### Check vdirsyncer status
```bash
vdirsyncer discover
```

### Verify CalDAV connection
```bash
curl -u "$FASTMAIL_EMAIL:$FASTMAIL_APP_PASSWORD" \
  "https://caldav.fastmail.com/dav/calendars/user/$FASTMAIL_EMAIL/"
```

### Manual sync
```bash
vdirsyncer sync -v DEBUG
```

## Security Notes

- App password stored in .env (never in git)
- Passed to containers via stdin (not mounted)
- Calendar data cached locally in container (ephemeral)
- Each container sync is fresh from Fastmail

## Next Steps

Once setup is complete, agents can:
- Read your calendar to check availability
- Create events based on conversations
- Set reminders for upcoming events
- Answer questions about your schedule
- Sync changes back to Fastmail automatically
