# /admin Investigator Tool

Part of [[Himawari-SMP]]. A staff (mod + owner) anti-cheat tool: a glowing book that opens a board of
players "under investigation", grouped by category, with one-click spectate-teleport.

## Flow
- `/admin` (gated `smp.mod`) gives the staffer the **Investigator** book — an `ENCHANTED_BOOK` marked
  `CUSTOM_DATA { survivalmod_investigator: true }` with a glint + light-purple name
  (`investigation/InvestigatorBook`, mirrors the `trial/TrialBooks` marker pattern).
- Right-click the book → `UseItemCallback` in `SurvivalMod.java` (same hook as the trial toggle)
  detects `InvestigatorBook.isInvestigator(...)` and opens `gui/InvestigationMenu`.
- `InvestigationMenu` — one icon per category (duping / combat / movement / xray / other, fixed in
  `InvestigationStore.CATEGORIES`), each showing its flagged count.
- `gui/InvestigationCategoryMenu` — the flagged players as **player heads** (reuses the `BalTopMenu`
  head pattern). Click a head → `setGameMode(SPECTATOR)` + `teleportTo` the target if online, else
  "<name> is offline".

## Managing the board
- `/admin investigate <player> <category> [reason]` — flag (validates the category).
- `/admin clear <name>` — remove all flags on a player (by name; works offline via the stored name).
- `/admin list` — print everyone flagged.

## Data
- `investigation/InvestigationStore` — in-memory list, **local `<world>/survivalmod_investigations.json`
  primary**, mirrored to the Supabase `investigations` table (service-key writes, anon read merged at
  boot). `SupabaseClient.fetch/insert/deleteInvestigationsByPlayer`.

## Related
[[moderation-bans]] · [[Himawari-SMP]] · [[shared-supabase]]
