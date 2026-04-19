# GlowESP — How It Works

## Overview

GlowESP in CS2 is achieved by writing to the `m_Glow` object embedded in each player pawn. The game renders a glow effect around entities based on this data — no direct drawing needed, the engine handles it.

---

## Core Memory Layout

```
Entity Pawn Address (e.g. 0x7FF8A4D00000)
└── +0xCC0: m_Glow  (CGlowProperty)
    ├── +0x30: m_GlowStyle  (int) — must be set to 3 for glow effect
    ├── +0x40: m_GlowColor  (Color, RGBA layout)
    │            R = offset +0x43
    │            G = offset +0x42
    │            B = offset +0x41
    │            A = offset +0x44
    └── +0x51: m_bGlowOccluded  (bool) — set to true to keep glow active
```

---

## GlowESP Flow (Step by Step)

1. **Read entity list** — iterate through `gGame.GetEntityListEntry() + (i * 0x70)` for up to 64 slots. Each slot contains a pointer to a `PlayerController`.
2. **Skip local player** — compare the controller address against `LocalEntity.Controller.Address`.
3. **Read pawn address** — call `Entity.UpdateController()` then `Entity.UpdatePawn(Entity.Pawn.Address)`.
4. **Team check** — if `MenuConfig::TeamCheck` is enabled, skip teammates using `Entity.Controller.TeamID == LocalEntity.Controller.TeamID`.
5. **Alive check** — skip dead players via `Entity.IsAlive()`.
6. **Calculate glow base**: `pawnAddr + 0xCC0` (`m_Glow` offset).
7. **Set glow type** — read `glowBase + 0x30`, if not `3`, write `3`.
8. **Write glow color** — pack RGBA into a `DWORD` with ARGB layout and write to `glowBase + 0x40`.
9. **Enable glow** — write `true` to `glowBase + 0x51`.

---

## Required Offsets

### Global Offsets (`Offsets.h` — `Offset::` namespace)

| Offset Name | Hex Value | Description |
|---|---|---|
| `EntityList` | `0x24B3268` | Base pointer to entity list in client.dll |
| `LocalPlayerController` | `0x22F8028` | Pointer to local player's controller |
| `LocalPlayerPawn` | `0x206D9E0` | Pointer to local player's pawn |
| `Matrix` | `0x2313F10` | View matrix for world-to-screen |
| `ViewAngle` / `a_ViewAngle` | `0x231E9B8` | Current view angles |
| `GlobalVars` | `0x2062540` | Global game variables (tick count, time, etc.) |
| `ForceJump` | `0x2066C70` | Bunnyhop force jump address |
| `CSGOInput` | `0x231E330` | Input system address |

### Entity List Stride

| Name | Value | Description |
|---|---|---|
| Entity entry stride | `0x70` | Each entity controller entry is spaced `0x70` bytes apart in the list |

### Controller Offsets (`Offset::Entity`)

| Offset Name | Hex Value | Netvar / Description |
|---|---|---|
| `m_hPawn` | `0x6C4` | Handle to the player's pawn |
| `iszPlayerName` | `0x6F8` | Player display name string |
| `m_iPendingTeamNum` | `0x840` | Pending team assignment |
| `m_iPawnHealth` | `0x918` | Health stored on controller |
| `m_bPawnIsAlive` | `0x914` | Alive flag on controller |

### Pawn Offsets (`Offset::Pawn`)

| Offset Name | Hex Value | Netvar / Description |
|---|---|---|
| `Pos` / `m_vOldOrigin` | `0x1588` | Player world position |
| `CurrentHealth` | `0x354` | `m_iHealth` |
| `MaxHealth` | `0x350` | `m_iMaxHealth` |
| `m_iTeamNum` | `0x3F3` | Team number (T=2, CT=3) |
| `fFlags` | `0x400` | `m_fFlags` (in air, crouching, etc.) |
| `m_pGameSceneNode` | `0x338` | Pointer to scene node (bones live here) |
| `BoneArray` | `0x1E0` | Bone matrix array (0x160 + 0x80 inside scene node) |
| `angEyeAngles` | `0x3DD0` | Eye angles |
| `vecLastClipCameraPos` | `0x3DA4` | Camera position |
| `iShotsFired` | `0x270C` | `m_iShotsFired` |
| `aimPunchAngle` | `0x16CC` | Recoil punch angle |
| `aimPunchCache` | `0x16E8` | Recoil punch cache vector |
| `iIDEntIndex` | `0x3EAC` | `m_iIDEntIndex` (aim target entity) |
| `CameraServices` | `0x1410` | Camera services pointer |
| `iFovStart` | `0x294` | FOV start (inside camera services) |
| `flFlashDuration` | `0x15F8` | Flashbang alpha overlay |
| `bSpottedByMask` | `0x26EC` | `m_entitySpottedState + 0xC` |
| `m_bIsScoped` | `0x26F8` | Is player scoped |
| `m_pWeaponServices` | `0x13D8` | Weapon services pointer |
| `pClippingWeapon` | `0x3DC0` | Current clipping weapon |

### Glow-Specific Offsets (hardcoded in GlowESP.hpp)

| Offset | Value | Description |
|---|---|---|
| `m_Glow` | `+0xCC0` | Offset from pawn to `CGlowProperty` |
| `m_GlowStyle` | `+0x30` | Glow type — set to `3` |
| `m_GlowColor` | `+0x40` | Color (Color struct: r+g+b+a packed DWORD) |
| `m_bGlowOccluded` | `+0x51` | Bool to keep glow active |

---

## Color Packing (ARGB DWORD)

```cpp
// ImColor stores: x=R, y=G, z=B, w=A (0.0–1.0 range)
// CS2 Color struct stores: A, R, G, B byte layout in memory

DWORD colorArgb = ((DWORD)(glowColor.Value.w * 255) << 24) | // Alpha
                  ((DWORD)(glowColor.Value.z * 255) << 16) | // Red
                  ((DWORD)(glowColor.Value.y * 255) << 8)  | // Green
                  ((DWORD)(glowColor.Value.x * 255));         // Blue
ProcessMgr.WriteMemory<DWORD>(glowBase + 0x40, colorArgb);
```

---

## Entity Iteration Logic

```
for (int i = 0; i < 64; i++)
{
    // Entity list is an array of pointers, each entry = 0x70 bytes
    DWORD64 entityAddress = 0;
    ReadMemory(gGame.GetEntityListEntry() + (i * 0x70), entityAddress);
    
    if (entityAddress == 0 || entityAddress == LocalEntity.Controller.Address)
        continue;
    
    // Read controller → pawn → apply glow
}
```

---

## Common Issues

- **Glow not visible**: Ensure `m_GlowStyle` at `+0x30` is set to `3`. Other values disable the glow.
- **Team players glowing**: The team check depends on `Entity.Controller.TeamID` matching local — verify `m_iTeamNum` (`0x3F3`) is correct for your build.
- **Stale entity pointers**: Always validate that the controller and pawn addresses are non-zero before reading.
- **Crash on read**: Wrap each `ReadMemory` / `WriteMemory` call with error checking or use the existing `ProcessMgr` wrapper.

---

## Updating Offsets

Offsets can drift after game updates. The project includes a pattern scanner in `Utils/OffsetUpdater.hpp` and a `Signatures` namespace with byte patterns for dynamic resolution. Run the offset updater or re-dump offsets using a tool like [cs2-dump](https://github.com/a2x/cs2-dump) after each game patch.

#pragma once
#include "MenuConfig.hpp"
#include "Entity.h"

namespace GlowESP
{
    inline bool Enable = false;
    inline ImColor EnemyColor = ImColor(255, 50, 50, 255);
    inline ImColor TeamColor = ImColor(50, 150, 255, 255);

    inline void Run(const CEntity& LocalEntity)
    {
        if (!MenuConfig::ShowChams) return;

        for (int i = 0; i < 64; i++)
        {
            DWORD64 entityAddress = 0;
            if (!ProcessMgr.ReadMemory<DWORD64>(gGame.GetEntityListEntry() + (i + 1) * 0x70, entityAddress))
                continue;

            if (entityAddress == LocalEntity.Controller.Address)
                continue;

            CEntity Entity;
            if (!Entity.UpdateController(entityAddress))
                continue;

            if (!Entity.UpdatePawn(Entity.Pawn.Address))
                continue;

            if (MenuConfig::TeamCheck && Entity.Controller.TeamID == LocalEntity.Controller.TeamID)
                continue;

            if (!Entity.IsAlive())
                continue;

            bool isEnemy = Entity.Controller.TeamID != LocalEntity.Controller.TeamID;
            ImColor glowColor = isEnemy ? MenuConfig::ChamsEnemyColor : MenuConfig::ChamsTeamColor;

            DWORD64 pawnAddr = Entity.Pawn.Address;
            DWORD64 glowBase = pawnAddr + 0xCC0; // m_Glow

            int currentGlowType = 0;
            ProcessMgr.ReadMemory<int>(glowBase + 0x30, currentGlowType);
            if (currentGlowType != 3) {
                int newType = 3;
                ProcessMgr.WriteMemory<int>(glowBase + 0x30, newType);
            }

            // Write Color (RGBA layout for CS2 Color struct)
            DWORD colorArgb = ((DWORD)(glowColor.Value.w * 255) << 24) |
                              ((DWORD)(glowColor.Value.z * 255) << 16) |
                              ((DWORD)(glowColor.Value.y * 255) << 8) |
                              ((DWORD)(glowColor.Value.x * 255));
            ProcessMgr.WriteMemory<DWORD>(glowBase + 0x40, colorArgb);

            // Keep it glowing
            bool bTrue = true;
            ProcessMgr.WriteMemory<bool>(glowBase + 0x51, bTrue);
        }
    }
}
```

