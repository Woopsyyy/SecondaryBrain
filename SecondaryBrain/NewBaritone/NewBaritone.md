---
type: project-node
project: NewBaritone
stack: Fabric / Minecraft 26.2 / Java 25
status: active
updated: 2026-06-23
tags: [project/newbaritone, minecraft, fabric, baritone, pathfinding]
---

# NewBaritone

Fork of [cabaletta/baritone](https://github.com/cabaletta/baritone) — the "Google Maps for
Blockgame" pathfinding/automation client mod — maintained for the current **calendar-versioned
Minecraft (26.x line)** on **Fabric**. Repo lives in WSL at
`~/Project/Minecraft Bot/NewBaritone` (Windows: `\\wsl.localhost\Ubuntu\...`).

Sibling of [[Himawari-SMP]] in tooling: also a **Fabric MC 26.2 / Java 25 mod built from WSL** with a
`deployToMods` Gradle task that copies the jar into the local mods folder.

## Build & deploy

- **Always build from inside WSL** (`wsl.exe -e bash -lc '...'`). Running the Windows-side Gradle over
  the `\\wsl.localhost\` UNC path dies with `java.io.IOException: Incorrect function` (native file
  hasher fails over UNC).
- `gradlew` must have **LF line endings** (`.gitattributes` is `* text=auto`, so a Windows checkout
  gives it CRLF, which WSL bash rejects: `$'\r': command not found`). Fixed with `sed -i 's/\r$//' gradlew`.
- Toolchain: Fabric Loom 1.15.5, Gradle 9.4.0, JDK 25 (Temurin in `~/.gradle/jdks/...`). MC 26.x ships
  **unobfuscated / official-mapped**, so no intermediary mappings and no `remapJar` — the custom `jar`
  task assembles directly from the source sets, yet Loom still bundles `nether-pathfinder` as a JIJ
  nested jar (`META-INF/jars/nether-pathfinder-1.4.1.jar`).
- Build: `./gradlew jar` → `build/libs/baritone-<ver>.jar`. Deploy:
  `./gradlew deployToMods` → copies to `/mnt/c/Users/woopsy/AppData/Roaming/.minecraft/mods`
  (override with `-PmodsDir=...`). Local `.minecraft` runs MC 26.2 + Fabric API 0.152.x, sodium, iris,
  litematica/malilib, voicechat.
- `fabric.mod.json` `environment` is **`client`** (Baritone mixins target client-only classes).

## Inspecting unknown 26.x API (how the port was done)

Minecraft is unobfuscated, so the decompiled merged jar at
`~/.gradle/caches/fabric-loom/26.2/minecraft-merged.jar` is the source of truth. Use the Gradle JDK's
`javap`/`jar` (`~/.gradle/jdks/eclipse_adoptium-25-amd64-linux.2/bin`) to look up real signatures, e.g.
`javap -cp <merged.jar> net.minecraft.client.gui.Gui`.

## 26.1 → 26.2 API deltas fixed (2026-06-23)

- `net.minecraft.util.Tuple` **removed** → added drop-in `baritone.api.utils.Tuple` (`getA`/`getB`),
  swapped the import in 8 files.
- Colored blocks collapsed to **colour collections**: `Blocks.WHITE_SHULKER_BOX…` →
  `Blocks.SHULKER_BOX` + `Blocks.DYED_SHULKER_BOX.asList()`; `Blocks.WHITE_BED…` →
  `Blocks.BED.asList()` (`ColorCollection<Block>.asList()`). Affected `CachedChunk`.
- `Gui.getChat()` **removed** (ChatComponent no longer reachable from Gui) → log via
  `Minecraft.getInstance().gui.chatListener().handleSystemMessage(msg, false)` (loses the cosmetic
  Baritone `GuiMessageTag`).
- Toast: `Minecraft.getToastManager()` → `Minecraft.getInstance().gui.toastManager()`.
- Current screen moved off `Minecraft` onto **`Gui.screen()`** (`Minecraft.screen` field gone);
  `Minecraft.setScreen` → `Minecraft.setScreenAndShow`.
- `BlockPos.getCenter()` removed → `Vec3.atCenterOf(pos)` (elytra distance checks).
- `LevelRenderer.setBlocksDirty(int×6)` removed → `resetLevelRenderData()` (the `#render` command).
- Entity-type constants moved `EntityType.X` → **`EntityTypes.X`** (e.g. `FIREWORK_ROCKET`).
- `MixinMinecraft` `@Redirect` on `Minecraft.screen` GETFIELD no longer matches `tick()` → set
  `require = 0` so the mod loads cleanly (input-passthrough-while-pathing tweak goes inert).

## Rendering overlay — STUBBED on 26.2 ⚠️

26.2 removed **immediate-mode rendering** (`Tesselator`, `VertexFormat.Mode`,
`RenderType.draw(MeshData)`) for a low-level GPU-buffer system (`PreparedRenderType.drawFromBuffer(
GpuBuffer…)`). Baritone's path/goal/selection **visual overlay** (`baritone.utils.IRenderer`) was
rewritten as a **signature-preserving no-op** so everything compiles and runs — but the blue path
lines / goal box / selection boxes are **not drawn**. Pathing, mining, follow, build, elytra, the
danger system and all chat commands are unaffected. Full overlay port to the GpuBuffer pipeline is a
deferred follow-up.

## Danger memory (the "save world danger" feature)

`baritone.pathing.learning.DangerMemorySystem` already learns unsafe chunks (damage/death/lava/path
fails), feeds them into A* via `Favoring`, decays over time, and persists to
`baritone/danger_memory.json`. **Auto-save on world close already exists** —
`onWorldEvent` flushes to disk when the world unloads / on disconnect. Controlled by the `dangerMemory`
setting + `#dangermemory` command. (Currently one global file, not per-world.)

## Root-cause fix: bot frozen / "commands not working" (2026-06-23)

Symptom: mod loads, `#` commands parse and respond (`mine`/`goto`/`tunnel` print goals), **but the bot
never moves/mines** and there are **no errors in `latest.log`**.

Root cause: **the in-game `TickEvent` never fired.** `MixinMinecraft`'s PRE-tick `@Inject` (`runTick`)
anchored on `@At(FIELD, GETFIELD net/minecraft/client/Minecraft.screen)` inside `tick()` — but 26.2
moved `screen` off `Minecraft` onto `Gui`, so that instruction is gone and the injector matched nothing
(Fabric tolerated the miss silently — no crash, no log). With no PRE tick, `tickProvider` stayed null
and the POST-tick inject early-returned too, so **every tick-driven system was dead** (PathingBehavior,
PathExecutor, InputOverrideHandler, MineProcess, DangerMemorySystem) while synchronous command
responses still worked → looked like "commands don't work."

Fix: re-anchored `runTick` to `@At("HEAD")` of `tick()` (fires before the player ticks, so the
`PlayerMovementInput` override applies the same tick). Lesson: with `mixins.baritone.json`
`defaultRequire: 1`, injection-point misses **do not** reliably crash here — audit every injector's
`@At target` against the 26.2 jar, don't trust "it loaded".

## Feature work (post-port)

- **Look smoothing (deployed 2026-06-23):** `smoothLook` now defaults **true** and pitch is smoothed on
  the ground too (`LookBehavior` POST), so the head pans instead of jittering while mining/pathing. The
  jitter came from CLIENT-mode block-interaction + `randomLooking113=2.0` per-tick offset, with no
  averaging. `smoothLook` only affects the *visual* (functional aim is set in PRE), so this is safe.
  Tune with `#set smoothLookTicks <n>`.
- **malilib control GUI (v1 deployed 2026-06-23):** new package `baritone.gui.BaritoneGui extends
  fi.dy.masa.malilib.gui.GuiBase`; opened by the new `#gui` command (`GuiCommand`, registered in
  `DefaultCommands`). Tabs: Goto, Mine, Follow, Explore, Farm, Build, Sel, Waypoints, Control, Console.
  Each panel assembles a command string and runs `getCommandManager().execute(String)` — identical to
  chat, all chat commands still work. malilib added as `compileOnly files(...)` in `build.gradle`
  (provided at runtime by the user's installed malilib; `-PmalilibJar=` overrides the path), and
  `fabric.mod.json` now `depends` malilib `>=0.29.0`. **No hotkey yet** (v2): malilib hotkeys need an
  `IInitializationHandler` + `ClientModInitializer` entrypoint + keybind registry; deferred to keep v1
  robust. malilib API reference (0.29.0): `GuiBase.addButton(ButtonBase, IButtonActionListener)`,
  `addTextField(GuiTextFieldGeneric, ITextFieldListener)`, `addLabel(x,y,w,h,color,String...)`,
  `initGui()`/`clearElements()`; `ButtonGeneric(x,y,w,h,text,String...)`;
  `GuiTextFieldGeneric(x,y,w,h,Font)` + `getTextWrapper()/setTextWrapper()`.

- **GUI keybind + malilib entrypoint (v2 deployed 2026-06-23):** added a Fabric `client` entrypoint
  `baritone.gui.BaritoneFabricClient` (the only Fabric entrypoint; rest of mod is mixin-loaded) that
  registers `BaritoneMalilibInit` (`IInitializationHandler`) with malilib's
  `InitializationHandler.getInstance()`. `BaritoneHotkeys` (`IKeybindProvider`) exposes a `ConfigHotkey`
  `openBaritoneGui` (default **B**) whose callback opens `BaritoneGui`; registered via
  `InputEventHandler.getKeybindManager().registerKeybindProvider(...)`. Key is rebindable in malilib config.
- **Mining/chopping progress HUD (deployed 2026-06-23):** `BaritoneHud implements
  fi.dy.masa.malilib.interfaces.IRenderer`, registered with
  `RenderEventHandler.getInstance().registerInGameGuiRenderer(...)` in `BaritoneMalilibInit`. Draws a bar
  in `onExtractGuiOverlayPost(GuiContext, float, ProfilerFiller)` via `RenderUtils.drawRect(GuiContext,…)`
  + `renderText(GuiContext,…)` (malilib handles the 26.2 extract-then-render pipeline, so no raw
  GuiGraphicsExtractor needed). New `IMineProcess.getDesiredQuantity()`/`getMinedCount()` (impl in
  `MineProcess`, counts matching inventory items via the mine `filter`); gated by new setting
  `showMiningProgress` (default true).

## Known caveats / unverified

- Cannot launch MC in the build environment, so **runtime is unverified**: mixin apply-time matching
  against refactored 26.2 internals (beyond the ones explicitly hardened) may still need in-game
  iteration.
- Live `.minecraft/mods` has **duplicate fabric-api jars** (0.152.1 + 0.152.2) — pre-existing, see
  [[Himawari-SMP]] note.
- "New commands" from the original request were never specified; only the existing command set +
  `#dangermemory` ship today.
