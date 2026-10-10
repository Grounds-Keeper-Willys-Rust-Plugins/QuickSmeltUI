# QuickSmeltUI

A full-screen admin console for **Quick Smelt 5.1.15** by misticos and WhiteThunder. Quick
Smelt has no commands and no UI at all, so every setting has to be hand-edited in JSON, using
furnace short *prefab* names that are easy to confuse with item short names. This plugin turns
that config into a browsable UI with real furnace and item icons.

Built and verified against the live server on 2026-09-03. Rust protocol 2632.

Companion to `RecycleManagerUI`, and built on the same pattern. See
`README.RecycleManagerUI.md` for the shared design rationale.

---

## 1. Scope

| | |
|---|---|
| Plugin file | `oxide/plugins/QuickSmeltUI.cs` |
| Target plugin | `QuickSmelt` (Quick Smelt 5.1.15) |
| Target config | `oxide/config/QuickSmelt.json` |
| Own config | none, by design |
| Permission | `quicksmelt.admin` |

QuickSmeltUI never duplicates Quick Smelt's smelting logic. It reads and writes that plugin's
JSON config, then reloads it.

---

## 2. Requirements

### R1. Access control

- **R1.1** Every entry point is gated on `quicksmelt.admin`.
- **R1.2** Quick Smelt registers only `quicksmelt.use`, so this plugin registers
  `quicksmelt.admin` itself, and only if it does not already exist.
- **R1.3** Oxide logs one warning at load because the permission is not prefixed with this
  plugin's own name. That is deliberate: the permission name was specified, and it keeps the
  two Quick Smelt permissions in one namespace.
- **R1.4** Both the chat command and every console command re-check the permission per call.

### R2. Entry points

- **R2.1** `/quicksmeltui` in chat opens the window.
- **R2.2** `quicksmeltui open` and `quicksmeltui close` from the client console.
- **R2.3** All in-window controls issue console commands under the single `quicksmeltui` namespace,
  because a CUI button command always runs as a console command.

### R3. Edit model

- **R3.1** Opening reads `QuickSmelt.json` into a working copy and records its modification
  time.
- **R3.2** Edits stage in memory. Nothing touches disk until **Save and apply**.
- **R3.3** An **UNSAVED** badge shows in the title bar while the working copy differs.
- **R3.4** **Save and apply** writes the file, runs `oxide.reload QuickSmelt`, and four
  seconds later reports whether the plugin came back.
- **R3.5** **Revert** discards the working copy and re-reads from disk.
- **R3.6** If the file changed on disk since it was loaded, the first save is refused and the
  button becomes **Overwrite and apply**.
- **R3.7** Missing sections are created in memory on load, so a hand-trimmed config cannot
  crash the UI.
- **R3.8** **Fuel Usage Multipliers** and **Smelting Frequencies** deserialize into
  `Dictionary<string, int>` on the Quick Smelt side. Values written to those two maps are
  therefore rounded to whole numbers. Writing `2.7` produces `3`. Without this, a fractional
  value would throw inside Quick Smelt's `LoadConfig`, which catches the exception and
  silently falls back to *default* configuration, discarding every setting the admin had.
- **R3.9** The UI warns that reloading rebuilds every furnace controller, so lit furnaces
  stop and restart once.

### R4. The scope model

Every one of Quick Smelt's seven per-furnace maps is keyed by a **furnace short prefab
name**, for example `electricfurnace.deployed`, or by the literal `global`. Lookup is exact
furnace name first, then `global`, then a built-in default of 1.

- **R4.1** A scope is listed if it appears as a key in any of the seven maps.
- **R4.2** `global` is always listed first and is shown as **All furnaces**.
- **R4.3** Scopes are labelled by the display name of the deployable item that places them,
  not by the raw prefab name, with the prefab name shown underneath.
- **R4.4** A scope whose key is not a real oven on this server is drawn in red. Quick Smelt
  warns about these at load; this makes them visible before that.
- **R4.5** Adding a furnace seeds only its speed entry. Every other map stays absent so the
  furnace keeps inheriting from `global`.
- **R4.6** Removing a furnace strips it from all seven maps, behind a two-step confirm.
- **R4.7** `global` cannot be removed.

### R5. Tabs

| Tab | Contents |
|---|---|
| **General** | Require-permission and debug switches, how saving works, content counts, placeholder cleanup |
| **Furnaces** | Grid of every configured scope, with the furnace icon and a rate summary. Opens the furnace editor |
| **Rates** | One table of every scope against all four numeric settings, editable inline |
| **Output** | Scope selector, then the output multipliers for that scope |
| **Allow and block** | Scope selector, then the allow list and block list side by side |

### R6. Furnace editor

Opens from a card on the Furnaces tab or a row on the Rates tab.

- **R6.1** Shows the furnace icon, display name and short prefab name.
- **R6.2** Four numeric rows: smelt speed, fuel burn speed, fuel consumed, smelt frequency.
  Each carries a one-line explanation of what it actually does.
- **R6.3** A value inherited from `global` is drawn dim and labelled **inherited**. Typing
  into it creates an entry for this furnace.
- **R6.4** A value set explicitly gets an **Inherit** button that deletes the key so the
  furnace falls back to `global` again.
- **R6.5** Fields that must be whole numbers are labelled as such.
- **R6.6** Smelt frequency is flagged in amber as read but never used. See finding F1.
- **R6.7** Three buttons jump to that scope's output multipliers, allow list and block list,
  each showing its current entry count.
- **R6.8** **Remove this furnace** requires a second confirming click.

### R7. Rates tab

- **R7.1** One row per scope, ten rows per page.
- **R7.2** Four inline numeric fields per row, dim when inherited.
- **R7.3** Clicking anywhere on a row that is not a field opens the furnace editor.
- **R7.4** `global` is tinted so it reads as the fallback row.

### R8. Output multipliers

- **R8.1** Scales the stack size a smelt produces, and also the byproduct a fuel item leaves
  behind, which is how charcoal yield is tuned.
- **R8.2** The scope selector shows up to twelve scopes as buttons. More than that, and the
  UI says so and points at the Furnaces tab.
- **R8.3** Entries render as two columns of rows, each with the item icon, name, an editable
  multiplier and a remove button, eight rows per column.
- **R8.4** The inner key `global` is shown as **Every other item** with an accent tag rather
  than a broken icon, because it is a real and useful key, not an item.
- **R8.5** Keys that are not real items are drawn in red rather than hidden.
- **R8.6** **+ Add item** opens the item picker filtered to items that smelting can produce.

### R9. Allow and block lists

- **R9.1** Both lists are shown side by side for the selected scope.
- **R9.2** The UI states the precedence Quick Smelt actually implements: the block list wins,
  and an allowed item smelts even below its normal temperature.
- **R9.3** Rows show the item icon, name and a remove button, ten per list.
- **R9.4** **+ Add item** opens the item picker filtered to smeltable items.
- **R9.5** Unknown and placeholder entries are drawn in red.

### R10. Pickers

- **R10.1** The **furnace picker** lists every `BaseOven` prefab in the game manifest, six by
  three per page, searchable by display name and prefab name. Furnaces already configured are
  tinted and labelled. On this server it finds 32 ovens, 20 of which have a deployable item
  and therefore an icon.
- **R10.2** The **item picker** searches all 1252 item definitions by short name and display
  name, ten by four per page.
- **R10.3** The relevance toggle changes meaning with the target. For output multipliers it
  filters to items smelting can produce, 24 of them. For the allow and block lists it filters
  to items that can be smelted, 30 of them. It can be switched off.

### R11. Placeholder cleanup

Quick Smelt ships a default config containing the literal keys `furnace.shortname` and
`item.shortname` as documentation. They configure nothing.

- **R11.1** The General tab counts them and offers **Remove placeholders**.
- **R11.2** They are drawn in red wherever they appear, labelled as placeholders.
- **R11.3** On the stock config this removes twelve entries.

### R12. Rendering constraints

- **R12.1** The window is one `AddUI` console command. Rust drops console-command packets
  above `server.maxpacketsize_command`, default **100000 bytes**.
- **R12.2** Measured worst case is the Rates tab at fifteen scopes, **157 elements, 47995
  bytes**. The item picker is 135 elements, 39975 bytes. Remeasured after the command rename,
  which lengthened every button string.
- **R12.3** Grid cards are a single button element carrying children. Icons and labels are
  marked `BlocksRaycast = false` so the card takes the click.
- **R12.4** When a modal is open the tab behind it is not drawn.
- **R12.5** Paging bounds every list, which is what keeps R12.1 satisfiable.
- **R12.6** Layout uses anchor `0 0` with pixel offsets relative to the parent's bottom-left.
  Floats are formatted invariant, so a non-English server locale cannot emit `1,5`.

### R13. Lifecycle

- **R13.1** Unloading destroys the window for every connected player.
- **R13.2** Disconnecting drops that player's state.
- **R13.3** Item and furnace catalogs are built once at `OnServerInitialized`. The furnace
  scan walks the whole game manifest and took 5 ms.

---

## 3. Console command reference

```
quicksmeltui open
quicksmeltui close
quicksmeltui tab General|Furnaces|Rates|Output|Lists
quicksmeltui toggle useperm|debug
quicksmeltui scope <furnaceShortName|global>
quicksmeltui goto <tab> <scope>
quicksmeltui fadd
quicksmeltui fopen <scope>
quicksmeltui fclose
quicksmeltui fset <scope> Speed|FuelSpeed|FuelUsage|Frequency <value>
quicksmeltui fclear <scope> <field>
quicksmeltui fdelarm
quicksmeltui fdel <scope>
quicksmeltui omadd <scope>
quicksmeltui omset <scope> <item> <value>
quicksmeltui omdel <scope> <item>
quicksmeltui wladd <scope>
quicksmeltui wldel <scope> <item>
quicksmeltui bladd <scope>
quicksmeltui bldel <scope> <item>
quicksmeltui pksearch <text>
quicksmeltui pkpage <n>
quicksmeltui pkrel
quicksmeltui pkpick <name>
quicksmeltui pkcancel
quicksmeltui fpage|rpage|opage|lpage <n>
quicksmeltui cleanup
quicksmeltui save | saveforce | revert
```

---

## 4. Verification performed

Driven live against the running server by reflection, with the config backed up and restored.
Confirmed:

- All five tabs render without exception.
- Placeholder cleanup removed all twelve stock entries.
- Adding `electricfurnace.deployed` seeded only its speed entry.
- Fractional input to an integer field rounded correctly. `2.7` was written as `3`, and `1.4`
  as `1`, so Quick Smelt's `Dictionary<string, int>` deserialization stays valid.
- Float fields kept their precision. `4.5` and `2.25` round-tripped exactly.
- Nested output multiplier map created for a new scope.
- Allow and block list entries added and written.
- Save produced Quick Smelt's exact schema and the plugin reloaded with no validation
  warnings, meaning every furnace and item short name written was valid.
- **Inherit** removed a key and collapsed the scope when it was the last one.
- The two-step delete guard refused an unarmed removal and honoured an armed one.
- Payload measured on every screen against the 100000-byte limit.
- After renaming the command to `quicksmeltui`, `find quicksmeltui` in the server console
  lists `global.quicksmeltui` and `find qsui` lists nothing. Every screen and dialog was
  driven again and payloads remeasured.
- After restoring the backup, the config is byte-identical to its pre-test contents.

---

## 5. Findings in Quick Smelt, for later work

These are behaviours of `QuickSmelt.cs` itself, not of this UI.

**F1. Smelting Frequencies does nothing.** `_smeltingFrequency` is read from the config at
`QuickSmelt.cs:449` and never used anywhere else in the plugin. The config key names itself
"Smelt items every N smelting ticks", and an admin who sets it will see no effect and no
warning. Either the feature was removed and the setting left behind, or it was never wired
up. This UI flags the field in amber. Fixing it upstream means dividing the cook interval, or
skipping ticks in `Cook`.

**F2. Speed multipliers below 1 misbehave.** In `SmeltItems`, `amountConsumed` is
`(int)((1 + floor(...)) * _speedMultiplier)`. A multiplier under 1 truncates that to zero, so
no input is consumed while `ItemManager.Create` is still called with a zero amount. The same
truncation hits fuel: `fuelToConsume = (int)(_fuelUsageMultiplier * _speedMultiplier)` becomes
zero, and `fuel.UseItem(0)` consumes nothing, which is effectively infinite fuel. Quick Smelt
is only safe at multipliers of 1 or above. Nothing in the plugin or its config says so.

**F3. The plugin has no commands and no permission for administration.** Before this UI the
only way to change anything was to edit JSON on disk and reload by hand. `quicksmelt.use` is
the only permission it registers.

**F4. Furnace keys are prefab names, not item names.** `electric.furnace` is the item;
`electricfurnace.deployed` is the key the config wants. Quick Smelt does detect this and
suggests the right name in its warning, which is good, but the warning only appears in the
server log at load. This UI removes the guess entirely by listing real ovens with icons.

**F5. `Unload` can throw.** It calls `oven.GetComponent<FurnaceController>()` and then uses
the result without a null check. Every oven should have the component, but an oven that
spawned during the load window would not, and the exception would abort the rest of the
unload loop, leaving other furnaces with stale controllers.

**F6. `OnOvenStart` checks only the owner.** `OnOvenToggle` accepts either the furnace owner
or the acting player having `quicksmelt.use`, but `OnOvenStart`, which handles igniters and
electrical starts, checks the owner alone. The same furnace is therefore boosted or not
depending on how it was lit.

**F7. Output multipliers are applied with an integer cast after multiplication.**
`(int)(cookable.amountOfBecome * amountConsumed * OutputMultiplier(...))` truncates, so a
multiplier of 1.5 on a single-unit smelt yields 1, not 2. Fractional multipliers only behave
predictably on larger batches.

**F8. The default config ships placeholder keys.** `furnace.shortname` and `item.shortname`
appear in all seven maps. They are skipped by validation and do nothing, but they look like
real configuration. This UI offers to remove them.

**F9. A malformed config silently reverts to defaults.** `LoadConfig` catches everything,
prints one line, and calls `LoadDefaultConfig`. The file on disk is left alone, so the server
runs on defaults while the config still reads as customised. This is what R3.8 exists to
avoid triggering.

---

## 6. Planned work for this UI

1. **Live apply without a reload.** Quick Smelt exposes no API, so every save currently
   restarts lit furnaces. A hook that rebuilt the controllers in place would remove that.
2. **Warn on multipliers below 1**, given F2, rather than accepting them silently.
3. **A per-furnace preview**, computing the effective smelt time and fuel burn for a chosen
   item so the numbers can be sanity-checked before saving.
4. **Copy settings between furnaces**, since tuning several ovens the same way is common.
5. **Show which furnaces are actually placed on the map**, and how many, next to each scope.
6. **A read-only viewer permission**, so a moderator can inspect without editing.
7. **Bulk add**, applying a rate to several furnaces in one pass.
8. **A diff preview on save**, listing exactly which keys change before the file is written.
9. **Scroll views instead of paging**, gated on a payload budget check so R12.1 still holds.
