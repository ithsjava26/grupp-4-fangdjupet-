# Fångdjupet

![CI](https://github.com/ithsjava26/grupp-4-fangdjupet-/actions/workflows/ci.yml/badge.svg)

A text-based role-playing game for the console. The hero fights through five rooms of a
dungeon - Goblin, Skeleton, Goblin, Mage and finally a Troll boss - finds better equipment
on the way and can save the game between battles.

Group project in the course Javaprogrammering (ITHS), built by three people with feature
branches, pull requests and code review.

## How to run

**Requirements:** JDK 26 and Maven (or IntelliJ IDEA, which includes Maven).

**IntelliJ IDEA**
1. Open the project folder (the one with `pom.xml`).
2. Make sure the project SDK is Java 26 (*File → Project Structure → SDK*).
3. Open `src/main/java/se/fangdjupet/Main.java` and press the green ▶ next to `main`.

**Terminal**
```bash
mvn compile
java -cp target/classes se.fangdjupet.Main
```

**Run the tests**
```bash
mvn test
```
The tests also run automatically with GitHub Actions on every pull request and push to `master`.

## How to play

```
Fangdjupet
==========
1. Start new adventure
2. Continue saved adventure
3. Show hero's equipment
4. Fight next enemy
5. Save game
e. Exit
```

- In battle you choose **1. Attack** or **2. Drink a potion** every turn. A potion uses your turn.
- Your attack style comes from your weapon: sword/axe/dagger = melee, bow = ranged (+20 % damage),
  staff = magic (ignores armor).
- After each room you find an item. Better weapons and armor are equipped automatically.
- The game is saved to `savegame.txt` in the project folder (plain text, `key=value`).

## Modules and responsibilities

| Package | Responsible | Contents |
|---|---|---|
| `combat` | **Johan** | `Hero`, `Enemy`, `AbstractCombatant`, `Battle`, `AttackStrategy` with `MeleeAttack` / `RangedAttack` / `MagicAttack`, `DamageCalculator` |
| `items` | **Abbe** | `Item` hierarchy (`Weapon`, `Armor`, `Potion`), `Inventory`, enums `ItemType`, `WeaponType`, `ArmorSlot` |
| `factory` | **Abbe** | `ItemFactory`, `ItemCatalog`, `EnemyFactory`, `EnemyType` |
| `core` | **Oskar** | Shared event system: `EventBus`, `GameEvent`, `EventType`, `GameEventListener` |
| `game` | **Oskar** | `Game` (menu and battle loop), `World`, `GameState`, `Dungeon`, `ConsoleUi`, `MenuOption`, `ConsoleEventLogger`, `EventHistory` |
| `game.save` | **Oskar** | `SaveManager`, `SaveGameException`, `ItemSerializer`, `SaveData` |

Oskar was also project manager: planning, code review of every pull request and the CI pipeline.

### How the packages depend on each other

```
game  ──►  combat, items, factory, core      (game connects everything)
factory ─► combat, items                     (creates enemies and items)
combat ──► core                              (battles publish events)
items                                        (depends on nothing)
```
`combat` and `items` never know about each other. `GameState` in `game` takes the damage
from the weapon and the defence from the armor and passes the numbers to `Battle`.

## Design patterns

### Strategy - attack types (`combat`)
`AttackStrategy` has three implementations: `MeleeAttack`, `RangedAttack` (×1.2 damage) and
`MagicAttack` (ignores defence). `Battle` only calls `strategy.calculateDamage(...)` and never
needs to know which attack it is. The hero's strategy is chosen from the equipped weapon
(`GameState.getAttackStrategy()`), the enemy's from its type.
**Why:** a new attack type can be added without changing `Battle`.

### Factory - items and enemies (`factory`)
`ItemFactory` creates weapons, armor and potions; `EnemyFactory.createEnemy(EnemyType, level)`
creates enemies from a type and a level (+25 % health and damage per level).
`ItemCatalog` builds ready-made items like *Rusty Sword* through the factory.
**Why:** all creation happens in one place. The validation stays in the constructors, so the
rules are the same no matter how an object is created. `SaveManager` also uses `ItemFactory`
when it loads items from the save file.

### Observer - event log (`core` + `game`)
`Battle` and `Game` publish events (`ENEMY_DEFEATED`, `ITEM_FOUND`, …) to the `EventBus`.
Two listeners react: `ConsoleEventLogger` prints them (`[BATTLE] Goblin was defeated.`) and
`EventHistory` remembers them (used for *Enemies defeated: 5* at the end).
**Why:** the code that makes things happen does not need to know how they are shown.
Adding a new listener, e.g. a log file, needs no change in `Battle`.

## Tests

182 JUnit 5 tests. The central logic of every module is tested:

- **Combat:** damage calculation, defence, attack strategies, battle flow and events
  (`DamageCalculatorTest`, `BattleTest`, `HeroTest`, `EnemyTest`, …)
- **Items:** inventory handling, capacity, filtering with generics, factories
  (`InventoryTest`, `ItemFactoryTest`, `EnemyFactoryTest`, `ItemCatalogTest`)
- **Save/load:** round trips, equipment, Swedish locale, broken or edited files
  (`SaveManagerTest`, `ItemSerializerTest`, using `@TempDir`)
- **Game:** `GameTest` plays the game with scripted input, including a full run through
  all five rooms to victory.

## Git workflow

- Every member worked in their own feature branches (`feature/combat-core`, `feature/inventory-core`,
  `feature/save-load`, …) and opened pull requests: Oskar 8, Abbe 7 and Johan 4.
- Every pull request was reviewed by another member before it was merged with a merge commit,
  so the history shows who did what.
- A GitHub Actions pipeline (`.github/workflows/ci.yml`) runs all tests on every pull request.

## Group reflection

### What worked well
- **A shared contract first.** We started with the package structure and a short plan in each
  `package-info`, and the event system (`core`) was the first real pull request. After that we
  could work in parallel without waiting for each other.
- **Code review found real bugs.** For example: `Inventory.getItems()` returned the internal list
  so the capacity could be bypassed, `removeItem()` disappeared during a refactor, the first armor
  values blocked almost all damage, and the three attack strategies first returned exactly the same
  damage. All of it was fixed before merging.
- **Small, focused pull requests** with a clear description made reviews fast and easy to follow.

### Conflicts and how we solved them
- **A conflict Git could not see.** When Abbe changed the `Weapon` and `Armor` API (`getDamage()`
  became `getTotalDamage()` and the constructors changed), GitHub showed *no conflict*, because we
  had changed different files. But `GameState` on `master` no longer compiled together with his
  branch. We found it by building the branches together, Abbe merged `master` into his branch, and
  we fixed `GameState` and its tests together in a live coding session before merging.
  After that we added the CI pipeline, so problems like this show up automatically.
- **API agreements between modules.** The game loop needed enemies with their own attack. We agreed
  on the exact constructor `Enemy(name, maxHealth, attackDamage, attackStrategy)` in writing before
  Johan and Abbe started, so `EnemyFactory` and `Enemy` fitted together without breaking anything.
- **Language.** We started with Swedish comments in some packages and agreed to write all code,
  comments and commit messages in English.

### What we learned / would do differently
- Agree on method names and constructors between modules in writing *before* coding.
- Set up CI on day one instead of halfway through the project.
- Use `git status` before every commit, so each commit contains exactly what its message says.