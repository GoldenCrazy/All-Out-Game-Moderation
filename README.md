# Game Moderation System

A suite of moderation tools including ban, temp ban, warn, and kick — with persistent ban state, full-screen overlays, and owner/editor-only chat commands.

---

## Features

- `/ban {username} {reason...}` — permanently bans a connected player with a multi-word reason
- `/tempban {username} {duration} {reason...}` — temporarily bans a player for a set duration (up to 6 weeks)
- `/unban {username}` — unbans a connected player
- `/warn {username} {reason...}` — warns a player with a dismissible full-screen overlay (session only)
- `/kick {username} {reason...}` — kicks a connected player from the game
- Bans **persist across reconnects** via the Save API
- Warnings are **session-only** and do not persist across reconnects
- Full-screen ban overlay shown to banned players (non-dismissible)
- Full-screen warn overlay shown to warned players (dismissible via a Close button)
- Banned players are **frozen**, **cannot use abilities**, and **cannot interact** with anything
- Temp bans **expire automatically** — even if the player stays online
- Commands are restricted to **game owner and editors only**

---

## Setup

There are two paths depending on whether you're starting from scratch with a clone of this project, or dropping the moderation system into your own existing game.

---

### Path A — Starting from a Clone of This Project

**1. Clone the repository**

If you have [Git](https://git-scm.com/downloads) installed, run this in a terminal:

```bash
git clone https://github.com/GoldenCrazy/All-Out-Game-Moderation.git
```

**Don't know how to use a terminal? Here's the easy way using VS Code:**

1. Download and install [Visual Studio Code](https://code.visualstudio.com/) if you don't have it
2. Open VS Code
3. Press `Ctrl+Shift+P` (or `Cmd+Shift+P` on Mac) to open the Command Palette
4. Type `Git: Clone` and select it
5. Paste in this URL:
   ```
   https://github.com/GoldenCrazy/All-Out-Game-Moderation.git
   ```
6. Choose a folder on your computer to save the project into
7. When it finishes, click **Open** to open the cloned project

> If VS Code says Git isn't installed, download it from [git-scm.com](https://git-scm.com/downloads), install it, then restart VS Code and try again.

Once the project is open, continue below.

**2. Open `ao.project`** in the root of the project.

Find these two lines:

```
name: "Cool Game"
id:   "game id here"
```

Replace them with your actual game name and game ID from your game link:

```
name: "Fat Simulator"
id:   "654eedec2412b8afb8a993ca"
```

> You can find your game ID in your public games link.

**3. You're done.** All systems are already wired up — `ban_system.csl`, `warn_system.csl`, and `kick_system.csl` are included and `main.csl` already has all the hooks in place. Jump straight to [Using the Commands](#using-the-commands).

---

### Path B — Adding to an Existing Game

If you already have a game and just want to drop these systems in, follow these steps.

**1. Copy the system files** into your project alongside your existing `.csl` files. Don't modify them — they are self-contained.

- `ban_system.csl`
- `warn_system.csl`
- `kick_system.csl`

**2. Hook them into your `main.csl`** by adding the following fields and calls in the matching places.

#### On your `Player` class, add these fields:

```csl
Player :: class : Player_Base {
    is_banned:           bool;
    ban_reason:          string;
    ban_expires_at_s:    s64;
    ban_freeze_applied:  bool;
    is_warned:           bool;
    warn_reason:         string;

    // ... rest of your player class
}
```

#### In `Player.ao_start` — restore ban state on join:

```csl
ao_start :: method() {
    load_ban_state(this);  // ← add this
}
```

#### In `Player.ao_update` — handle ban expiry and freeze:

```csl
ao_update :: method(dt: float) {
    // Clear temp bans once they expire, even if the player stays online.
    current_time_s := (get_nanoseconds_since_epoch() / 1000000000).(s64);
    if is_banned && ban_expires_at_s > 0 && current_time_s >= ban_expires_at_s {
        clear_ban_state(this);  // ← add this block
    }

    // Apply the freeze once while banned, and remove it once when unbanned.
    if is_banned && !ban_freeze_applied {
        add_freeze_reason("banned");
        ban_freeze_applied = true;
    } else if !is_banned && ban_freeze_applied {
        remove_freeze_reason("banned");
        ban_freeze_applied = false;
    }
}
```

#### In `Player.ao_late_update` — draw the warn and ban screens:

```csl
ao_late_update :: method(dt: float) {
    if is_local_or_server() {
        draw_warn_screen(this);  // ← add this
        draw_ban_screen(this);   // ← add this
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

That's everything. All commands are automatically registered via `@chat_command` — no extra registration needed.

---

## Using the Commands

All commands are restricted to the **game owner and editors** only.

### Ban a player (permanent)

```
/ban {username} {reason}
```

The reason can be multiple words — everything after the username is captured.

```
/ban Matt Exploiting and scamming
/ban ot_golden Spamming the chat
/ban Mas No reason given
```

### Temp ban a player

```
/tempban {username} {duration} {reason}
```

Duration format: a number followed by a unit — `m` (minutes), `h` (hours), `d` (days), or `w` (weeks). Maximum duration is `6w`.

```
/tempban Matt 1h Spamming the chat
/tempban ot_golden 7d Exploiting
/tempban Mas 2w Repeated offenses
```

The ban screen shows the time remaining and updates live. The ban expires automatically whether the player is online or offline.

### Unban a player

```
/unban {username}
```

```
/unban ot_golden
```

### Warn a player

```
/warn {username} {reason}
```

The reason can be multiple words.

```
/warn Matt Please follow the rules
/warn ot_golden Stop spamming
```

The warned player sees a full-screen overlay with the reason. They can dismiss it by clicking the **Close** button. Warnings are not saved — they clear when the player disconnects.

### Kick a player

```
/kick {username} {reason}
```

```
/kick Matt Breaking the rules
/kick ot_golden No reason given
```

> **Note:** The player must be **currently connected** to the game for all commands to work. Ban state persists — if you ban someone while they're online, they will still be banned when they rejoin.

---

## How Persistence Works

Ban state is saved per-player using the Save API. Warnings and kicks are not persisted.

| Key              | Value                            |
|------------------|----------------------------------|
| `ban_status`     | `"1"` if banned, `"0"` if not   |
| `ban_reason`     | The reason string                |
| `ban_expires_s`  | Unix timestamp (seconds) of expiry, or `0` for permanent |

When a player joins, `load_ban_state` reads these keys and restores their ban state automatically. If a temp ban has already expired, it is cleared immediately on join. When a player is unbanned, all keys are cleared.

---

## File Reference

| File               | Purpose                                                   |
|--------------------|-----------------------------------------------------------|
| `ban_system.csl`   | Ban, temp ban, and unban logic, commands, and UI          |
| `warn_system.csl`  | Warn command and dismissible warning overlay UI           |
| `kick_system.csl`  | Kick command                                              |
| `main.csl`         | Your game entry point — hooks into all moderation systems |
| `ao.project`       | Project config — update `name` and `id` if cloning       |