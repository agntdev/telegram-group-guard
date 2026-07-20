# Telegram Group Guard — Bot specification

**Archetype:** community

**Voice:** professional and concise — write every user-facing message, button label, error, and empty state in this voice.

A Telegram bot that automates group moderation by greeting new members, verifying newcomers with a single 'I'm human' button, detecting spam patterns, enforcing configurable thresholds, and providing admin commands and analytics on moderation actions.

> This is the complete contract for the bot. Implement EVERY entry point, flow, feature, integration, and edge case below. The completeness review checks the bot against this document after each build pass.

## Primary audience

- Group owners and admins
- Regular group members

## Success criteria

- Automated verification of new members with a single button
- Spam detection and automated moderation actions based on configurable thresholds
- Admin commands for manual moderation and settings changes
- Analytics on joins, verifications, and removals

## Entry points

Every feature must be reachable from the bot's command/button surface (button-first; only /start and /help are slash commands).

- **/start** (command, actor: user, command: /start) — Open the main menu
- **I'm human** (button, actor: user, callback: verify:human) — Verify new members by clicking the 'I'm human' button
  - inputs: user ID, verification timestamp
  - outputs: verification status, unrestricted access
- **/warn** (command, actor: admin, command: /warn) — Warn a user with a specified reason
  - inputs: user ID, reason
  - outputs: warning message, moderation log entry
- **/mute** (command, actor: admin, command: /mute) — Mute a user for a specified duration with a reason
  - inputs: user ID, duration, reason
  - outputs: mute message, moderation log entry
- **/kick** (command, actor: admin, command: /kick) — Kick a user with a specified reason
  - inputs: user ID, reason
  - outputs: kick message, moderation log entry
- **/ban** (command, actor: admin, command: /ban) — Ban a user for a specified duration with a reason
  - inputs: user ID, duration, reason
  - outputs: ban message, moderation log entry
- **/trust** (command, actor: admin, command: /trust) — Mark a user as trusted (exempt from automated checks)
  - inputs: user ID
  - outputs: trust status update, moderation log entry
- **/untrust** (command, actor: admin, command: /untrust) — Remove a user from the trusted list
  - inputs: user ID
  - outputs: untrust status update, moderation log entry
- **/set_welcome** (command, actor: admin, command: /set_welcome) — Edit the welcome message and rules
  - inputs: welcome message text, rules text
  - outputs: updated welcome message, moderation log entry
- **/set_auto_actions** (command, actor: admin, command: /set_auto_actions) — Configure which automatic actions are active and their thresholds
  - inputs: action type, thresholds
  - outputs: updated auto-action settings, moderation log entry
- **/show_log** (command, actor: admin, command: /show_log) — Show recent moderation events
  - inputs: number of entries to show
  - outputs: moderation log entries
- **/stats** (command, actor: admin, command: /stats) — Overview of joins, verifications, and removals over time
  - inputs: time period
  - outputs: analytics summary

## Flows

### New Member Verification
_Trigger:_ New member joins group

1. Post welcome message with 'I'm human' button
2. Restrict new member from sending messages
3. Wait for verification button click or timeout
4. If verified, lift restrictions and log verification
5. If not verified, remove member and log removal

_Data touched:_ Member, Verification challenge, Moderation event log

### Spam Detection and Moderation
_Trigger:_ User sends message

1. Check for spam signals (links from new accounts, repeated messages, rapid flooding)
2. If spam threshold reached, apply configured action (warn, mute, kick, ban)
3. Log moderation action with reason and rule triggered
4. Send public explanation for action

_Data touched:_ Member, Moderation event log

### Admin Command Execution
_Trigger:_ /warn, /mute, /kick, /ban, /trust, /untrust, /set_welcome, /set_auto_actions, /show_log, /stats

1. Parse command and parameters
2. Validate admin privileges
3. Execute command action
4. Log moderation event
5. Send confirmation response

_Data touched:_ Group, Member, Moderation event log

## Data entities

Durable data (must survive a restart) uses the toolkit's persistent store, never in-memory maps.

- **Group** _(retention: persistent)_ — Bot settings and runtime state per group
  - fields: welcome message, rules, verification timeout, spam thresholds, auto-action settings, trusted users list
- **Member** _(retention: session)_ — User profile plus trust/exempt status and moderation history within the group
  - fields: user ID, trust status, moderation history, recent message hashes/timestamps
- **Verification challenge** _(retention: session)_ — Pending verification instances for new members
  - fields: user ID, timestamp, verification status
- **Moderation event log** _(retention: persistent)_ — Short chronological records of automated or admin actions
  - fields: timestamp, user ID, action, reason, initiator (automated/admin)
- **Admin command actions** _(retention: session)_ — Manual moderation requests and settings changes
  - fields: command type, parameters, execution status

## Integrations

- **Telegram** (required) — Bot API messaging
Call external APIs against their real contract (correct endpoints, ids, params); credentials from env. Do not fake responses.

## Owner controls

- Configure verification timeout
- Set spam thresholds
- Enable/disable auto-actions
- Edit welcome message and rules
- Mark users as trusted
- View moderation log
- View analytics

## Notifications

- Direct messages to admins for automatic removals and threshold changes (configurable)

## Permissions & privacy

- Restrict new members from sending messages until verified
- Log moderation events with reasons
- Store minimal member state for spam detection with short TTL

## Edge cases

- New member verification timeout
- Spam detection false positives
- Admin command execution errors
- Moderation log retention limits

## Required tests

- Verify new member verification flow with button click and timeout
- Test spam detection with various spam patterns
- Validate admin command execution and logging
- Check moderation log retention and display
- Test analytics reporting

## Assumptions

- Default verification timeout is 3 minutes
- Default automated actions are warn → mute → kick
- Default mute duration is 10 minutes
- Default account age threshold for link blocking is 7 days
- Default flood detection window is 30 seconds with 5 messages threshold
- Log retention is 90 days per group
- Admin notifications are off by default
