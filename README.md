# NSFollowPlus 1.0.0

A complete Paper/Purpur plugin for player follows, privacy, alerts and a configurable leaderboard (Top 100 by default). The build targets **Java 17** and the **Paper 1.20 API**; run it on Paper/Purpur **1.20+** (newer Minecraft versions may require Java 21 to run the server). The ready-to-install JAR is `NSFollowPlus-1.0.0.jar` in this project folder (mirrored in `release/`); the source archive is `NSFollowPlus-Source.zip`.

## Install

1. Put `NSFollowPlus-1.0.0.jar` in the server's `plugins/` directory and restart the server.
2. The plugin creates `plugins/NSFollowPlus/config.yml`, `plugins/NSFollowPlus/menus.yml`, `plugins/NSFollowPlus/messages.yml`, and `plugins/NSFollowPlus/database/follow.db` (with any `-wal` / `-shm` sidecars in `database/`). The plugin root holds no database files.
3. Optional MySQL: create an empty database, set `storage.type: mysql` and the credentials in `config.yml`, then `/follow reload` or restart. **Changing backends does not migrate data.** Back up/migrate data yourself before pointing at a different database.

**Upgrading:** Stop the server and back up the entire plugin data folder, including SQLite journal files. With a previous NSFollowPlus installation, the plugin automatically moves the old root-level `follow.db`, `follow.db-wal` and `follow.db-shm` into `database/` before connecting; a pre-existing `storage.sqlite.file: follow.db` still works and now points inside `database/`. If source and destination both exist, startup stops rather than overwrite either database: resolve the conflict while the server is stopped. Custom SQLite filenames also remain under `database/`; values in `config.yml` are relative to the plugin directory (`database/follow.db`) or to the database directory (`follow.db`). Parent directories are created before opening SQLite. Paths escaping `database/` are rejected.

**Upgrading from the previous branding:** stop the server, remove the previous JAR (do not run both), and copy its `config.yml` and SQLite `follow.db` (plus any `follow.db-wal` / `follow.db-shm` files) into `plugins/NSFollowPlus/database/` before starting. If using MySQL, copy its `config.yml` so the new plugin points to the same database. The schema remains compatible; update permission grants to `nsfollowplus.use`. Starting with a fresh data folder instead creates an empty database.

```bash
mvn clean package
# Deploy target/NSFollowPlus-1.0.0.jar (also supplied in release/)
```

Maven packages HikariCP, SQLite JDBC (including its native binaries), and MySQL Connector/J inside the plugin JAR. Paper supplies its own API and SLF4J; no additional plugin dependencies are needed. Use a current Paper/Purpur release and keep the plugin's data folder backed up (including SQLite's WAL files during a live backup).

### MySQL example

```sql
CREATE DATABASE nsfollowplus CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'nsfollowplus'@'127.0.0.1' IDENTIFIED BY 'replace-with-a-strong-password';
GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, ALTER, INDEX, REFERENCES
  ON nsfollowplus.* TO 'nsfollowplus'@'127.0.0.1';
```

Set `storage.mysql.ssl: true` for a remote database. This uses `VERIFY_IDENTITY`: the certificate must match the hostname and its CA must be trusted by the JVM. The default `ssl: false` is intended for a trusted local connection only. Protect the config file because it contains the database password. MySQL 8+ is recommended; the repository's MySQL dialect was also exercised against a local MariaDB 11.8 instance.

## Commands

| Command | Behavior |
| --- | --- |
| `/follow` | Opens your **NSFollowPlus \| Overview** (27 slots). |
| `/follow <player>` | Follows that player if not already followed; if already following, reports it without changing the relationship. |
| `/follow profile [player]` | Opens your Overview or a target-focused, read-only Social Profile. |
| `/follow leaderboard` | Opens **Leaderboard \| (1/N)** when enabled. |
| `/follow toggle` | Toggles permission for others to follow you. |
| `/follow alert` | Toggles join/login alerts from players you follow. |
| `/follow help` | Shows every command and the `nsfollowplus.use` / `nsfollowplus.admin` permissions; console can use it. |
| `/follow reload` | Reloads all three YAML files, including live storage changes; requires `nsfollowplus.admin` (operators by default). Console can use it. |
| `/unfollow <following_player>` | Opens the existing **Unfollow Confirmation** GUI; Confirm alone performs the idempotent removal, Cancel does nothing. |
| `/followers [player]` | Opens your or the specified player's paginated Followers list. |
| `/following [player]` | Opens your or the specified player's paginated Following list. |
| `/followplus`, `/nsfollow` | Aliases of `/follow`, with the same arguments and completion. |

`nsfollowplus.use` grants player-facing commands (everyone by default); `nsfollowplus.admin` grants `/follow reload`. Tab completion for `/follow` includes the listed subcommands and online players **except yourself**; `/follow profile [player]`, `/followers [player]` and `/following [player]` complete online player names, including yours. `/unfollow <player>` completes **only players you currently follow**, from an asynchronously refreshed per-player database cache (warmed on join and invalidated after successful follow/unfollow). A cold/expired cache temporarily returns no names rather than unrelated players; press Tab again once its refresh completes. The cache is also cleared on storage reload. Command execution always verifies the follow edge from the database, independent of the tab cache. `/follow top`, `/follow toggle on|off`, and `/follow alert on|off` are invalid. Known offline players can still be targeted by command even though name suggestions list online players only; name resolution never performs a blocking Mojang lookup. Reserved words cannot be followed via the one-argument form. A private profile blocks **new** follows, not unfollowing.

## YAML customization and live reload

All player-facing text, items, materials, model data, glow, slots, filler ranges, menu sizes, head placement, click mappings, navigation and sound choices are read from YAML (not compiled Java strings/layouts). Defaults match the supplied GUI references; Following now requires confirmation before unfollowing. The files are saved in `plugins/NSFollowPlus/`:

| File | Controls |
| --- | --- |
| `config.yml` | SQLite/MySQL settings; `enable_leaderboard`, `leaderboard_cache_time` (seconds), `max_leaderboard_entries`, `play_sounds_on_click`, `allow_self_view_in_leaderboard`, sound/volume/pitch. |
| `menus.yml` | All five menus, including `unfollow_confirmation`: titles, sizes/rows, items, fillers, dynamic heads, content slots and navigation. |
| `messages.yml` | `%prefix%`, all chat/alert/error/usage/page-limit/reload text, pending-name separators, and optional per-message `ACTION_BAR` output. |

**Apply changes:** edit the files and run `/follow reload` or `/nsfollow reload` as an operator or user with `nsfollowplus.admin`. Console reload works without `nsfollowplus.use`; other players receive the YAML-configured denial. An invalid YAML file, bad material/action, overlapping head/button slots, or unavailable replacement database rejects the reload and retains the previous active configuration. Visible plugin inventories are reopened using the new layout/title. A confirmation already being cancelled or confirmed is returned to Following on reload rather than reopening a stale confirmation. Database initialization and connection changes run on the bounded database worker; switching SQLite/MySQL **does not transfer any existing follow data**. For migration, stop your server, back up the entire database (including WAL files), copy/migrate your data and only then change the storage target. Older custom files can omit new keys; missing keys inherit bundled defaults. Older `menus.yml` files are not overwritten. For the new border-only default, set `menus.followers`, `menus.following`, and `menus.leaderboard` → `fillers.gray_glass.slots` to `'0-8,9,17,18,26,27,35,36,44,45-47,51-53'`. The renderer keeps all `content_slots` AIR unless occupied by real player heads, even if an older/custom filler range still includes them; changing the on-disk range makes the YAML itself match the frame. If an older file still uses `GRAY_STAINED_GLASS_PANE`, change the filler `material` in those three menus **and** `unfollow_confirmation` to `BLACK_STAINED_GLASS_PANE`, and set each `name` to `'&0 '` (black-formatted invisible space), then run `/follow reload`. Confirmation remains fully black-filled except for its buttons and target head. Bukkit may treat a literal empty display name as absent and show the block's default name on hover. The filler key name is retained for backwards compatibility; fresh installations use black glass automatically. **Existing on-disk Overview privacy YAML is not overwritten:** a missing `states.mixed_material` inherits `YELLOW_DYE`; the previous bundled `states.disabled_material: PINK_DYE` is interpreted as `RED_DYE` at load so upgrades do not keep the old pink indicator. For a self-describing YAML file, copy the bundled `menus.overview.items.privacy` block (including `states.enabled_material: LIME_DYE`, `states.mixed_material: YELLOW_DYE`, `states.disabled_material: RED_DYE`, the status lore, and ordinary `LEFT`/`RIGHT: TOGGLE_PRIVACY`); remove any old `SHIFT_RIGHT` action and run `/follow reload`. A new `privacy-partial` message inherits the bundled default if it is absent from your existing `messages.yml`; other custom messages are retained. **If upgrading with an existing on-disk `messages.yml`, the exact previous bundled `usage` string is translated automatically. If you customized that line to mention the old syntax, replace it with the new bundled syntax (or delete the key to inherit it).** New `help`, `already-following`, `not-following` and shortcut-usage entries inherit bundled defaults when missing. `leaderboard.refresh-seconds` from old configs is still recognized until you explicitly set `leaderboard_cache_time`.

**GUI schema:** slots are zero-based. `size` is a multiple of 9 from 9–54; `rows: 1` through `6` overrides `size` when present. `fillers.<id>.slots` and `content_slots` accept `slots: [0, 1, 2]` or inclusive ranges such as `slots: '0-8,45-53'`. Multiple filler sections render in YAML order; later sections override earlier panes. On paged menus, fillers and loading decoration skip `content_slots`; only real player entries render heads there, leaving all unused content slots AIR. Items and navigation render on the remaining slots. A content/head slot cannot overlap a button. Move button slots and `content_slots` together when changing the geometry; the number of content slots is also the SQL page size, so pagination automatically follows your layout.

```yaml
# An additional Overview button (under menus.overview.items):
help_button:
  slot: 20
  material: LIME_DYE
  custom_model_data: 301     # optional
  glow: true                 # optional hidden enchantment shimmer
  play_sound: true           # gated by config.yml's global sound switch
  name: '<#55FFFF><bold>Friends</bold>'
  lore: ['&7Browse your &afollowers']
  actions:
    LEFT: OPEN_FOLLOWERS
    RIGHT: CLOSE

# A small paged menu can instead use, e.g.:
# menus.followers.rows: 4
# menus.followers.content_slots: '9-16,18-25'
# Move its existing navigation slots inside the new 36-slot inventory.
```

Built-in actions are `OPEN_FOLLOWERS`, `OPEN_FOLLOWING`, `OPEN_LEADERBOARD`, `REFRESH_OVERVIEW`, `TOGGLE_PRIVACY` (the four privacy combinations in one atomic cycle), `TOGGLE_ALLOW_FOLLOWERS`, `TOGGLE_ALERTS`, `CLOSE`, `PREVIOUS_PAGE`, `NEXT_PAGE`, `BACK_TO_MAIN`, `UNFOLLOW_TARGET`, `CONFIRM_UNFOLLOW`, and `CANCEL_UNFOLLOW`. The two independent toggle actions remain available only for legacy/custom bindings; the bundled privacy item maps ordinary `LEFT` and `RIGHT` to `TOGGLE_PRIVACY`. Bind actions by Bukkit click type (`LEFT`, `RIGHT`, `SHIFT_RIGHT`, etc.) or `ANY` as a fallback. Only mapped actions run; all inventory clicks/drags remain cancelled. `kind: VIEWER_PROFILE` and `kind: PRIVACY` apply to Overview items; on a target's Overview the profile slot shows the **target** head and the privacy dye is informational only (no action). Set `menus.overview.target_title` and the privacy item's `read_only_lore` to customize that view. `menus.following.content.read_only.lore` controls non-owner Following heads, which cannot open confirmation; the list `target_title` and `%back_destination%` customize navigation back to the target profile. `kind: TARGET_PROFILE` displays the selected target’s async-loaded head in `unfollow_confirmation`; privacy has configurable enabled/mixed/disabled dye materials and distinct status strings for each persisted setting. Dynamic profile slots always render `PLAYER_HEAD`, with the player's UUID and name assigned to `SkullMeta` immediately. Bundled `material` and `placeholder_material` are both `PLAYER_HEAD`; legacy custom placeholder materials are accepted for compatibility but ignored for profile slots. `content.rank_1`, `content.top_3` (ranks 2–3), and `content.rank_4_plus` customize leaderboard tiers independently. Navigation `previous` and `next` have `enabled` and `disabled` name/lore/material styles; set `visible: false` on an item, navigation button or filler to hide it. Optional `custom_model_data` and `glow` work on static items, initial skulls and resolved heads.

Formatting supports `&b`, `&l`, `&#55FFFF`, `<#55FFFF>`, `<bold>`, and other MiniMessage tags, including mixed legacy/MiniMessage text. GUI placeholders: `%player%` (viewer), `%target%` (head owner), `%rank%` (global leaderboard rank or list position), `%followers%`, `%following%`, `%status%`, `%page%`, `%max_pages%`, `%next_page%`, `%max_leaderboard_entries%`, `%allow_followers_status%`, `%follow_alerts_status%`, `%back_destination%` (Main Menu or Profile). Each default `messages.yml` entry uses `%prefix%`, preserving the previous `&b&lFOLLOW &8‣ ` prefix; older custom full-line messages continue to work. Message placeholders also include `%follower%`, `%count%`, and `%names%`. To move a notification to the action bar:

```yaml
follow-success:
  channel: ACTION_BAR   # or CHAT
  text: '%prefix%&aYou are now following &b%target%&a!'
```

Disabling the leaderboard blocks `/follow leaderboard` and the corresponding GUI action with a configurable message; hide its overview button using `menus.overview.items.leaderboard.visible: false` if desired. `allow_self_view_in_leaderboard: false` hides the viewer's own entry but preserves everyone else's original rank numbers. Changing `max_leaderboard_entries` adjusts both the SQL LIMIT and the default overview label. Sounds are globally disabled by `play_sounds_on_click: false`, with per-item `play_sound: true`; by default only successful privacy-item toggles play the configured `UI_BUTTON_CLICK`.

## Default menus

The GUI tooltip palette matches the reference screenshots using Minecraft's legacy equivalents: yellow `&e` = `#FFFF55`, gold `&6` = `#FFAA00`, aqua `&b` = `#55FFFF`, green `&a` = `#55FF55`, gray `&7` = `#AAAAAA`, dark gray `&8` = `#555555`, and red `&c` = `#FF5555`. The shared renderer supports legacy, hex and MiniMessage; the palette here describes the **default** `menus.yml`. The new yellow mixed and red disabled dyes intentionally supersede the older pink-off screenshot. JPEG capture and client resource packs can alter sampled pixel colors; the in-game color codes are authoritative.

- **Overview / Social Profile (27 slots):** head **4** (refresh), Heart of the Sea **11** (followers), Compass **13** (following), Nether Star **15** (Top 100), Lime/Yellow/Red Dye **22** (privacy). `/follow profile [player]` reuses this layout with the chosen account's UUID-owned skull, counts, privacy status, and follower/following lists; its privacy dye is read-only and has no toggle instruction. Lists for another player are read-only and their Back button returns to that target profile. Your own Overview retains the exact clickable privacy lore and behavior. Tooltip colors follow the supplied screenshots; the GUI brand label reads NSFollowPlus.
- **Leaderboard (54 slots):** ranked heads fill the screenshot's framed center in reading order, **28 per page**, capped at the top 100 (up to four pages). Every head has a yellow `Rank #N`, white username, dark-gray `Dynamic`, gold `Information:`, and aqua/gray labels with green follower/following counts.
- **Followers / Following (54 slots):** aqua member-name heads keyed by player UUID, **28 per page**, newest follows first. Followers show online/offline status and `Follower of <profile owner>`; your own Following shows a red `▸ Click to Unfollow` prompt. Clicking an **own** Following head opens the confirmation menu without changing the database. Another player's Following heads are informational and cannot trigger unfollow; Back returns to that player's profile. Status is sampled on the main thread when the menu renders; skins resolve asynchronously.
- **Unfollow Confirmation (27 slots):** `Unfollow | %target%`, black stained glass 0–26, **Lime Dye 11** to confirm, the selected target’s async-loaded skull **13** (informational only), and **Red Dye 15** to cancel. Confirm uses an idempotent asynchronous transaction, updates both follower/following lists and counters, sends the existing configurable `&b&lFOLLOW &8‣ &cYou are no longer following &b<name>&c.` message, and reopens Following on its original page (clamped after deletion). Cancel reopens your own Following page without changing the relationship. A stale confirmation for a relationship already removed reports `not-following` instead of claiming a second success. `/unfollow <player>` opens the same confirmation, returning to page 1 on Cancel/Confirm. By default, both actions use ordinary left/right clicks by the inventory’s owner; all other clicks and drags remain cancelled. The target UUID is bound to the menu holder, never inferred from editable item text. All three item slots, materials, names/lores, colors and the menu title are editable under `menus.unfollow_confirmation` in `menus.yml`.
- **Shared decoration/navigation:** black stained-glass panes with a blank-looking `&0 ` name and no lore occupy **only** border slots `0–8, 9, 17, 18, 26, 27, 35, 36, 44, 45–47, 51–53`. The 7×4 center slots **10–16, 19–25, 28–34, 37–43** show heads **only** for real player entries; every unused center slot stays **AIR**, including all 12 unoccupied slots on the final Top 100 page. No orange glass appears in these three menus. Arrow **48** = previous, Barrier **49** = back to `NSFollowPlus | Overview`, Arrow **50** = next; those three slots are navigation, not filler. Disabled controls have gray titles and red warning lore; enabled controls are green, and the back title is red.
- **Privacy Dye (slot 22):** the material reflects **both** persisted flags: `LIME_DYE` = followers and alerts enabled; `YELLOW_DYE` = exactly one enabled (either combination); `RED_DYE` = both disabled. The two lore lines always show the actual value of their respective flags, green `ENABLED` or red `DISABLED`. Ordinary **left or right click** cycles atomically through `(on,on) → (on,off) → (off,off) → (off,on) → (on,on)`; yellow appears twice with different lore. The hex-colored name is `★ PRIVACY SETTINGS`; the last lore line is `# Click to Toggle Settings!`. On successful database commit, the dye/lore update in-place, one configured click sound plays, and exactly one `FOLLOW ‣` feedback message is sent (the new `privacy-partial` text for either yellow state). A failed write leaves the item unchanged and plays no success sound. Shift-clicks, double-clicks and other variants are cancelled; rapid repeated clicks while a write is pending are ignored. `/follow toggle` changes **only** `allow_followers`; `/follow alert` changes **only** `follow_alerts` (the existing join-alert flag). Their successful commits immediately redraw any open Overview showing the changed account—including read-only target profiles—using the committed state, without closing/reopening the inventory.

While results are loading the menu can briefly show a loading item or placeholder page count; the populated inventory uses the specified title, slots and lore. Every rendered profile slot, including the Overview loading head and confirmation target, is a **PLAYER_HEAD from the start** with the player's name and UUID on its Paper profile. A texture is added asynchronously when available without changing the owner UUID/name, title, lore or (where applicable) click target. If Mojang has no skin for that identity or is unavailable, the item remains a player head with an unresolved texture until a later retry or menu reopen; the client may show a default skin. All inventory clicks (including shift/number-key/double-click) and drags are cancelled, and transitions are scheduled after the click event to avoid Bukkit inventory desync.

### Alerts

When a follow succeeds, an online target with alerts enabled is notified. For an offline target, a new-follower notice is kept in `pending_follow_alerts` and delivered once at their next join (up to five names appear in a summary). Unfollowing removes a still-pending notice; turning alerts off discards undelivered notices. When a followed player joins, their **online** followers with `follow_alerts` enabled receive a join message in batches of up to 80 recipients per tick. Join messages are local to the Paper server handling the join; a multi-server proxy needs an additional messaging bus for network-wide join broadcasts.

## Persistence and performance

`players` stores UUID, current username, normalized lookup name, `allow_followers` / `follow_alerts` (default true), counters and last join time. `follows` stores each `(follower_uuid, followed_uuid)` edge: querying it by either direction reconstructs the requested follower/following UUID lists without duplicating them in a serialized field. `pending_follow_alerts` holds offline notifications. Foreign keys and a unique edge key protect integrity; follows, counter updates and pending notices share one transaction. Sorting/indexes cover lookup, paginated lists and rankings.

- SQLite: all database and journal files live under `plugins/NSFollowPlus/database/`; the folder is created before JDBC connects. WAL, foreign keys, a 5-second busy timeout and a single pooled connection.
- MySQL: an InnoDB connection pool and sorted player-row locks for concurrent follow changes.
- All normal JDBC work uses a **bounded, single background queue**; only table creation on plugin startup is synchronous. A full queue fails safely instead of executing SQL on the server thread.
- The leaderboard is an immutable off-thread cache with configurable SQL limit and refresh seconds; it also updates after coalesced follow changes. GUI opens can request a fresh ranking.
- Heads use Paper's `Bukkit.createProfile(uuid, name)` to attach the owner identity to a `PLAYER_HEAD` on the main thread before skin resolution. `PlayerProfile.update()` fetches textures asynchronously, then the existing skull metadata is updated on the main thread. Online skin data is reused immediately. If a UUID is not a Mojang UUID (common in offline-mode servers), a name-only profile is tried as a fallback. Three workers limit concurrent lookups; each request has a five-second wait limit, one delayed retry, and a 512-entry UUID/name single-flight LRU (six-hour successful skins; 45-second misses). No blocking `Bukkit.getOfflinePlayer(...)` or network lookup occurs on the tick thread.

## Tests and verification

```bash
mvn clean verify  # SQLite integration tests + packaged JAR

# Optional: against an empty, disposable MySQL/MariaDB database:
export NSFOLLOWPLUS_TEST_MYSQL_HOST=127.0.0.1
export NSFOLLOWPLUS_TEST_MYSQL_PORT=3306
export NSFOLLOWPLUS_TEST_MYSQL_DATABASE=nsfollowplus_test
export NSFOLLOWPLUS_TEST_MYSQL_USER=nsfollowplus_test
export NSFOLLOWPLUS_TEST_MYSQL_PASSWORD=your-test-password
mvn test
```

The tests cover the configurable Unfollow Confirmation menu, UUID-bound targets, cancel/confirm action mapping, menu layouts/actions/fillers/statuses, sound/feature settings, page-size-aware SQLite pagination and configurable SQL leaderboard limits, all default screenshot colors, privacy click safety, MiniMessage/legacy colors, skin cache/UUID fallback/retries, SQLite migration, message fallback and optional MySQL transactions. Purpur 1.20.4 was also used to verify default and custom YAML rendering, permission checks, successful reload, atomic rejection of invalid layouts and live SQLite backend changes. A second in-server probe checked the confirmation layout, untouched database on cancel, same-tick click safety, configurable confirm slot, async transaction, and exact feedback. A further Purpur probe verified that Overview's loading head, Followers, Following, Leaderboard and confirmation all render `PLAYER_HEAD` with the correct UUID/name before textures resolve, even when an older configuration requests a non-head placeholder. A prior Purpur 1.20.4 in-server probe checked 1,084 inventory assertions: the three 54-slot menus show black panes only on the 23 border slots, AIR in all 28 empty content slots (including during loading), and navigation at 48–50. This also held with a legacy `0-53` filler and a custom loading item targeting slot 10. A populated Followers/Following slot immediately held a named, UUID-owned `PLAYER_HEAD`, with remaining center slots AIR; the 27-slot confirmation retained its black filler and buttons. The command refactor passed a further Purpur 1.20.4 probe with **54 checks** of registrations/aliases, help, rejected old syntax, idempotent follow-if-absent, cache-limited unfollow suggestions, target UUID-owned profile heads, read-only target privacy/Following heads, and Cancel/Confirm database safety. The new four-state release passed a separate Purpur 1.20.4 probe with **two actually connected players and 162 checks**: command-only updates to each flag, same-inventory live refresh in both the owner's and a read-only viewer's profiles, all four dye/lore/SQL states under LEFT and RIGHT clicks, rapid-repeat suppression, and read-only click safety. The client received exactly **four** `UI_BUTTON_CLICK` sound packets for four accepted dye clicks (none for command toggles). In-client visual inspection and tab completion with actual online players remain in `MANUAL_QA.md`.
