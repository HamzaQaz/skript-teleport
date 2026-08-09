# Skript style guide

How systems in `~/skripts` are built. Written from four folders that are in
production — `econ`, `teleport`, `filter`, `twists` — and from the parse
errors that got them there.

Read the **Landmines** section before writing any Skript. Most of it was
learned by a server refusing to load.

Target: **Paper 26.2**, **Skript 2.16.1+**, **no addons**. Addons are the most
common thing to break on a new Minecraft version, so everything is vanilla
Skript unless a feature genuinely cannot be done without one — and then the
dependency is documented as *required*, never "optional".

---

## 1. Shape of a system

One folder per system. One numbered file per concern. Each folder is its own
git repo, cloned straight into `plugins/Skript/scripts/<name>`.

```
00-core.sk       config tree, shared helpers, the GUI toolkit
01-<engine>.sk   the thing the system actually does
02-<gui>.sk      menus, if they are big enough to want their own file
0N-admin.sk      staff tooling, always last
README.md
.github/workflows/skript-load.yml
```

Numbering exists for `on load` ordering and for reading order. Nothing else
depends on it — functions and variables are global across every script on the
server the moment they load.

Keep files single-purpose. `04-lands.sk` is the Lands integration; if Lands
goes away, that file goes away and nothing else notices.

### Namespacing is not optional

**Every function name is global across every script on the server.** Two
scripts defining `gui_set` is a hard load error that breaks both. So every
folder owns a prefix, for functions *and* variable trees:

| Folder | Functions | Variables |
|---|---|---|
| econ | `cfg_` `ex_` `gui_` `cv_` `term_` `money` `gold` | `{cfg::*}` `{ex::*}` |
| teleport | `tp_` `tpa_` `tpui_` `rtp_` | `{tp::*}` `{tpa::*}` |
| filter | `fl_` `fcfg_` `fgui_` `disc_` | `{fcfg::*}` `{filter::*}` |
| twists | `tw_` `twui_` | `{tw::*}` |

Before adding a helper, grep the other folders for the name. Short generic
names (`fset`, `r`, `comma`) are how collisions happen.

---

## 2. Config lives in variables

`options:` blocks are **file-local**, so they are useless for a multi-file
system. Config goes in a variable tree, seeded once:

```skript
function tw_default(key: text, value: object):
    if {tw::cfg::%{_key}%} is not set:
        set {tw::cfg::%{_key}%} to {_value}

function tw_seed():
    tw_default("period", 604800)   # seconds between Switches
    tw_default("slots", 3)         # how many run at once

on load:
    tw_seed()
    log "[Twists] Core loaded."
```

**Editing a default in the file does nothing on a server that has already
loaded** — the variable exists. Say so in a comment at the top of the seeder
and again in the README, because it will confuse someone otherwise. Provide
`/<x>admin config [<key> <value>|reset]`.

Every folder logs one line on load. The CI asserts that line appears, so a
run where nothing loaded cannot pass by being quiet.

---

## 3. Landmines

Every one of these has broken a real load.

### Functions

- **A function containing a delay cannot return a value.** Skript drops out of
  function context at the first `wait`. Make it return nothing and report its
  own result.
- Don't call a text-returning function inside a string:
  `broadcast "%tw_grad("hi")%"` fails twice over — a string containing only a
  function call is rejected as a redundant wrapper, and nested quotes inside
  `%...%` don't parse at all. **Assign to a local, interpolate the local.**

```skript
set {_hdr} to tw_grad("THE SWITCH IS COMING")
broadcast "%twfx()%%{_hdr}%"
```

### Timespans

- `wait` and `apply ... for ...` **will not take a variable timespan.**
  Durations must be literals. To wait a computed number of seconds:

```skript
loop {_wait} times:
    wait 1 second
```

### Loops

- **Never `apply X to loop-player for 3 seconds`** — the parser reads
  `loop-player for 3 seconds` as a loop reference and fails. Copy it out:

```skript
loop all players:
    set {_pl} to loop-player
    apply speed 2 to {_pl} for 30 seconds
```

- Same for anything else that reads awkwardly after `loop-player`:
  `time in world of loop-player` mis-binds. Hoist it into a local.
- Nested loops use `loop-value-1` / `loop-value-2`, outermost first.
- **Never mutate a list you are looping.** Collect keys into a temporary list,
  then act on that list.
- A counter stored inside the tree it counts (`{ex::hist::size}` beside
  `{ex::hist::1}`) gets picked up by `size of {ex::hist::*}` and by loops over
  it. Either keep counters in a separate tree or skip the key explicitly.

### Events

- `drops` exists **only** in death and harvest events.
- `on heal` exposes `event-entity`, not `victim`.
- `on regain health` is not a structure. It is `on heal`.
- `time in <world>` is a *time*, not a tick number — compare with
  `{_w} is night` and set with a literal like `19:00`.
- Health is in **hearts**. A full player is `10`, not `20`.
- **Two handlers at the same priority run in arbitrary order.** If handler A
  needs to know what B did, A must check the state itself rather than assume
  ordering — otherwise behaviour differs between restarts.
- `on chat with priority lowest` is correct for anything that cancels a
  message, so relay plugins never see it. Just don't rely on ordering between
  two of them.

### Titles and effects

- `send title X with subtitle Y to {_p} for 3 seconds` fails: the `to %players%`
  slot only matches in one position, and a trailing duration pushes it out —
  the target then falls back to the command sender, which doesn't exist inside
  a function. Drop the duration.
- `without any particles` is not accepted by the `apply` effect.

### Items

- `ender pearl` is ambiguous in an assignment (teleport cause / damage type /
  item). Fine as a typed function argument; use `1 of ender pearl` otherwise.

---

## 4. GUIs

### The trap that gives away free items

Opening an inventory fires `InventoryCloseEvent` for the view it replaces —
**including the player's own inventory**. So the obvious pattern:

```skript
set {mygui::%uuid of player%} to true     # set the flag
open {_inv} to {_p}                       # ...which this immediately deletes
```

deletes its own flag during the open call, the click guard fails, `cancel
event` never runs, and **every item in the menu is takeable and draggable.**

Always open through a helper with a busy marker:

```skript
function twui_open(p: player, inv: inventory):
    set {twui::busy::%uuid of {_p}%} to true
    open {_inv} to {_p}
    set {twui::mode::%uuid of {_p}%} to "twists"
    delete {twui::busy::%uuid of {_p}%}

on inventory close:
    if {twui::busy::%uuid of player%} is set:
        stop
    delete {twui::mode::%uuid of player%}
```

**Exactly one `on inventory close` per folder.** A second one will delete the
first's state. The drag guard keys off the same mode variable, so a menu that
skips the helper also loses drag protection.

### Rules

- Click handlers open with the mode check, then `cancel event`, then
  `clicked inventory is event-inventory`.
- Resolve clicks through a slot map written at draw time —
  `{ui::map::<uuid>::<slot>}` — never by parsing the item's display name. A
  colour code in someone's nick breaks name parsing.
- Clear the map at the top of every draw, so a stale entry can't fire an
  action at the wrong target.
- Click type by substring: `set {_ct} to "%click type%" in lower case`, then
  `if {_ct} contains "right"`. The expression name has moved between versions;
  a substring test survives all of them.
- Build lore as a list variable (`clear {_l::*}` … `add "..." to {_l::*}`).
  Multiline continuation syntax varies between versions.
- Every lore line starts `<!italic>`.
- Scope narrow event hooks by inventory type (`type of event-inventory is
  anvil inventory`) so they don't collide with menus.

---

## 5. Voice

Comments explain **why**, never what. `# add 1 to the counter` is noise;
`# k should never grow on its own — if it does, someone minted value` is the
reason the line exists.

- A header block per file: what it owns, the model, the invariants.
- When you avoid a trap, document it **where the trap was**, in prose. Most of
  section 3 exists as comments in the code it applies to.
- Wrap around 76 columns. Full sentences.
- Say what a number is for: `tw_default("slots", 3)   # how many run at once`.
- Anything left unwired is marked **UNCONFIGURED** with what to fill in and
  how to find the answer. Guessing an integration is worse than admitting it —
  see `rtp_claim_free()`.
- Player-facing text is plain and slightly dry. No exclamation marks.

---

## 6. Testing

Skript reports parse errors at load and nowhere else. There is no linter that
understands its grammar, so a bad clause order sits in a commit looking fine.

Every repo has `.github/workflows/skript-load.yml`, which boots real Paper
with real Skript, drops the folder into `plugins/Skript/scripts/`, stops
cleanly, and fails on any Skript error. Everything is resolved from the
PaperMC API at run time — version, download URL, **and the minimum Java
version** — so it doesn't rot.

**Install the same plugins production has, or you get errors that aren't
real:**

- Vault alone registers no economy. `balance` only parses once a *provider*
  has registered, so CI installs EssentialsX too.
- `placeholder ...` needs skript-placeholders **and** PlaceholderAPI.

A green CI proves it **loads**. It does not prove the gameplay is right — say
so rather than implying it's tested.

---

## 7. Deploying

Each folder is a git repo cloned directly into the scripts directory:

```bash
cd plugins/Skript/scripts
git clone https://github.com/HamzaQaz/skript-econ.git econ
```

- **If the folder already has files in it, delete it first.** Git only manages
  its own tracked files; leftovers load alongside the clone and every reload
  fails with "function already exists".
- A pull never reloads Skript. Always `/sk reload <folder>` after.
- `.git/` inside the scripts folder is harmless — Skript only reads `.sk`.
- Filenames carry no download suffixes. Rename with `git mv` so history
  follows the file.

### Command conflicts

EssentialsX owns a lot of names, and **whichever plugin registers first
wins** — Skript loads late, so Essentials wins by default and your command
silently never runs. Disable them on the Essentials side; disabling a command
kills its aliases too.

Where a name is genuinely better left with Essentials (`/back`, `/tphere`),
take a different name rather than fighting for it.

---

## 8. Economy conventions

- **One unit.** The Vault balance *is* the unit. Anything Skript doesn't own —
  the Lands bank screens, PlaceholderAPI on a scoreboard, shop plugins —
  renders the raw Vault number and cannot be wrapped, so a sub-unit would be
  invisible to half the server's UI.
- Integers for execution, floats for display. Rounding is the house edge; it
  is also what pins a ticker to one value for a 1.25% band of price, so
  headline rates are computed analytically and only per-trade costs use the
  integer path.
- One `burn()` function every sink routes through, so `/ecoadmin burns` stays
  a true picture. Note the sinks that *can't* (Lands upkeep destroys money
  internally) rather than pretending the number is complete.
- Display conversion goes through one helper (`ipx()`), and internal maths
  never uses it. Never print a raw internal rate to a player.
