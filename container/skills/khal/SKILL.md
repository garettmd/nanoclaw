---
name: khal
description: Manage calendars from the command line — view events, create appointments, check availability, and sync with CalDAV servers like Fastmail. Use whenever you need to interact with calendars, not just when explicitly asked.
allowed-tools: Bash(khal:*,vdirsyncer:*)
---

# Calendar Management with khal

## Prerequisites

This skill requires Fastmail CalDAV integration to be configured. See `docs/fastmail-caldav.md` for setup instructions.

## Quick start

```bash
vdirsyncer sync              # Sync calendars first
khal list today              # Show today's events
khal list today 7d           # Next 7 days
khal new tomorrow 14:00 1h "Team Meeting"  # Create event
vdirsyncer sync              # Sync changes back
```

## Core workflow

1. **Sync down**: `vdirsyncer sync` (get latest from Fastmail)
2. **Read/Create**: Use khal commands
3. **Sync up**: `vdirsyncer sync` (push changes to Fastmail)

## Setup helper

Before first use, configure vdirsyncer with credentials:

```bash
# Create config from template
mkdir -p ~/.vdirsyncer
sed -e "s/FASTMAIL_EMAIL/$FASTMAIL_EMAIL/g" \
    -e "s/FASTMAIL_APP_PASSWORD/$FASTMAIL_APP_PASSWORD/g" \
    /workspace/project/config-examples/vdirsyncer.config > ~/.vdirsyncer/config

# Discover and sync calendars
vdirsyncer discover
vdirsyncer sync
```

## Commands

### Viewing events

```bash
# Today's events
khal list today

# Specific date
khal list 2026-03-10

# Date range (next 7 days)
khal list today 7d

# Specific range
khal list 2026-03-10 2026-03-17

# Relative dates
khal list tomorrow
khal list now 3d

# Show as calendar grid
khal calendar
khal calendar 3  # Next 3 months
```

### Searching events

```bash
# Search by keyword
khal search "meeting"
khal search "dentist"

# Case-insensitive by default
khal search "TEAM"  # Finds "team", "Team", "TEAM"
```

### Creating events

```bash
# Basic: date, time, duration, title
khal new 2026-03-10 14:00 1h "Team Meeting"

# With description
khal new 2026-03-10 14:00 1h "Team Meeting" -d "Discuss Q2 roadmap"

# With location
khal new 2026-03-10 14:00 1h "Team Meeting" -l "Conference Room A"

# All-day event
khal new 2026-03-15 "Project Deadline"

# Multi-day event
khal new 2026-03-20 2026-03-22 "Conference"

# Relative dates
khal new tomorrow 09:00 30m "Standup"
khal new today 17:00 1h "Review session"

# With specific calendar (if multiple)
khal new 2026-03-10 14:00 1h "Team Meeting" -a Work
```

### Duration formats

```bash
1h        # 1 hour
30m       # 30 minutes
1h30m     # 1 hour 30 minutes
2h        # 2 hours
15m       # 15 minutes
```

### Interactive mode

```bash
# Interactive event creation (prompts for details)
khal interactive
```

### Calendar info

```bash
# List configured calendars
khal printcalendars

# Show calendar details
khal printcalendars -v
```

## Output parsing

### khal list output format

```
Today (03/04/2026):
09:00-10:00 Daily Standup
14:00-15:30 Team Meeting :: Conference Room A

Tomorrow (03/05/2026):
10:00-11:00 Client Call
```

### Parsing tips

```bash
# Check if there are any events
khal list today | grep -v "^$" | wc -l

# Extract event titles only
khal list today | grep -oP '^\d{2}:\d{2}.*?\K[A-Z].*'

# Check for specific event
khal list today | grep -i "meeting"

# Get events as simple list (no dates)
khal list today --format "{start-time}-{end-time} {title}"
```

### Format strings

```bash
# Custom output format
khal list --format "{title} at {start-time}" today

# Available placeholders:
# {title}, {start-time}, {end-time}, {start-date}, {end-date}
# {location}, {description}, {calendar}
```

## Common patterns

### Check availability

```bash
# See if time slot is free
vdirsyncer sync
if khal list 2026-03-10 | grep "14:00"; then
  echo "Busy at 2pm"
else
  echo "Free at 2pm"
fi
```

### Daily digest

```bash
# Morning summary
vdirsyncer sync
echo "Today's schedule:"
khal list today
echo ""
echo "Tomorrow:"
khal list tomorrow
```

### Create and confirm

```bash
# Create event and verify
vdirsyncer sync
khal new tomorrow 14:00 1h "Team Meeting" -d "Discuss roadmap"
vdirsyncer sync
khal list tomorrow
```

### Week ahead

```bash
vdirsyncer sync
khal list today 7d
```

### Find next occurrence

```bash
# Find next "standup" event
vdirsyncer sync
khal search "standup" | head -1
```

### Event reminders

```bash
# Check for events in next 2 hours
vdirsyncer sync
UPCOMING=$(khal list now 2h)
if [ -n "$UPCOMING" ]; then
  echo "Upcoming events: $UPCOMING"
fi
```

## Integration with NanoClaw workflows

### Schedule from chat

When user says: "Schedule team meeting tomorrow at 2pm for 1 hour"

```bash
vdirsyncer sync
khal new tomorrow 14:00 1h "Team Meeting"
vdirsyncer sync
khal list tomorrow  # Confirm
```

### Check before scheduling

```bash
# Check availability before creating
vdirsyncer sync
if khal list 2026-03-10 | grep -q "14:00"; then
  echo "Sorry, you have a conflict at 2pm"
else
  khal new 2026-03-10 14:00 1h "New Meeting"
  vdirsyncer sync
  echo "Meeting scheduled"
fi
```

### Daily briefing (scheduled task)

```bash
#!/bin/bash
vdirsyncer sync
echo "*Your schedule for today:*"
khal list today
echo ""
echo "*Tomorrow:*"
khal list tomorrow
```

### Event search for user

When user asks: "When is my dentist appointment?"

```bash
vdirsyncer sync
khal search "dentist"
```

## Troubleshooting

### Sync issues

```bash
# Verbose sync for debugging
vdirsyncer sync -v DEBUG

# Force full sync
rm -rf ~/.vdirsyncer/status/*
vdirsyncer sync

# Check vdirsyncer config
cat ~/.vdirsyncer/config

# Test CalDAV connection
curl -u "$FASTMAIL_EMAIL:$FASTMAIL_APP_PASSWORD" \
  "https://caldav.fastmail.com/dav/calendars/user/$FASTMAIL_EMAIL/"
```

### No events showing

```bash
# Verify calendars are discovered
vdirsyncer discover

# Check local calendar files
ls -la ~/.calendar/fastmail/

# Re-sync
vdirsyncer sync
```

### khal not finding events

```bash
# Check khal config
cat ~/.config/khal/config

# List configured calendars
khal printcalendars
```

## Advanced usage

### Updating events

khal doesn't have direct edit commands. To update:

```bash
# Option 1: Delete and recreate
# (Not ideal - use vdirsyncer + direct file edit instead)

# Option 2: Edit ICS file directly
vdirsyncer sync
# Find event file in ~/.calendar/fastmail/CALENDAR_NAME/*.ics
# Edit the .ics file
vdirsyncer sync  # Push changes
```

### Multiple calendars

```bash
# Create in specific calendar
khal new tomorrow 10:00 1h "Work Meeting" -a Work
khal new 2026-03-15 "Birthday" -a Personal

# View specific calendar
khal list -a Work today

# View all calendars
khal list today
```

### Export events

```bash
# Events are stored as .ics files
ls ~/.calendar/fastmail/*/

# Copy/backup calendar
cp -r ~/.calendar/fastmail ~/backup/
```

## Tips

- Always sync before reading (get latest changes)
- Always sync after creating (push changes to server)
- Use relative dates (today, tomorrow) for natural language
- Use `khal list today 7d` for week view (most common)
- Search is case-insensitive by default
- Duration must be specified for timed events
- All-day events don't need time/duration

## Security notes

- Credentials stored in .env (never in git)
- vdirsyncer config created at runtime from template
- Local calendar cache at ~/.calendar/fastmail/ (ephemeral in containers)
- Each container run syncs fresh from Fastmail
