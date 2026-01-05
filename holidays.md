# TSWoW/TrinityCore Holiday System Documentation

Analysis of the WOTLK holiday system and TrinityCore server, covering database tables, DBC files, client-side interface, and their relationships.

## Overview

The holiday system in WOTLK is split between:
- **Client-side data** (DBC files): Holiday definitions, names, descriptions, and calendar display data
- **Server-side data** (SQL tables): Date overrides, durations, and game event integration
- **Client-side interface** (LuaXML): Calendar UI that displays holidays to players

Holidays are special calendar events that appear in the in-game calendar. They can be yearly (recurring annually), weekly (recurring weekly), or use defined dates. Each holiday can have multiple stages with different durations, and is linked to game events that control server-side behavior.

## Client-Side Data (DBC Files)

DBC (Data Base Client) files are available to the WoTLK client. These contain the core holiday definitions that the client needs to display holidays in the calendar UI. TSWoW provides DBC wrappers in [`dbc/Holidays.ts`](https://github.com/tswow/tswow/tree/master/tswow-scripts/wotlk/dbc/Holidays.ts), [`dbc/HolidayNames.ts`](https://github.com/tswow/tswow/tree/master/tswow-scripts/wotlk/dbc/HolidayNames.ts), and [`dbc/HolidayDescriptions.ts`](https://github.com/tswow/tswow/tswow-scripts/wotlk/dbc/HolidayDescriptions.ts).

### Holidays.dbc

The main holiday definition file. Each entry defines a holiday's core properties:

- **ID**: Unique holiday identifier (primary key)
- **Duration**: Array of 10 duration values (in hours) for different holiday stages
- **Date**: Array of 26 date values (packed date format) defining when the holiday occurs
- **Region**: WoW region identifier
- **Looping**: Whether the holiday loops continuously (used for events like Call to Arms)
- **CalendarFlags**: Array of 10 calendar filter flags
- **HolidayNameID**: Reference to HolidayNames.dbc entry
- **HolidayDescriptionID**: Reference to HolidayDescriptions.dbc entry
- **TextureFilename**: Path to the holiday's calendar texture
- **Priority**: Display priority (higher values appear first)
- **CalendarFilterType**: Holiday type (-1 = Yearly, 0 = Weekly, 1 = Defined Dates, 2 = Custom Period)
- **Flags**: Additional holiday flags

The date values are packed into 32-bit integers with the following bit layout:
- Bits 0-5: Minutes (0-63)
- Bits 6-10: Hours (0-31)
- Bits 11-13: Day of week (0-7, for weekly holidays)
- Bits 14-19: Day of month (0-63, stored as day-1)
- Bits 20-23: Month (0-15, stored as month-1)
- Bits 24-28: Year offset (for yearly holidays, 31 = recurring yearly)
- Bits 29-30: Timezone

### HolidayNames.dbc

Contains localized holiday names for all supported languages:

- **ID**: Unique identifier (primary key)
- **Name**: Localized name array (17 languages: enUS, koKR, frFR, deDE, zhCN, zhTW, esES, esMX, ruRU, ptPT, itIT, ptBR, jaJP, plPL, csCZ, thTH, arAR)

Each holiday references a HolidayNames.dbc entry via its `HolidayNameID` field.

### HolidayDescriptions.dbc

Contains localized holiday descriptions for all supported languages:

- **ID**: Unique identifier (primary key)
- **Description**: Localized description array (17 languages)

Each holiday references a HolidayDescriptions.dbc entry via its `HolidayDescriptionID` field.

These DBC files define what holidays exist and how they're displayed in the calendar, but they don't contain server-side timing logic or date overrides.

## Server-Side Database Tables (World Database)

The world database contains server-side configuration that supplements and overrides the DBC data. The DBC structure definitions are in TrinityCore's [`DBCStructure.h`](https://github.com/tswow/TrinityCore/blob/master/src/server/shared/DataStores/DBCStructure.h).

### holiday_dates

This table allows server-side override of holiday dates and durations. See [`sql/holiday_dates.ts`](https://github.com/tswow/tswow/tree/master/tswow-scripts/wotlk/sql/holiday_dates.ts) for the TSWoW wrapper:

- **id**: Holiday ID (primary key, references Holidays.dbc)
- **date_id**: Date index (primary key, 0-25, corresponds to Date array index in DBC)
- **date_value**: Packed date value override (overrides DBC Date[date_id])
- **holiday_duration**: Duration override in hours (overrides DBC Duration[date_id])

**Key Relationship**: The `id` field references `Holidays.dbc.ID`, and `date_id` corresponds to the index in the `Date` array. When the server loads holidays, it reads from this table to override DBC values, allowing server-specific date adjustments without modifying client DBC files.

**Usage**: This table is used when you need to:
- Override specific holiday dates for your server
- Adjust holiday durations
- Add custom date entries beyond the 26 DBC date slots

The TrinityCore server loads this data in `GameEventMgr::LoadHolidayDates()` and applies it to the in-memory DBC structure.

### game_event

The main game event table that links holidays to server-side events. See [`sql/game_event.ts`](https://github.com/tswow/tswow/tree/master/tswow-scripts/wotlk/sql/game_event.ts) for the TSWoW wrapper:

- **eventEntry**: Unique game event ID (primary key)
- **start_time**: Absolute start timestamp (can be NULL for holiday-driven events)
- **end_time**: Absolute end timestamp (can be NULL for holiday-driven events)
- **occurence**: Delay in minutes between event occurrences
- **length**: Event duration in minutes
- **holiday**: Holiday ID (references Holidays.dbc.ID, 0 = not a holiday event)
- **holidayStage**: Holiday stage index (0-based, corresponds to date_id)
- **description**: Event description for server logs
- **world_event**: Event type (0 = normal, 1 = world event, 5 = internal)
- **announce**: Announcement flag (0 = don't announce, 1 = announce, 2 = use config)

**Key Relationship**: When `holiday` is non-zero, the event is a holiday event. The server automatically calculates `start_time`, `length`, and `occurence` from the holiday's DBC data and `holiday_dates` table entries. The `holidayStage` field indicates which stage of a multi-stage holiday this event represents.

**Holiday Event Behavior**: 
- For holiday events, you should NOT manually set `start_time`, `end_time`, `occurence`, or `length` - these are calculated automatically from the holiday definition
- The server's `GameEventMgr::SetHolidayEventTime()` function handles this calculation
- Holiday events with ID >= 500 are ignored by the server (custom holiday IDs)

### game_event_* Tables

Holiday game events can be linked to various game systems through additional `game_event_*` tables. These tables control what happens in the game world when a holiday event is active:

- **`game_event_creature`**: NPCs that spawn during the holiday event
- **`game_event_creature_quest`**: Quests offered by NPCs during the holiday
- **`game_event_gameobject`**: GameObjects that spawn or despawn during the holiday (negative `eventEntry` removes objects)
- **`game_event_gameobject_quest`**: Quests available from GameObjects during the holiday
- **`game_event_npc_vendor`**: Vendor items available from NPCs during the holiday (negative `eventEntry` removes items)
- **`game_event_npcflag`**: NPC flag changes during the holiday (e.g., making vendors available)
- **`game_event_model_equip`**: NPC model/equipment changes during the holiday (negative `eventEntry` removes changes)
- **`game_event_pool`**: Pool templates (spawn groups) linked to the holiday
- **`game_event_battleground_holiday`**: Battleground holidays (links events to specific battlegrounds)
- **`game_event_condition`**: Conditions that must be met for the event to activate
- **`game_event_quest_condition`**: Quest conditions tied to the holiday event
- **`game_event_seasonal_questrelation`**: Seasonal quest relations for the holiday
- **`game_event_prerequisite`**: Event prerequisites (events that must be active first)
- **`game_event_arena_seasons`**: Arena seasons tied to the holiday event

**Key Points**:
- All these tables reference `game_event.eventEntry` (not the holiday ID directly)
- When a holiday event is active, the server checks these tables to determine what NPCs spawn, what quests are available, etc.
- Negative `eventEntry` values in some tables (like `game_event_gameobject`, `game_event_npc_vendor`) remove things during the event rather than adding them
- These tables allow holidays to have complex server-side behaviors beyond just calendar display

**Example**: A Winter Veil holiday might:
- Spawn special NPCs (`game_event_creature`)
- Add holiday vendor items (`game_event_npc_vendor`)
- Offer holiday quests (`game_event_creature_quest`)
- Change NPC appearances (`game_event_model_equip`)
- Spawn holiday decorations (`game_event_gameobject`)

## Holiday Types and Stages

Holidays support three main types, each with different scheduling behaviors:

### Yearly Holidays (CalendarFilterType = -1)

Recurring annually on specific dates. Examples: Christmas, New Year's Eve.

- Uses month/day/hour/minute from the Date array
- Can have multiple stages (different dates throughout the year)
- Each stage can have different durations
- Automatically repeats every year

### Weekly Holidays (CalendarFilterType = 0)

Recurring weekly on a specific day of the week. Examples: Weekly fishing tournaments.

- Uses day of week/hour/minute from the Date array
- Can have multiple stages
- Automatically repeats every week
- Occurrence is set to 10080 minutes (1 week)

### Defined Dates Holidays (CalendarFilterType = 1)

Uses specific dates defined in the Date array without automatic recurrence. Examples: Darkmoon Faire.

- Uses month/day/hour/minute from the Date array
- Dates are explicitly defined and don't auto-repeat
- Useful for events that occur on irregular schedules

### Custom Period Holidays (CalendarFilterType = 2)

Used for looping events with custom periods. Examples: Call to Arms.

- Uses a period-based system
- Less commonly used, primarily for PvP events
- Not fully supported in TSWoW's high-level API yet

## Holiday Stages

A single holiday can have multiple stages, each with its own:
- Start date/time
- Duration
- Associated game event

Stages are indexed from 0-25 (matching the Date and Duration arrays). Each stage can have a corresponding entry in `game_event` with `holidayStage` set to the stage index.

**Example**: A holiday might have:
- Stage 0: Preparation phase (starts Dec 20, lasts 2 hours)
- Stage 1: Main event (starts Dec 21, lasts 24 hours)
- Stage 2: Cleanup phase (starts Dec 22, lasts 2 hours)

## Client-Side Interface (LuaXML)

The holiday calendar interface is implemented in the Blizzard Calendar addon, located in:
- `Interface/AddOns/Blizzard_Calendar/Blizzard_Calendar.lua`
- `Interface/AddOns/Blizzard_Calendar/Blizzard_Calendar.xml`
- `Interface/AddOns/Blizzard_Calendar/Blizzard_CalendarTemplates.xml`
- `Interface/AddOns/Blizzard_Calendar/Localization.lua`

### Calendar Display

The calendar UI displays holidays using:
- **Holiday Name**: Retrieved from `HolidayNames.dbc` via the client's `CalendarGetHolidayInfo()` function
- **Holiday Description**: Retrieved from `HolidayDescriptions.dbc`
- **Holiday Texture**: Uses the `TextureFilename` from `Holidays.dbc`, displayed in the calendar day buttons
- **Holiday View Frame**: `CalendarViewHolidayFrame` displays detailed holiday information when clicked

### Calendar Functions

Key client-side functions for holidays:
- `CalendarGetHolidayInfo(eventIndex)`: Returns name, description, and texture for a holiday
- `CalendarGetDayEvent(monthOffset, day, eventIndex)`: Gets holiday event information for a specific day
- Holiday events are filtered by `CalendarFilterType` to show in appropriate calendar views

The calendar supports filtering holidays by type (weekly vs yearly) and displays them with appropriate textures and overlays.

## Data Flow and Relationships

```
┌─────────────────┐
│  Holidays.dbc   │──┐
│  (Client DBC)   │  │
└─────────────────┘  │
                     │ References
┌─────────────────┐  │
│ HolidayNames    │◄─┘
│ .dbc            │
└─────────────────┘
        │
        │ References
        ▼
┌─────────────────┐
│ HolidayDescript │
│ ions.dbc        │
└─────────────────┘

┌─────────────────┐
│ holiday_dates   │──┐
│ (SQL Table)     │  │ Overrides
└─────────────────┘  │
        │            │
        │ References│
        ▼            │
┌─────────────────┐ │
│  Holidays.dbc   │◄┘
│  (Server Load)  │
└─────────────────┘
        │
        │ Used by
        ▼
┌─────────────────┐
│  game_event     │
│  (SQL Table)    │
└─────────────────┘
        │
        │ Referenced by
        ▼
┌─────────────────────────────────────────┐
│  game_event_* Tables                     │
│  (Creatures, GameObjects, Quests,       │
│   Vendors, NPC Flags, Conditions, etc.) │
└─────────────────────────────────────────┘
        │
        │ Controls
        ▼
┌─────────────────┐
│ Server Events   │
│ (NPCs, Quests,  │
│  Scripts, etc.) │
└─────────────────┘
```

### Loading Process

1. **Client**: Loads `Holidays.dbc`, `HolidayNames.dbc`, `HolidayDescriptions.dbc` at startup
2. **Server**: 
   - Loads `Holidays.dbc` into memory
   - Loads `holiday_dates` table and overrides DBC values
   - Loads `game_event` table and all `game_event_*` tables
   - For each game event with `holiday != 0`:
     - Looks up the holiday in DBC
     - Calculates start time from holiday Date[holidayStage]
     - Sets duration from holiday Duration[holidayStage]
     - Sets occurrence based on CalendarFilterType
   - Activates events based on calculated times
   - When a holiday event becomes active, the server checks all `game_event_*` tables to:
     - Spawn/despawn NPCs and GameObjects
     - Enable/disable quests
     - Add/remove vendor items
     - Modify NPC flags and appearances
     - Apply conditions and prerequisites

### TSWoW API Usage

TSWoW provides high-level APIs for creating and managing holidays:

```typescript
import { HolidayRegistry } from "tswow-stdlib/wotlk/std/GameEvent/Holiday";

// Create a yearly holiday
const holiday = HolidayRegistry.create('mymod', 'myholiday')
    .Name.set({enGB: 'My Holiday'})
    .Description.set({enGB: 'A special holiday event'})
    .Type.YEARLY.set()
    .Texture.set('Interface\\Calendar\\Holidays\\Calendar_DefaultHoliday')
    .Priority.set(10);

// Add a stage (date and duration)
holiday.Stages.add(
    Month.DECEMBER,  // Month
    25,               // Day of month
    0,                // Hour
    0,                // Minute
    24,               // Duration
    'HOURS'          // Duration unit
);

// Link to a game event
holiday.Stages.get(0).GameEvents.addMod('mymod', 'myevent', (event) => {
    // Configure the game event
    // Note: start_time, duration, occurrence are auto-set from holiday
});
```

## Limitations and Notes

- Holiday IDs >= 500 are ignored by TrinityCore's holiday timing system (custom IDs)
- Custom Period holidays (type 2) are not fully supported in TSWoW's high-level API
- Hourly holidays don't appear in the calendar UI (use regular game events instead)
- The `holiday_dates` table can extend beyond the 26 DBC date slots, but only the first 26 are used for initial DBC loading
- Holiday textures must be in the client's MPQ files or custom patches
- Localization requires entries in all 17 supported languages for full client support

