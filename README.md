# Teleport Suite

TPA with a proper chest UI, and a two-ring wilderness teleport.

Drop this whole folder into `plugins/Skript/scripts/` so you have
`plugins/Skript/scripts/teleport/`. Skript loads subfolders automatically.
Then `/sk reload teleport`.

## Files

| File | What it does |
|---|---|
| `00-core.sk` | Config tree, formatting, cooldowns, payment, **the warmup engine**, chest-UI toolkit, `/tpback` |
| `01-tpa.sk` | Request model, expiry, and every `/tpa*` command |
| `02-tpa-gui.sk` | Hub, player picker, settings, block list |
| `03-rtp.sk` | The whole wilderness teleport — `/rtp`, `/rtp far` |
| `05-admin.sk` | `/tpadmin`, `/rtpadmin` |

Load order matters only for `on load` seeding, which is why files are
number-prefixed. Functions and variables are global across all scripts;
`options:` blocks are **not**, which is why config lives in `{tp::cfg::*}`.

Every function is prefixed `tp_` / `tpa_` / `tpui_` / `rtp_` so this can
sit alongside the Ducat economy. Both suites define their own GUI helpers
and both would happily be called `gui_set` — duplicate function names are a
hard load error in Skript, not a warning.

## Requirements

- **Paper 26.2**, **Skript 2.16.1+** — same targets as the economy folder
- **Vault** + an economy provider, but only if you set a teleport cost.
  Leave `tpa_cost` and `rtp_cost` at `0` and nothing touches Vault.

No GUI addon. Vanilla Skript inventory events throughout, same reasoning as
the economy scripts — GUI addons are the most common thing to break on a
new Minecraft version.

## ⚠ Read this first: EssentialsX owns these command names

EssentialsX registers `/tpa`, `/tpahere`, `/tpaccept`, `/tpdeny`,
`/tpacancel` and `/tptoggle`. Whichever plugin registers a name first wins
it; the loser is only reachable as `/essentials:tpa`. Skript loads late, so
**Essentials wins by default and you will be using its TPA, not this one.**

Fix it on the Essentials side. In `plugins/Essentials/config.yml`:

```yaml
disabled-commands:
  - tpa
  - tpahere
  - tpaccept
  - tpdeny
  - tpacancel
  - tptoggle
  - tpauto
```

Then `/essentials reload` and `/sk reload teleport`. Verify with
`/help tpa` — it should show this script's description.

`/back` is deliberately **not** taken. Essentials' `/back` has death-return
behaviour people rely on; this suite adds `/tpback` (`/tpb`) for
teleport-return only. If you'd rather have one command, disable Essentials'
`back` and add `back` to the aliases line in `00-core.sk`.

## Permissions

| Node | For |
|---|---|
| `teleport.tpa` | All `/tpa*` commands and the hub |
| `teleport.rtp` | `/rtp` |
| `teleport.back` | `/tpback` |
| `teleport.admin` | `/tpadmin`, `/rtpadmin` |
| `teleport.bypass.cooldown` | Skips every cooldown |
| `teleport.bypass.warmup` | Instant teleports, no countdown |
| `teleport.bypass.cost` | Free |

## Config

Everything lives in `{tp::cfg::*}`, seeded on first load. Editing the
defaults in `00-core.sk` will **not** apply afterwards — the variable
already exists.

```
/tpadmin config                     list every key
/tpadmin config <key> <value>       set one
/tpadmin config reset               wipe and reseed from file
/tpadmin info                       pending requests, current rules
```

The keys worth knowing:

| Key | Default | Notes |
|---|---|---|
| `tpa_expire` | 120 | Seconds a request survives |
| `tpa_warmup` | 3 | Countdown after accepting. `0` = instant |
| `tpa_cooldown` | 15 | Between *sending* requests |
| `tpa_max_out` | 5 | Simultaneous outgoing requests |
| `rtp_cooldown` | 300 | The one number players will argue about |
| `rtp_water` | false | Allow landing on water and in ocean biomes |
| `nofall` | 8 | Seconds of fall immunity after any teleport |
| `move_tolerance` | 0.75 | Blocks of slack during a warmup |

## The RTP

Two commands, two rings:

```
/rtp        somewhere within 75,000 blocks of 0,0
/rtp far    somewhere between 75,000 and 100,000 blocks
```

Points are sampled uniformly over **area**, not radius (`r = sqrt(u)` scaled
between the rings). Picking a uniform radius is the classic mistake — it
crowds everyone into the middle and the outer half never gets used.

### Why it flies you around

`highest block at` on an unloaded chunk calls `syncLoad`, which **blocks the
main thread** until that chunk is read or generated. At these distances every
candidate is virgin terrain, so testing them from a background loop parks the
whole server — it shows up in a profiler as ~97% of server thread time in
`LockSupport.parkNanos`. That is not lag, that is the tick loop waiting.

Teleporting a player is the one operation Paper loads chunks *asynchronously*
for. So the search flies the player to each candidate instead of reaching out
to it: hop to y=320 above a random point, wait 4 ticks for the chunk to
arrive, then read blocks — free, because they are now resident. And since the
player's view distance has loaded the neighbourhood, `rtp_sweep()` tests 24
more points nearby for nothing. One chunk load buys 25 candidates.

**Do not turn this back into a background task.** There is no cache and there
should not be one.

| Key | Default | |
|---|---|---|
| `rtp_near_min` / `rtp_near_max` | 0 / 75000 | `/rtp` |
| `rtp_far_min` / `rtp_far_max` | 75000 / 100000 | `/rtp far` |
| `rtp_hops` | 12 | teleports before giving up — one async chunk load each |
| `rtp_sweep` | 24 | free checks per hop — **raise this before raising hops** |
| `rtp_probe` | 320 | hover height; 250 in roofed worlds |
| `rtp_worlds` | `world` | comma separated; anywhere else refuses outright |

`/rtpadmin info` shows the rings and the average hops per trip. An average
near the limit means the safety filters are too tight for your terrain.

Only worlds in `rtp_worlds` are allowed. The End and any void world would
spend every hop over nothing and then fail, so they are refused up front
rather than wasting the search.

### Disk cost

At 75–100k blocks out, every trip generates brand-new terrain that is then
written to disk permanently. A few hundred `/rtp far` uses will add real
gigabytes to your world folder and there is no way around that — it is
inherent to sending people somewhere nobody has been. Watch the world size,
and consider a world border if it gets away from you.

## Before you go live

1. **Disable the Essentials commands.** See above. This is the one step
   that will otherwise silently make the whole folder look broken.

2. **Check your world border.** The far ring reaches 100,000 blocks. If you
   have a border smaller than that, every far candidate lands outside it and
   the search will always fail.

3. **Wire up the claim check if you ever claim that far out.**
   `rtp_claim_free()` in `03-rtp.sk` returns `true` unconditionally and is
   marked UNCONFIGURED. Your Lands claim-worlds list is `world` only and
   nobody has claimed near 75,000 blocks, so it does not bite yet.

4. **Move off flat file — you are already past this point.** The console is
   printing *"Cannot write variables to the database 'default' at sufficient
   speed; many variables will be lost if the server crashes."* That is
   Skript's flat-file backend failing to keep up, and it means real data
   loss on a crash. Switch the variable backend to SQLite in
   `plugins/Skript/config.sk` before anything else in this list.

## Commands

```
/tpa                    the hub
/tpa <player>           ask to teleport to them
/tpahere <player>       ask them to come to you
/tpaccept [player]      newest request if you don't name one
/tpdeny [player]
/tpacancel [player]
/tpatoggle              do not disturb
/tpablock [player]      no argument opens the block list
/tpback                 undo your last teleport

/rtp                    within 75,000 blocks
/rtp far                75,000 to 100,000 blocks

/tpadmin info | requests | warmups | config | clear <player|all>
/rtpadmin info
```

## Notes on the internals

- **Warmups are a token, not a flag.** `{tp::warm::<uuid>}` holds the unix
  time the countdown started; cancelling is a delete. The loop checks its
  own token every quarter second and stops if it changed, so there is no
  boolean to get stuck true and no way to land two teleports from one
  accept.
- **`/tpaccept` resolves the destination at the *end* of the countdown**,
  not the start, when the other player is online. That's what makes it feel
  right when they're walking.
- **The block list tells senders the same thing as do-not-disturb.**
  Telling someone they've been blocked individually is how block lists turn
  into an argument in public chat.
- **Menus never trust the item they drew.** Every clickable head writes
  `{tpui::map::<uuid>::<slot>}` on draw and the click handler reads the
  UUID back from there. Parsing it out of the display name breaks the
  moment someone has a colour code in their nick.
- **`tpui_open()` exists because of a Bukkit quirk.** Opening an inventory
  fires a close event for the one already open — including the player's own
  inventory — so the naive "set flag on open, clear on close" pattern
  deletes its own flag every time and every click sails past uncancelled.
  Same fix as the economy folder's `gui_open()`. Never call `open ... to
  player` directly for a menu.
- **`rtp_tps()` is the only line that reads server TPS.** If your Skript
  build doesn't parse `tps`, that one function is all you need to edit —
  return `20` and the cache builder simply stops backing off under load.
