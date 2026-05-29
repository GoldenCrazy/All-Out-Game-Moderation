# Game Ban System

A ban/unban system with persistent ban state, a full-screen ban UI, and owner/editor-only chat commands.

---

## Features

- `/ban {username} {reason...}` — bans a connected player with a multi-word reason
- `/unban {username}` — unbans a connected player
- Bans **persist across reconnects** via the Saved Data
- Full-screen ban overlay shown to banned players
- Banned players are **frozen**, **cannot use abilities**, and **cannot interact** with anything
- Commands are restricted to **game owner and editors only**

---

## Setup

There are two paths depending on whether you're starting from scratch with a clone of this project, or dropping the ban system into your own existing game.

---

### Path A — Starting from a Clone of This Project

If you cloned this repository and are building your game on top of it, you need to configure the project with your own game details before doing anything else.

**1. Open `ao.project`** in the root of the project.

Find these two lines:

```
name: "Cool Game"
id:   "game id here"
```

Replace them with your actual game name and game ID from your game link:

example:
```
name: "Fat Simulator"
id:   "654eedec2412b8afb8a993ca"
```

> You can find your game ID in your public games link.

**2. You're done.** The ban system is already wired up — `ban_system.csl` is included and `main.csl` already has all the hooks in place. Jump straight to [Using the Commands](#using-the-commands).

---

### Path B — Adding to an Existing Game

If you already have a game and just want to drop the ban system in, follow these steps.

**1. Copy `ban_system.csl`** into your project alongside your existing `.csl` files. Don't modify it — it's self-contained.

**2. Hook it into your `main.csl`** by adding the following calls in the matching places:

#### On your `Player` class, add the two fields:

```csl
Player :: class : Player_Base {
    is_banned:  bool;
    ban_reason: string;

    // ... rest of your player class
}
```

#### In `Player.ao_start` — restore ban state on join:

```csl
ao_start :: method() {
    load_ban_state(this);  // ← add this
}
```

#### In `Player.ao_update` — freeze banned players every frame:

```csl
ao_update :: method(dt: float) {
    if is_banned {
        add_freeze_reason("banned");  // ← add this
    }
}
```

#### In `Player.ao_late_update` — draw the ban screen:

```csl
ao_late_update :: method(dt: float) {
    if is_local_or_server() {
        draw_ban_screen(this);  // ← add this
    }
}
```

#### In `Player.ao_can_use_ability` — block abilities for banned players:

```csl
ao_can_use_ability :: method(ability: Ability_Base) -> bool {
    if is_banned {
        return false;  // ← add this block
    }
    return true;
}
```

#### In `Player.ao_can_use_interactable` — block interactions for banned players:

```csl
ao_can_use_interactable :: method(interactable: Interactable) -> bool {
    if is_banned {
        return false;  // ← add this block
    }
    return true;
}
```

That's everything. The `/ban` and `/unban` commands are automatically registered via `@chat_command` — no extra registration needed.

---

## Using the Commands

Both commands are restricted to the **game owner and editors** only for now, this may change later on.

### Ban a player

```
/ban {username} {reason}
```

The reason can be multiple words — everything after the username is captured.

```
/ban Matt Exploiting and scamming
/ban ot_golden Spamming the chat
/ban Sparkbot No reason given
```

### Unban a player

```
/unban {username}
```

```
/unban ot_golden
```

> **Note:** The player must be **currently connected** to the game for `/ban` and `/unban` to work. The ban state itself persists — if you ban someone while they're online, they will still be banned when they rejoin.

---

## How Persistence Works

Ban state is saved per-player using the Save API with two keys:

| Key           | Value                          |
|---------------|--------------------------------|
| `ban_status`  | `"1"` if banned, `"0"` if not |
| `ban_reason`  | The reason string              |

When a player joins, `load_ban_state` reads these keys and restores their ban state automatically. When a player is unbanned, both keys are cleared.

---

## File Reference

| File             | Purpose                                              |
|------------------|------------------------------------------------------|
| `ban_system.csl` | Self-contained ban logic, commands, and UI           |
| `main.csl`       | Your game entry point — hooks into the ban system    |
| `ao.project`     | Project config — update `name` and `id` if cloning  |

---
