# TSWoW/TrinityCore Achievement System Documentation

Analysis of the WOTLK achievement system and TrinityCore server, covering database tables, DBC files, and their relationships.

## Overview

The achievement system in WOTLK is split between:
- **Client-side data** (DBC files): Achievement definitions, categories, criteria
- **Server-side data** (SQL tables): Rewards, criteria conditions, character progress

Achievements track player accomplishments through criteria, which are conditions
that must be met to complete an achievement. When all criteria are satisfied,
the achievement is marked as completed and rewards may be granted.

## Client-Side Data (DBC Files)

DBC (Data Base Client) files are available to the WoTLK client. These contain
the core achievement definitions that the client needs to display achievements in the UI.
TSWoW provides DBC wrappers in [`dbc/Achievement.ts`](https://github.com/tswow/tswow/tree/master/tswow-scripts/wotlk/dbc/Achievement.ts), [`dbc/Achievement_Criteria.ts`](https://github.com/tswow/tswow/tree/master/tswow-scripts/wotlk/dbc/Achievement_Criteria.ts), and [`dbc/Achievement_Category.ts`](https://github.com/tswow/tswow/tree/master/tswow-scripts/wotlk/dbc/Achievement_Category.ts).

### Achievement.dbc
- **ID**: Unique achievement identifier
- **Title**: Localized achievement name (16 languages)
- **Description**: Localized achievement description (16 languages)
- **Category**: Reference to Achievement_Category.dbc
- **Points**: Achievement points awarded
- **IconID**: Spell icon ID for display
- **Flags**: Achievement behavior flags (hidden, counter, realm-first, etc.)
- **Faction**: -1 (both), 0 (Horde), 1 (Alliance)
- **Map**: Associated map ID (-1 if none)
- **Previous**: Parent achievement ID (for achievement chains)
- **Minimum_Criteria**: Minimum number of criteria that must be completed
- **Shares_Criteria**: Reference to another achievement whose criteria count

### Achievement_Criteria.dbc
- **ID**: Unique criteria identifier
- **Achievement_Id**: Reference to Achievement.dbc
- **Type**: Criteria type (kill creature, complete quest, etc.)
- **Asset_Id**: Primary asset (creature ID, quest ID, etc.)
- **Quantity**: Required quantity/amount
- **Start_Asset**: Starting condition asset
- **Start_Event**: Starting condition event
- **Fail_Asset**: Failure condition asset
- **Fail_Event**: Failure condition event
- **Flags**: Criteria flags (progress bar, hidden, etc.)
- **Timer_Asset_Id**: Timer asset reference
- **Timer_Start_Event**: Timer start event
- **Timer_Time**: Timer duration in seconds
- **Description**: Localized criteria description
- **Ui_Order**: Display order in UI

### Achievement_Category.dbc
- **ID**: Unique category identifier
- **Name**: Localized category name
- **Parent**: Parent category ID (-1 if top-level)
- **Ui_Order**: Display order in UI

These DBC files define what achievements exist and how they're structured, but
they don't contain server-side logic or character-specific progress data.

## Server-Side Database Tables (World Database)

The world database contains server-side configuration that supplements the DBC data.
The DBC structure definitions are in TrinityCore's [`DBCStructure.h`](https://github.com/tswow/TrinityCore/tree/c797f13b2c9f5a1a2fadef03eab660ffb675800d/src/server/shared/DataStores/DBCStructure.h).

### achievement_dbc
This table is used to override or extend DBC achievement data on the server.
It contains a subset of Achievement.dbc fields. See [achievement_dbc.ts](https://github.com/tswow/tswow/tree/master/tswow-scripts/wotlk/sql/achievement_dbc.ts) for the TSWoW wrapper:
- **ID**: Achievement ID (primary key, references Achievement.dbc)
- **requiredFaction**: Faction requirement override
- **mapID**: Map requirement override
- **points**: Points override
- **flags**: Flags override
- **count**: Minimum criteria count override
- **refAchievement**: Reference achievement override

**Note**: This table is typically used for server-specific modifications or
achievements that need to be hidden from the client but tracked server-side.

### achievement_reward
Defines rewards given when an achievement is completed. See [achievement_reward.ts](https://github.com/tswow/tswow/tree/master/tswow-scripts/wotlk/sql/achievement_reward.ts) and [AchievementReward.ts](https://github.com/tswow/tswow/tree/master/tswow-scripts/wotlk/std/Achievement/AchievementReward.ts) for TSWoW wrappers:
- **ID**: Achievement ID (primary key, references Achievement.dbc)
- **TitleA**: Alliance title reward (references CharTitles.dbc)
- **TitleH**: Horde title reward (references CharTitles.dbc)
- **ItemID**: Item reward (references ItemTemplate)
- **Sender**: Creature ID that sends the reward mail
- **Subject**: Mail subject (if not using template)
- **Body**: Mail body text (if not using template)
- **MailTemplateID**: Mail template ID (alternative to Subject/Body)

Rewards are granted automatically when an achievement is completed. Items are
sent via in-game mail from the specified sender creature.

### achievement_reward_locale
Localized versions of achievement reward mail text:
- **ID**: Achievement ID (primary key)
- **Locale**: Language code (primary key)
- **Subject**: Localized mail subject
- **Body**: Localized mail body

### achievement_criteria_data
Additional server-side conditions for criteria that can't be expressed in DBC. See [achievement_criteria_data.ts](https://github.com/tswow/tswow/tree/master/tswow-scripts/wotlk/sql/achievement_criteria_data.ts) for the TSWoW wrapper:
- **criteria_id**: Criteria ID (primary key, references Achievement_Criteria.dbc)
- **type**: Data type (primary key)
- **value1**: First value (varies by type)
- **value2**: Second value (varies by type)
- **ScriptName**: Custom script name for complex conditions

This table allows adding conditions like:
- Area/zone restrictions
- Creature/player class/race requirements
- Aura/buff requirements
- Level requirements
- Custom scripted conditions

Multiple rows per criteria_id are allowed (different types), allowing complex
combinations of conditions.

## Character-Specific Database Tables (Characters Database)

These tables track individual character progress and completed achievements.

### character_achievement
Stores completed achievements for each character:
- **guid**: Character GUID (primary key)
- **achievement**: Achievement ID (primary key, references Achievement.dbc)
- **date**: Unix timestamp when achievement was completed

When a character completes an achievement, a row is inserted here. This table
is loaded when the character logs in via [`LoadFromDB()`](https://github.com/tswow/TrinityCore/tree/c797f13b2c9f5a1a2fadef03eab660ffb675800d/src/server/game/Achievements/AchievementMgr.cpp) and saved periodically or on logout via [`SaveToDB()`](https://github.com/tswow/TrinityCore/tree/c797f13b2c9f5a1a2fadef03eab660ffb675800d/src/server/game/Achievements/AchievementMgr.cpp).

### character_achievement_progress
Tracks progress toward criteria that haven't been completed yet:
- **guid**: Character GUID (primary key)
- **criteria**: Criteria ID (primary key, references Achievement_Criteria.dbc)
- **counter**: Current progress value
- **date**: Unix timestamp when progress was last updated

This table stores incremental progress (e.g., "Kill 100 orcs: 47/100").
Once a criteria is completed, it may remain here for tracking purposes, but
the achievement completion is recorded in character_achievement. Progress is
loaded and saved alongside completed achievements in the same methods.

**Note**: Progress is only saved for criteria that have been started (counter > 0).
Completed criteria with timers that have expired are automatically removed.

## How It All Works Together

The core achievement management logic is implemented in TrinityCore's [`AchievementMgr`](https://github.com/tswow/TrinityCore/tree/c797f13b2c9f5a1a2fadef03eab660ffb675800d/src/server/game/Achievements/AchievementMgr.cpp) class, which handles loading, tracking, and completing achievements.

### Achievement Flow

1. **Definition**: Achievements are defined in Achievement.dbc with their
   criteria in Achievement_Criteria.dbc. Categories organize them in the UI.
   TSWoW provides wrappers in [`Achievement.ts`](https://github.com/tswow/tswow/tree/master/tswow-scripts/wotlk/std/Achievement/Achievement.ts) and [`AchievementCriteria.ts`](https://github.com/tswow/tswow/tree/master/tswow-scripts/wotlk/std/Achievement/AchievementCriteria.ts).

2. **Server Configuration**: Server-side tables (achievement_reward,
   achievement_criteria_data) add rewards and additional conditions.

3. **Progress Tracking**: As players perform actions, the server calls
   [`UpdateAchievementCriteria()`](https://github.com/tswow/TrinityCore/tree/c797f13b2c9f5a1a2fadef03eab660ffb675800d/src/server/game/Achievements/AchievementMgr.cpp) which:
   - Checks if criteria conditions are met (DBC + achievement_criteria_data)
   - Updates character_achievement_progress
   - Checks if achievement completion conditions are satisfied

4. **Completion**: When all criteria are met, [`CompletedAchievement()`](https://github.com/tswow/TrinityCore/tree/c797f13b2c9f5a1a2fadef03eab660ffb675800d/src/server/game/Achievements/AchievementMgr.cpp) is called:
   - Achievement is marked complete in character_achievement
   - Rewards are granted (titles, items via mail)
   - Achievement points are added
   - Client is notified to show completion UI
   - Guild is notified (if applicable)

5. **Persistence**: On character save/logout:
   - Completed achievements are saved to character_achievement
   - Active criteria progress is saved to character_achievement_progress

### Criteria Types

There are many criteria types (see [AchievementCriteria.ts](https://github.com/tswow/tswow/tree/master/tswow-scripts/wotlk/std/Achievement/AchievementCriteria.ts)), including:
- **KILL_CREATURE**: Kill specific creatures
- **COMPLETE_QUEST**: Complete quests
- **REACH_LEVEL**: Reach a level
- **WIN_BG**: Win battlegrounds
- **COMPLETE_RAID**: Complete raids
- **EARN_ACHIEVEMENT_POINTS**: Earn total achievement points
- And many more...

Each criteria type uses different fields in Achievement_Criteria.dbc:
- Asset_Id typically holds the primary reference (creature ID, quest ID, etc.)
- Quantity holds the required amount
- Additional fields vary by type

### Achievement Flags

Achievements can have various flags (see [AchievementFlags enum](https://github.com/tswow/tswow/tree/master/tswow-scripts/wotlk/std/Achievement/Achievement.ts)):
- **Statistic**: Never completes, just tracks progress
- **Hidden**: Not shown in UI until completed
- **Counter**: Counts statistics without completion
- **Realm First**: Tracks realm-first completions
- **Guild**: Guild-wide achievement
- **Account Bound**: Shared across account characters
- And more...

## Client UI Components

The achievement UI is implemented in Lua/XML files. The core Blizzard UI files are typically located in the client's MPQ files, but custom modifications can be placed in your tswow-install folder's module datasets, like `tswow-install/modules/default/datasets/dataset/luaxml/Interface/AddOns/Blizzard_AchievementUI/`.

### Key UI Files

- **Blizzard_AchievementUI.xml**: Defines UI frame structure
- **Blizzard_AchievementUI.lua**: Implements UI logic
- **Localization.lua**: Localization helpers

These files handle displaying achievements, categories, progress bars, and completion status based on data received from the server.

### UI Structure

The achievement UI displays:
- **Categories**: Organized by Achievement_Category.dbc hierarchy
- **Achievement List**: Shows achievements in selected category
- **Achievement Details**: Shows criteria, progress bars, rewards
- **Summary**: Shows completion statistics per category

Categories are displayed as:
- Top-level categories (General, Quests, Exploration, PvP, Dungeons, etc.)
- Sub-categories nested under parents
- Progress bars showing completion percentage

Achievements are displayed with:
- Icon (from SpellIcon.dbc via IconID)
- Name and description (localized from DBC)
- Criteria list with checkmarks for completed items
- Progress bars for incremental criteria
- Completion date
- Reward information (if any)

### Data Flow to Client

When a player logs in:
1. Server sends all completed achievements (SMSG_ALL_ACHIEVEMENT_DATA)
2. Server sends all active criteria progress
3. Client displays this data using DBC definitions for names/descriptions

When achievements are updated:
1. Server sends incremental updates (SMSG_ACHIEVEMENT_EARNED, etc.)
2. Client updates UI accordingly

## Summary: Data Split

### Client-Side (DBC Files)
- Achievement definitions (names, descriptions, icons)
- Criteria definitions (types, descriptions)
- Category hierarchy
- Display information

### Server-Side (World Database)
- Achievement overrides (achievement_dbc)
- Rewards (achievement_reward, achievement_reward_locale)
- Additional criteria conditions (achievement_criteria_data)

### Character-Specific (Characters Database)
- Completed achievements (character_achievement)
- Criteria progress (character_achievement_progress)

This split allows:
- Client to display achievements without server communication
- Server to enforce conditions and grant rewards
- Character data to persist across sessions
- Server-specific customizations without client patches

