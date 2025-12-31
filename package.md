# Package Client System in TSWoW

This document explains how the TSWoW package client system works.

## Table of Contents
- [Overview](#overview)
- [Key Concepts](#key-concepts)
- [Configuration](#configuration)
- [Module Traversal Logic](#module-traversal-logic)
- [What Gets Packaged](#what-gets-packaged)
- [Module Naming Conventions](#module-naming-conventions)
- [Build Commands](#build-commands)

---

## Overview

The package client system bundles your TSWoW modules into MPQ archive files that the WoW client can read. The system:

1. Scans your module structure for **endpoints** (folders like `assets/`, `datascripts/`, etc.)
2. Maps modules to MPQ files based on `Package.Mapping` configuration
3. Bundles assets, DBC files, and LuaXML files into their assigned MPQs
4. Creates the MPQ files in `bin/package/`

## Key Concepts

- **Module Endpoint**: A directory containing one of the special folders (`assets/`, `datascripts/`, `livescripts/`, `addon/`, `shared/`, `realms/`, `datasets/`)
- **Package.Mapping**: Configuration in `dataset.conf` that controls which modules go into which MPQ files
- **Special Modules**: `dbc`, `luaxml`, and `_build` are built-in modules that must be mapped

---

## Configuration

### Location
`modules/<module-name>/datasets/<dataset-name>/dataset.conf`

### Understanding Module Structure

Before configuring `Package.Mapping`, you need to understand what module endpoints exist.

#### Example: Default Module with Submodules

```
modules/default/
├── assets/              → Endpoint: "default"
│   └── Textures/
├── datascripts/         → Endpoint: "default" (same endpoint)
├── core/
│   └── datascripts/     → Endpoint: "default.core"
└── maps/
    ├── elwynn/
    │   └── assets/      → Endpoint: "default.maps.elwynn"
    └── durotar/
        └── assets/      → Endpoint: "default.maps.durotar"
```

**Module endpoints discovered:**
- `default` - Root level with assets/ and datascripts/
- `default.core` - Subdirectory with datascripts/
- `default.maps.elwynn` - Map assets
- `default.maps.durotar` - Map assets

### Configuration Examples

#### Example 1: Everything Together (Simplest)

```conf
Package.Mapping = ["All.MPQ:*"]
```

**What happens:**
- `All.MPQ` gets: `default`, `default.core`, `default.maps.elwynn`, `default.maps.durotar`, `dbc`, `luaxml`, `_build`

---

#### Example 2: Separate Database from Assets

```conf
Package.Mapping = [
    "Data.MPQ:dbc,luaxml,_build",
    "Assets.MPQ:*"
]
```

**What happens:**
- `Data.MPQ` gets: `dbc`, `luaxml`, `_build` (database and UI files)
- `Assets.MPQ` gets: `default`, `default.core`, `default.maps.elwynn`, `default.maps.durotar` (all asset files)

---

#### Example 3: Explicit vs Parent Matching

**Explicit mapping** - specify each module individually:
```conf
Package.Mapping = [
    "Core.MPQ:default.core",
    "Main.MPQ:default",
    "Data.MPQ:dbc,luaxml,_build"
]
```

**What happens:**
- `Core.MPQ` gets: `default.core` only
- `Main.MPQ` gets: `default` only (root level assets/datascripts)
- `Data.MPQ` gets: `dbc`, `luaxml`, `_build`
- ⚠️ **Problem**: `default.maps.elwynn` and `default.maps.durotar` have no mapping! They won't be packaged!

**Fix with wildcard:**
```conf
Package.Mapping = [
    "Core.MPQ:default.core",
    "Main.MPQ:default",
    "Maps.MPQ:*"  # Catches everything else including map submodules
]
```

**Fix with parent matching:**
```conf
Package.Mapping = [
    "Maps.MPQ:default.maps",      # Matches ALL map submodules (parent match!)
    "Core.MPQ:default.core",
    "Main.MPQ:default",
    "Data.MPQ:dbc,luaxml,_build"
]
```

**What happens:**
- `Maps.MPQ` gets: `default.maps.elwynn` + `default.maps.durotar` (matched by parent `default.maps`)
- `Core.MPQ` gets: `default.core`
- `Main.MPQ` gets: `default`
- `Data.MPQ` gets: `dbc`, `luaxml`, `_build`

**Key insight:** Specifying `default.maps` matches any module starting with `default.maps.` - useful for grouping related submodules!

---

#### Example 4: Per-Map MPQs

```conf
Package.Mapping = [
    "Elwynn.MPQ:default.maps.elwynn",
    "Durotar.MPQ:default.maps.durotar",
    "Main.MPQ:*"
]
```

**What happens:**
- `Elwynn.MPQ` gets: `default.maps.elwynn` only
- `Durotar.MPQ` gets: `default.maps.durotar` only
- `Main.MPQ` gets: `default`, `default.core`, `dbc`, `luaxml`, `_build`

---

### Required Mappings

You **must** include mappings for these special modules (or use `*` to catch them):
- **`dbc`** - Database Client Files (spell data, item data, etc.)
- **`luaxml`** - UI and Lua interface files
- **`_build`** - Build artifacts and compiled assets

If these are not mapped, you'll see errors like:
```
[dataset] Module dbc has no package mapping in dataset <name>, will not build it
```

---

## Module Traversal Logic

### How TSWoW Discovers Module Endpoints

Algorithm (from `tswow-scripts/util/Paths.ts`):

1. **Starts at module root** and recursively scans subdirectories
2. **Stops recursing** when it finds an endpoint folder (assets, datascripts, etc.)
3. **Does NOT look inside** endpoint folders for more endpoints
4. **Creates submodules** for any directory containing endpoint folders

### Example: Traversal in Action

```
modules/my-module/
├── assets/                    ← Found! Stop recursing. Endpoint: my-module
│   └── maps/
│       └── zone1/
│           └── assets/        ← NEVER FOUND (inside parent assets/)
├── subfolder/
│   └── assets/                ← Found! Endpoint: my-module.subfolder
└── deep/
    └── nested/
        └── folder/
            └── datascripts/   ← Found! Endpoint: my-module.deep.nested.folder
```

**Result**: Three module endpoints created:
- `my-module`
- `my-module.subfolder`
- `my-module.deep.nested.folder`

---

## What Gets Packaged

### Assets from Module Endpoints

The packaging system scans each module's `assets/` folder and includes **most** file types.

#### File Type Filtering (from `Package.ts` lines 99-105)

**Excluded file types** (NOT packaged):
- `.png` - Source images
- `.blend` - Blender files
- `.psd` - Photoshop files
- `.json` - JSON data files
- `.dbc` - DBC files in assets folders

**Included file types** (packaged):
- `.blp` - Blizzard Picture format (textures)
- `.m2` - Models
- `.wmo` - World Map Objects
- `.adt` - Map terrain data
- `.wdt`, `.wdl` - Map metadata
- `.skin` - Model skins
- `.anim` - Animations
- `.mp3`, `.wav` - Audio files
- Most other WoW asset formats

### DBC Files

DBC files are packaged separately from the `datasets/<dataset>/dbc/` directory:
- Only **modified** DBC files are included (by default)
- Use `--fullDBC` flag to include all DBC files
- Packaged into whichever MPQ has `dbc` mapped

### LuaXML Files

UI and Lua files from `datasets/<dataset>/luaxml/` directory:
- Only **modified** files are included (by default)
- Use `--fullInterface` flag to include all interface files
- Packaged into whichever MPQ has `luaxml` mapped

### Build Artifacts

The `_build` module contains compiled/generated assets from the build process.

---

## Module Naming Conventions

Module names follow the **directory path** with dots (`.`) separating levels. For example:
- `modules/my-mod/core/assets/` → module name `my-mod.core`
- `modules/my-mod/maps/zone1/assets/` → module name `my-mod.maps.zone1`

**Parent Matching:** `Package.Mapping` uses parent matching, so `my-mod.maps` matches `my-mod.maps.zone1`, `my-mod.maps.zone2`, `my-mod.maps.deep.nested.zone`, etc.

---

## Build Commands

```bash
package client
package client my-dataset
package client --fullDBC
package client --fullInterface
```
