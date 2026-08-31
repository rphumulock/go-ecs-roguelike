# Go ECS Roguelike

A tile-based roguelike in Go: procedurally generated dungeons, shadowcasting
field-of-view, A\* monster pursuit, and d10 melee combat — with every piece of
game state living in an Entity-Component-System world rather than a class
hierarchy.

## Why this project exists

I wanted to learn ECS by writing one, not by reading about one.

Entity-Component-System gets described in the abstract as
"composition over inheritance," which is easy to nod along to and hard to
actually feel until you've been on the other side of it. A roguelike is a good
place to feel it: the player and a skeleton are the same kind of thing to the
engine — a bag of components — and the difference between them is which
components they happen to carry. Rendering doesn't ask "is this a monster?", it
asks "does this have a `Position` and a `Renderable`?" Combat doesn't know what
a player is, only that both sides have `Health`, `Armor`, and a `MeleeWeapon`.
Adding a trait to a creature is adding a row to a struct list; it isn't a new
subclass.

It was also my excuse to write Go somewhere other than services and CLIs. Game
loops push on things backend Go rarely does: a fixed 60Hz `Update`/`Draw` cycle,
a turn state machine layered on top of a real-time loop, mutable state shared
across systems with no request boundary to hide behind, and immediate-mode
rendering where you redraw the world from scratch every frame.

And a roguelike makes you write the algorithms yourself. There's no library that
generates the dungeon, decides what the player can see, or walks a skeleton
around a corner toward you. Rooms-and-corridors generation, a visibility pass
with remembered-but-unseen tiles, and A\* pathfinding over a tile grid are all
things you either implement or don't have.

> [!NOTE]
> This is a finished-and-parked learning project, not a game I'm shipping. It
> plays: you explore, you fight, you die. It has no items, no levels below the
> first, and no win condition. The bugs listed under
> [Known rough edges](#known-rough-edges) are left in on purpose — they're where
> I stopped, and several of them are the interesting part.

## Stack

| Layer | Tool |
| --- | --- |
| Language | [Go 1.20](https://go.dev/) |
| Engine | [Ebitengine v2](https://ebitengine.org/) — windowing, sprite blitting, input, text |
| ECS | [bytearena/ecs](https://github.com/bytearena/ecs) — component registry, entities, tag queries |
| Field of view | [norendren/go-fov](https://github.com/norendren/go-fov) — shadowcasting over a grid you implement two methods for |
| Randomness | `crypto/rand` — cryptographically secure dice, which is overkill and was free |

## Running

Ebitengine compiles against native graphics libraries via cgo, so a bare
`go build` needs the platform's OpenGL/windowing headers. On Debian/Ubuntu
(including WSL):

```shell
sudo apt install libgl1-mesa-dev xorg-dev libasound2-dev
```

macOS needs Xcode command line tools; Windows needs nothing extra. See
[Ebitengine's install docs](https://ebitengine.org/en/documents/install.html)
for other platforms.

Then:

```shell
git clone https://github.com/rphumulock/go-ecs-roguelike.git
cd go-ecs-roguelike
go run .
```

Assets are loaded from `assets/` by relative path, so run from the repo root.

### Controls

| Key | Action |
| --- | --- |
| Arrow keys | Move — or attack, if a creature is in that square |
| `Q` | Wait, passing the turn to the monsters |

The window is 1280×960: an 80×50 tile play area at 16px per tile, with the
bottom 10 rows reserved for UI — message log on the left, player stats on the
right.

## How it works

### The world is components and tags

There are no `Player` or `Monster` classes. `world.go` registers a flat set of
components on the ECS manager at startup:

```
player  position  renderable  movable  monster
health  meleeWeapon  armor  name  userMessage
```

An entity is whatever components you hang on it. The player is built by chaining
them:

```go
manager.NewEntity().
    AddComponent(player, Player{}).
    AddComponent(renderable, &Renderable{Image: playerImg}).
    AddComponent(position, &Position{X: x, Y: y}).
    AddComponent(health, &Health{MaxHealth: 30, CurrentHealth: 30}).
    AddComponent(meleeWeapon, &MeleeWeapon{Name: "Battle Axe", MinimumDamage: 10, ...}).
    AddComponent(armor, &Armor{Name: "Plate Armor", Defense: 15, ArmorClass: 18}).
    ...
```

A skeleton is the same call with different values and `monster` in place of
`player`. `Player`, `Monster`, and `Movable` are empty structs — pure markers
that exist only to be queried on.

**Tags are the query language.** `ecs.BuildTag` bundles a set of components into
a named filter, and systems iterate whatever matches:

| Tag | Components required | Read by |
| --- | --- | --- |
| `players` | player, position, health, meleeWeapon, armor, name, userMessage | input, combat, HUD |
| `monsters` | monster, position, health, meleeWeapon, armor, name, userMessage | monster AI, combat |
| `renderables` | renderable, position | the render system |
| `messengers` | userMessage | the message log |

The payoff is that `renderables` doesn't care what an entity *is*. Anything with
a sprite and a location gets drawn — add a component pair to an item, a corpse,
or a trap and it renders with no change to `render_system.go`.

### The loop and the turn state machine

Ebitengine drives a real-time loop — `Update` at 60 ticks per second, `Draw` at
the display refresh rate. A turn-based game has to sit on top of that:

```go
func (g *Game) Update() error {
    g.TurnCounter++
    if g.Turn == PlayerTurn && g.TurnCounter > 8 {
        TakePlayerAction(g)
    }
    if g.Turn == MonsterTurn {
        UpdateMonster(g)
    }
    return nil
}
```

`TurnState` cycles `PlayerTurn → MonsterTurn → PlayerTurn`. The player half
blocks until input arrives; the monster half resolves every monster in one tick
and immediately hands control back. The `TurnCounter > 8` gate is a rate limit —
without it, a held arrow key would fire 60 moves a second.

`Draw` is pure output and runs every frame regardless of whose turn it is: draw
the level, then renderables, then the log panel, then the HUD.

### Level generation

`GenerateLevelTiles` is the classic rooms-and-corridors algorithm. Start with a
grid that is solid wall, then make 30 attempts to place a room:

1. Roll a rectangle 6–10 tiles on a side at a random position.
2. Reject it if it intersects any room already placed.
3. Carve its interior to floor.
4. Connect its center to the previous room's center with an L-shaped tunnel —
   a coin flip decides whether the corner goes horizontal-then-vertical or the
   other way.

Because failed placements are simply dropped, a level ends up with somewhere
under 30 rooms, and the "connect to the previous room" rule guarantees the whole
level is reachable without a separate connectivity pass.

The player starts at the center of room 0. Every other room gets a coin-flip
monster:

| | Health | Weapon | Damage | To-hit | Armor | AC |
| --- | --- | --- | --- | --- | --- | --- |
| Player | 30 | Battle Axe | 10–20 | +3 | Plate (15) | 18 |
| Orc | 30 | Machete | 4–8 | +1 | Leather (5) | 6 |
| Skeleton | 10 | Short Sword | 2–6 | +0 | Bone (3) | 4 |

### Sight and fog of war

`go-fov` does shadowcasting, but it needs to ask the map questions. `Level`
implements its two-method interface — `InBounds(x, y)` and `IsOpaque(x, y)` —
and that's the entire integration.

After each player move, `PlayerVisible.Compute(level, x, y, 8)` recalculates
what's visible within radius 8. Drawing then has three cases per tile:

- **Visible now** — drawn at full alpha, and flagged `IsRevealed`.
- **Revealed before, not visible now** — drawn at 10% alpha. This is the fog of
  war: the map you remember, dimmed.
- **Never seen** — not drawn at all.

Entities use the *player's* visibility even in the render system, so a monster
standing in a remembered-but-unlit room isn't shown. Monsters each compute their
own separate FOV, at the same radius, to decide whether they can see you.

### Combat

Melee resolution is one function, `AttackSystem`, and it's symmetric — it looks
up an attacker and defender by position across both the `players` and `monsters`
tags, so it has no idea which side is which:

```
d10 + weapon.ToHitBonus > defender.ArmorClass    →  hit
damage = rand(weapon.Min .. weapon.Max) - defender.Defense   (floored at 0)
```

Nothing is printed directly. The result is written into the combatants'
`UserMessage` components, and `ProcessUserLog` drains those into the on-screen
log each frame — systems communicate through components, not calls. A defender
dropped to 0 health gets a `DeadMessage`; the log system is what actually
disposes the entity after displaying it. If the defender was the player, the
turn state flips to `GameOver` and input stops being read.

### Monster AI

Each monster, on its turn:

1. Computes its own FOV. If it can't see the player, it does nothing — monsters
   across the level don't beeline toward you.
2. If the player is at Manhattan distance 1, attacks.
3. Otherwise runs A\* to the player and takes one step along the path.

`astar.go` is a hand-rolled implementation over the tile grid: open and closed
lists as slices, Manhattan distance for the heuristic, four-way cardinal
movement, walls excluded when expanding neighbors. Paths are reconstructed by
walking parent pointers back from the goal and reversing.

## Layout

```
main.go             Game struct, Ebitengine Update/Draw/Layout, entry point
world.go            Component registry, entity spawning, tag definitions
components.go       Every component type (Position, Health, MeleeWeapon, ...)

gamedata.go         Screen/tile/UI dimensions — the one place sizes are defined
map.go              GameMap: dungeons and the current level
dungeon.go          Dungeon: a named list of levels
level.go            Tiles, room carving, tunnels, fog-of-war drawing, FOV hooks
rect.go             Rectangle helpers: center, intersection test

player_system.go    Input → move or attack, then advance the turn
monster_system.go   Per-monster FOV, attack-or-pursue
combat_systems.go   To-hit and damage resolution for both sides
render_system.go    Blits every visible renderable
hud_system.go       Player stats panel
userlog_system.go   Message log panel; disposes dead entities

astar.go            A* pathfinding over the tile grid
dice.go             crypto/rand-backed dice rolls
turnstate.go        TurnState enum and transitions

assets/             Sprites and the UI panel (16x16 PNGs)
fonts/              Embedded M+ 1p font (vendored, currently unused — see below)
```

## Known rough edges

Left as-is. This is where the project stopped, and most of these are worth
reading precisely because they're the kind of thing this architecture makes easy
to get wrong.

- **The combat system holds stale globals.** `attacker` and `defender` in
  [`combat_systems.go`](./combat_systems.go) are package-level `*ecs.QueryResult`
  variables that are never reset to `nil` between attacks. The
  `if attacker == nil || defender == nil { return }` guard therefore only ever
  fires once, on the first call — after that, an attack that fails to match a
  defender silently resolves against whoever was in those variables last.

- **Dead monsters leave their square impassable.** The entity is disposed in
  `ProcessUserLog`, but nothing clears `Blocked` on the tile it was standing on.
  Since that tile isn't a `WALL`, walking into it routes to `AttackSystem` —
  which, per the bug above, then swings at a ghost.

- **Tile sprites are decoded from disk inside the generation loops.**
  `CreateTiles` calls `ebitenutil.NewImageFromFile("assets/wall.png")` once per
  tile — 4,000 file reads and PNG decodes per level — and `CreateRoom` does the
  same for `floor.png` on every floor tile, despite `loadTileImages()` already
  caching both into package globals.

- **A\* can index past the end of the tile array.** Its bounds checks compare
  against `gd.ScreenHeight` (60) instead of `levelHeight` (50, after the UI rows
  are subtracted). Expanding a node on the bottom row of the play area computes
  an index beyond `Tiles`.

- **A\*'s open-list check compares the wrong node.** When a neighbor is already
  queued, the code loops the entire open list and skips it `if edge.g > n.g` for
  *any* node `n` — not for the matching one. Cheaper paths to an already-queued
  tile are discarded almost every time.

- **The monster's death check looks at the wrong entity.** After a monster hits
  the player, `UpdateMonster` inspects `result.Components[health]` — the
  *monster's* health, which the monster's own attack can't have changed — to
  decide whether to clear a tile.

- **Input is level-triggered, not edge-triggered.** `TakePlayerAction` reads
  `ebiten.IsKeyPressed` and leans on the `TurnCounter > 8` gate to avoid
  repeats, so holding a key auto-fires at ~7 moves per second.
  `inpututil.IsKeyJustPressed` is the fix.

- **`GetRandomBetween` isn't inclusive.** It returns `GetDiceRoll(high-low) + low`,
  which spans `low+1 .. high` — so a weapon's `MinimumDamage` is never actually
  rolled, contrary to the doc comment.

- **`Blocked` means two different things.** It's set both for walls and for
  "a creature is standing here," which is why `IsOpaque` carries a
  `TODO: Change this to check for WALL, not blocked` and why movement code has
  to re-check `TileType` to tell a wall from an occupant.

- **The multi-level scaffolding is unused.** `GameMap.Dungeons` and
  `Dungeon.Levels` are modeled, but `NewGameMap` builds exactly one level and
  there are no stairs. Descending was the next thing on the list.

- **`BeforePlayerAction` is unreachable.** `UpdateMonster` assigns
  `game.Turn = PlayerTurn` directly instead of going through `GetNextState`, so
  the state machine only ever occupies two of its four states.

- **The HUD builds an image and a font it never uses.** `ProcessHUD` loads
  `hudImg` and `hudFont`, then draws with `userLogImg` and `mplusNormalFont` —
  the log system's copies.

- **Dead code in the dependency tree.** `mattn/go-sqlite3` is in `go.mod` and
  imported nowhere; the vendored `fonts/` package is unused because both UI
  systems import `github.com/RAshkettle/rrogue/fonts` — the tutorial's reference
  repo — instead of the local copy.

- **Game over just stops.** Reaching `GameOver` halts both the input and monster
  branches of `Update`. The log reads "Game Over!" and the window sits there;
  there's no death screen and no restart.

## Credits

The tutorial series at [fatoldyeti.com](https://www.fatoldyeti.com/) and its
reference implementation, [RAshkettle/rrogue](https://github.com/RAshkettle/rrogue).
