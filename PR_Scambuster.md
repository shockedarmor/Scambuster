# Guild Blacklist Support

## Dependency Notice

**This PR requires a companion PR in the [Scambuster-Spineshatter](https://github.com/shockedarmor/Scambuster-Spineshatter/tree/guild-wide-listing) repo to function end-to-end.**

This PR adds the framework-level infrastructure. The Spineshatter PR adds the data pipeline and the guild list itself. Both must be merged and deployed together.

---

## Overview

Adds guild-based blacklisting to the Scambuster framework. Previously Scambuster could only warn on individual players matched by GUID or name. This change adds a parallel system that fires a warning whenever a player is encountered who is a member of a blacklisted guild, regardless of whether that individual player is on any list.

Guild blacklisting is realm-scoped by design. A guild blacklisted on Spineshatter will never trigger on any other realm.

---

## What Changed

### `core.lua`

**New: `process_guild_data(l)`**
Called during `build_database` for every registered provider. Reads the `guild_data` field from the provider table and loads matching realm entries into `self.provider_guild_table` (in-memory only, rebuilt on every load). Realm scoping is enforced here: only entries whose realm key matches `self.realm_name` are loaded.

**New: `check_unit_guild(unit_token)`**
Checks a unit's guild against two sources:
- `self.provider_guild_table` — guilds distributed via addon updates by list maintainers
- `self.db.realm.guild_blacklist` — guilds added by the individual player via slash command

Provider entries take priority if the same guild name appears in both. Respects the same alert lockout period as player alerts to avoid spam. Skips gracefully if guild info is not yet cached by the client.

**New: `raise_guild_alert(unit_token, guild, entry)`**
Fires the chat message and sound alert for a guild hit. Uses the same `use_system_alert` and `use_alert_sound` settings as player alerts.

**Updated: `check_unit(unit_token, unit_guid, scan_context)`**
Calls `check_unit_guild(unit_token)` at the top of every scan, when a unit token is available. This means guild checks fire on mouseover, target, trade, and group scans automatically.

**Updated: `GROUP_ROSTER_UPDATE()`**
Added a second loop over unit tokens after the existing GUID-based scan loop, so group members are also checked against the guild blacklist.

**New: `slashcommand_guild(input)`**
Registered as `/sbguild`. Manages the player's personal guild blacklist at runtime. See slash command reference below.

**Updated: `validate_provider()`**
Added `guild_data` to the recognised provider fields so it does not generate spurious warnings during list import.

**Updated: `build_database()`**
Resets `self.provider_guild_table = {}` on each rebuild and calls `process_guild_data` for every provider after processing player cases.

### `config.lua`

**Updated: `defaults.realm`**
Added `guild_blacklist = {}` to the realm-scoped defaults. This is where per-player manually added guilds are persisted. Using `db.realm` ensures entries are automatically scoped to the realm the player is logged into with no extra key manipulation required.

**Updated: `defaults.profile`**
Added `guild_blacklist_enabled = true`. Enables guild blacklisting by default. Stored per-profile so different characters can have different preferences.

**Updated: `SB.options`**
Added a Guild Blacklist tab to the Scambuster AceConfig UI (`/sb`). Contains an enable/disable toggle and a slash command reference card.

---

## Two Sources, One Check

The guild blacklist operates from two independent sources that are checked together at scan time:

| Source | How it gets there | Scope | Persists |
|---|---|---|---|
| Provider guild list | Addon maintainer edits `list.lua`, pushes update | All users on next update | No — rebuilt in memory each load |
| Player personal list | `/sbguild add` slash command | That player's client only | Yes — stored in SavedVariables |

Because provider entries are in-memory only, removing a guild from `list.lua` takes effect for all users immediately on the next addon update with no stale data left in anyone's SavedVariables.

---

## Slash Command Reference

```
/sbguild add <GuildName> | <Reason>    Add a guild to your personal blacklist
/sbguild remove <GuildName>            Remove a guild from your personal blacklist
/sbguild list                          Show all blacklisted guilds (both sources, labelled)
/sbguild on                            Enable guild blacklisting
/sbguild off                           Disable guild blacklisting
/sbguild                               Show help and current status
```

Examples:
```
/sbguild add Blablabla | Mass scam, ninja looted SR run
/sbguild add Sketchy Guild
/sbguild remove Blablabla
/sbguild list
```

If no reason is provided the entry is stored with "No reason specified". Personal entries are visible in `/sbguild list` labelled as `user-added`. Provider entries are labelled with the provider name.

---

## Notes

- Guild name matching is case and space sensitive. Always use the exact in-game capitalisation.
- `GetGuildInfo()` depends on cached client data. On very first mouseover before the client has received guild info for a player there may be a miss. A second mouseover will catch it once the data is available. This is a WoW API limitation.
- Guild alerts respect the same alert lockout timer as player alerts (default 15 minutes) to avoid repeated warnings for the same guild.
