# Execution plan: configurable column anchor + output gravity

Local working document. **Not for upstream** (`CONTRIBUTING.md` forbids LLM-authored
content in PRs). The human-written feature request is in
`feature-request-column-fill-direction.md`.

## Implementation status: done (phases 1-4)

All four phases below are implemented, building, clippy-clean, rustfmt-clean, and passing
`cargo test -p niri -p niri-config -p niri-ipc` (206/19/1/2 tests) including the
`RUN_SLOW_TESTS=1` randomized proptests. Notable deviations from the plan as originally
written, found while implementing:

- **View-offset change is scoped to the bootstrap case only**, not threaded generally through
  `compute_new_view_offset_for_column`. Tracing the call sites showed `_fit` is already
  edge-agnostic (it picks whichever edge needs less motion) for ordinary navigation and
  resize — the "flush left" look of today's default column is an artifact of the empty-
  workspace view-offset reset, not of `_fit` itself. So only that one reset (in
  `ScrollingSpace::add_column`, the `was_empty` branch) needed to become anchor-aware; normal
  navigation/resize keep using the same `_fit`/`_centered` logic as before, unmodified. This is
  smaller than "generalize `compute_new_view_offset_centered` into `_aligned`" and avoids
  changing behavior nothing asked for (e.g. forcing every focus change to hug an edge).
- **`activate_prev_column_on_removal` needed a real fix, not just a config knob.** Its existing
  doc comment assumed columns are *only ever* created to the right of the active one; a
  right-facing anchor breaks that assumption (new columns land to the *left*). Left
  unaddressed, this was a latent `usize` underflow panic (`self.active_column_idx - 1` at
  `active_column_idx == 0`) the first time someone opened-then-immediately-closed a column
  under a right/toward-center-resolved-right anchor. Fixed by widening the field from
  `Option<f64>` to `Option<(bool, f64)>` (which side "previous" is on, plus the offset) and
  updating the removal-side consumer and the one other producer (`consume_or_expel_window_left`,
  which is inherently leftward regardless of `column-anchor` and so always stores `true`).
- **IPC `OutputGravity` dropped `radius`/`angle`.** They're one-liners
  (`dx.hypot(dy)`/`dy.atan2(dx)`) for any consumer, and including them pushed
  `niri_ipc::Output` past clippy's `large_enum_variant` threshold (it's embedded in
  `dbus::mutter_screen_cast::StreamTarget::Output`, unrelated to this feature). Kept `dx`/`dy`
  only; documented the derivation in the struct's doc comment. The local (non-IPC)
  `crate::utils::OutputGravity` still exposes `.radius()`/`.angle()` methods for in-process use.
- Everything else — the `column-anchor` semantics, the four config values, resolution
  precedence (`always-center-single-column`/`center-focused-column "always"` win unconditionally
  by sitting behind the same `is_centering_focused_column()` early-return), the Cartesian
  `dx`/`dy` gravity representation, and the phase boundaries — matches the plan as written.

## 1. Existing infrastructure

### Config
- `niri-config/src/layout.rs` — `Layout` (resolved) + `LayoutPart` (knuffel-decoded,
  all-optional) + `impl MergeWith<LayoutPart> for Layout`. Adding a field here gets
  per-output and per-workspace overrides **for free**:
  - per-output: `niri-config/src/output.rs:80` `layout: Option<LayoutPart>`
  - per-workspace: `niri-config/src/workspace.rs:19` `WorkspaceLayoutPart(pub LayoutPart)`
- `CenterFocusedColumn` (`layout.rs:162`) is the exact precedent for a new scalar enum:
  `knuffel::DecodeScalar`, variants kebab-cased in KDL.
- Resolution chain: `Options::from_config` → `with_merged_layout(output part)` →
  `with_merged_layout(workspace part)` → `adjusted_for_scale` (`src/layout/mod.rs:664-689`).
  Per-monitor `Options` are already rebuilt on config reload and on layout-config change.

### Layout / scrolling
- **Insertion index chokepoint**: `ScrollingSpace::add_column`
  (`src/layout/scrolling.rs:999`). With `idx == None` it picks
  `if was_empty { 0 } else { self.active_column_idx + 1 }`. That `+ 1` *is* the current
  hardcoded "grows rightward" rule. All `Workspace::add_tile` / `add_column` paths funnel
  here with `None` (`src/layout/workspace.rs:664,714,763`).
- **View alignment**: `compute_new_view_offset_fit` / `_centered` / `_for_column*`
  (`scrolling.rs:580-713`) plus the free `compute_new_view_offset` (`scrolling.rs:5588`).
  Only ~5 functions and ~8 call sites — a tractable surface.
  `compute_new_view_offset_centered` (`:628`) returns `-(area.size.w - width)/2. - area.loc.x`;
  a right anchor is the direct analogue `-(area.size.w - width) - area.loc.x`.
  The first column's left-flush position comes from the `-padding` branch of
  `compute_new_view_offset` (`:5616`); the right-anchored result is literally the other
  branch (`:5618`).
- `is_centering_focused_column()` (`:575`) is the precedent for "an option that changes
  which view-offset computation is used", and is also the early-return override that makes
  the `always-center-single-column` precedence rule (§2) free to implement.
- `verify_invariants` (`scrolling.rs:3790`) asserts nothing about absolute view offset —
  no invariant blocks a different anchor.

### Monitor positions — the architectural gap
- `Layout`/`Monitor`/`Workspace` are deliberately **position-unaware**. Explicit
  confirmation at `src/layout/mod.rs:3956`:
  `// FIXME: when and if the layout code knows about monitor positions, ...`
- Global positions live in `Niri::global_space` (`src/niri.rs:242`), assigned in
  `Niri::reposition_outputs` (`src/niri.rs:2834`), which is the single place outputs are
  laid out. It is called on config reload (`:1932`), output add (`:3038`) and output
  remove (`:3062`).
- `reposition_outputs` also does `output.change_current_state(.., Some(new_position))`,
  so `output.current_location()` works (used by `logical_output`, `src/utils/mod.rs:209`).
  Do **not** exploit that inside `src/layout/` — it would silently break the
  position-unaware design and the fake-output layout tests. Push the value in instead
  (same pattern as `layout_config`).

### IPC
- `niri_ipc::Output` (`niri-ipc/src/lib.rs:1210`) with `logical: Option<LogicalOutput>`
  (`:1258`, x/y/width/height/scale/transform).
- Built in `State::refresh_ipc_outputs` (`src/niri.rs:2062`) via `logical_output`.
- Pretty-printed by `niri msg outputs` in `src/ipc/client.rs`; schema behind the
  `json-schema` feature (`niri-ipc/Cargo.toml:22`).

### Tests / docs obligations
- `src/layout/tests.rs:3999` `arbitrary_layout_part()` — every new `LayoutPart` field must
  be generated here (CONTRIBUTING requires randomized-test coverage).
- `docs/wiki/Configuration:-Layout.md` — new options need a section with a
  `<sup>Since: XX.YY</sup>` annotation (see `:133` `always-center-single-column`).
  Cross-reference from `Configuration:-Outputs.md` is already generic.
- `resources/default-config.kdl` — commented example if warranted.
- `niri-config/tests/wiki-parses.rs` parses the wiki KDL snippets, so doc examples must
  be valid config.

## 2. Semantics

**One config key, four values.** Earlier drafts had a separate growth-direction option;
it was dropped because growth is not independent: content anchored to an edge can only
grow away from that edge, so the anchor already determines where the next column goes.
Growth is only a free choice when the anchor is the monitor *centre*, and that case is
deliberately deferred (§6).

```kdl
layout {
    column-anchor "left"  // "right" | "toward-center" | "away-from-center"
}
```

- `left` — current behaviour. First column at the left edge, new columns to the right.
- `right` — first column at the right edge, new columns to the left.
- `toward-center` — anchored to whichever monitor edge faces the centre of the monitor
  field; resolved per monitor.
- `away-from-center` — the mirror of `toward-center`.

Resolved per monitor into an internal pair:

| config value | monitor left of field centre | monitor right of field centre |
|---|---|---|
| `left` (default) | anchor Start, grow Right | anchor Start, grow Right |
| `right` | anchor End, grow Left | anchor End, grow Left |
| `toward-center` | anchor End, grow Left | anchor Start, grow Right |
| `away-from-center` | anchor Start, grow Right | anchor End, grow Left |

where `grow Right` = insert at `active_column_idx + 1` (today's behaviour) and
`grow Left` = insert at `active_column_idx`. Note `grow` is fully determined by `anchor`
in this table — it is kept as a separate internal value only because the two feed
different code paths (insertion index vs. view offset), not because they can disagree.

**Tie-break:** `dx == 0` (single monitor, or a monitor exactly centred in the field)
resolves as "left of field centre", so a single-monitor setup gives
`toward-center` ≡ `right` and `away-from-center` ≡ `left`, deterministically.

**Precedence (decided):** `always-center-single-column` always wins over the anchor.
`center-focused-column "always"` likewise. Both are explicit user intent about focus
behaviour, and both already early-return in `is_centering_focused_column()`
(`scrolling.rs:575`), so the anchor logic simply sits behind that check. `grow` is
unaffected by either. Document the interaction in the wiki.

## 3. Output gravity representation

Both representations: Cartesian drives the implementation, polar is the queryable/IPC
form for the user's future work.

Field = bounding box of all mapped outputs' logical geometry. Centre = bbox centre
(equivalently `(x_min + x_max) / 2`).

```rust
// normalised to the field half-extent, so -1.0 ..= 1.0 on each axis
struct OutputGravity { dx: f64, dy: f64 }
// polar, derived: radius = hypot(dx, dy), angle = atan2(dy, dx)
```

Degenerate case: a single output (or zero-width field) → `dx = dy = 0.0`.
Only `sign(dx)` is consumed by this feature; `dy`/polar exist for future work.

## 4. Phased implementation

Each phase is a self-contained, building, test-passing commit (CONTRIBUTING §
"Writing pull requests").

### Phase 1 — config surface, no behaviour
- `niri-config/src/layout.rs`: add
  ```rust
  #[derive(knuffel::DecodeScalar, Debug, Default, PartialEq, Eq, Clone, Copy)]
  pub enum ColumnAnchor { #[default] Left, Right, TowardCenter, AwayFromCenter }
  ```
  plus `pub column_anchor: ColumnAnchor` on `Layout` (default `Left`),
  `Option<ColumnAnchor>` on `LayoutPart` (`#[knuffel(child, unwrap(argument))]`), and the
  field in the `merge_clone!` list in `merge_with`.
- Re-export from `niri-config/src/lib.rs` if the other layout enums are re-exported there.
- `src/layout/tests.rs`: add an `arbitrary_column_anchor()` strategy and wire it into
  `arbitrary_layout_part()`.
- Wiki section + `Since:` annotation; `resources/default-config.kdl` comment.
- Verify: `cargo build`, `cargo test -p niri-config`.

### Phase 2 — static `left` / `right` behaviour in `ScrollingSpace`
The field-relative values resolve to `Left` for now (explicit `TODO(phase 3)`).
- Add `FillAnchor { Start, End }` and `GrowDirection { Left, Right }` in
  `src/layout/scrolling.rs` (or `src/layout/mod.rs` if `Monitor` needs them too), plus a
  `fn resolved_anchor(&self) -> (FillAnchor, GrowDirection)` on `ScrollingSpace` reading
  `self.options.layout.column_anchor`.
  (No `Center` variant — `always-center-single-column` / `center-focused-column` handle
  centring and take precedence anyway.)
- `add_column` (`:1008-1014`): `active_column_idx + 1` becomes
  `match grow { Right => active + 1, Left => active }`. Check the two follow-on branches
  that depend on `idx` vs `active_column_idx` (`:1026`, `:1033`) still behave — inserting
  *at* `active_column_idx` shifts the active index, which is the existing, already-handled
  path for `idx <= active_column_idx`.
- Generalize `compute_new_view_offset_centered` into `_aligned(anchor, ..)`:
  `Start` → existing `_fit`, `End` → `-(area.size.w - width) - area.loc.x`. Keep the
  existing centred formula for the `is_centering_focused_column()` path and keep the
  "column wider than view falls back to `_fit`" guard.
- Route `compute_new_view_offset_for_column` (`:655`) through the anchor, behind the
  existing `is_centering_focused_column()` early return.
- Decide and document whether the `End` anchor also mirrors the tie-break inside the free
  `compute_new_view_offset` (`:5612-5619`), i.e. whether scroll-to-reveal prefers the
  right edge under a right anchor. Recommended: yes — symmetric behaviour is the point of
  the option — but keep it a separate, clearly-labelled hunk so it can be reverted alone.
- Tests: new cases in `src/layout/tests.rs` (first column position, second column side,
  focus movement, removal) + run the proptests with `RUN_SLOW_TESTS=1`.
- Manual: `cargo run` nested, both values, with 1/2/3 columns, fullscreen, maximized,
  tabbed columns, overview, and interactive window drag.

### Phase 3 — output gravity plumbing
- New `OutputGravity` type. Location: `src/utils/mod.rs` next to `logical_output`, or a
  small `src/output_gravity.rs`; it must be visible to both `src/niri.rs` and
  `src/layout/`.
- Compute in `Niri::reposition_outputs` (`src/niri.rs:2834`) after the mapping loop:
  bbox over `self.global_space.output_geometry(..)` for all mapped outputs, then per
  output `dx`, `dy`.
- Push into the layout: `Layout::update_output_gravity(&Output, OutputGravity)` →
  `Monitor` stores it → include it in the `Options` rebuild so `ScrollingSpace` sees it.
  Mirror the existing `update_layout_config` plumbing
  (`src/layout/monitor.rs:1214`, `src/niri.rs:1914`) — including its
  "returns true if changed" shape, to avoid needless option rebuilds.
- Resolve `toward-center` / `away-from-center` in `resolved_anchor()` per the table in §2.
- Confirm gravity is refreshed on output **resize/mode change** too, not only add/remove/
  config reload (`Niri::output_resized`, `src/niri.rs:1923`), since the bbox moves.
- Tests: layout tests construct monitors with explicit gravity values (no fake positions
  needed); add a unit test for the bbox → `dx`/`dy` maths including the single-output and
  identical-position degenerate cases.

### Phase 4 — IPC exposure
- `niri-ipc/src/lib.rs`: add `gravity: Option<OutputGravity>` to `Output` with
  `dx`, `dy`, `radius`, `angle` documented (angle in radians, CCW from +x).
  Keep `niri-ipc` free of compositor deps — define the IPC struct there and convert.
- Populate in `State::refresh_ipc_outputs` (`src/niri.rs:2062`).
- Print in `niri msg outputs` (`src/ipc/client.rs`) and regenerate/verify the JSON schema.
- Note: this is an additive field, so existing IPC consumers are unaffected.

## 5. Risks / things to watch
- DnD insert-hint view offsets (`scrolling.rs:2571-2578`) use the centered/fit helpers
  directly and will inherit anchor changes — verify the hint still lands correctly.
- `view_offset_to_restore` (`scrolling.rs:3816`) and the view-offset gesture paths.
- Moving a window between monitors whose resolved anchor differs (interactive move,
  `move-column-to-monitor-*`): the target workspace re-computes its own view offset, but
  check the animation does not jump.
- Overview rendering scales workspaces; confirm the anchor does not fight the overview
  zoom transform.
- `empty-workspace-above-first` is vertical — unaffected.

## 6. Deliberately deferred
- **A centre anchor** (first column centred on the monitor). Only meaningful together
  with a separate growth-direction option, since centred content can grow either way.
  Adds a config key whose sole purpose is one anchor value. If it is ever wanted, it comes
  back additively as a fifth `column-anchor` value plus a `column-grow` key documented as
  applying only to it.
- **Symmetric / zigzag growth** (alternating left-right around a centred anchor). Same
  dependency on a centre anchor, and it makes column order unpredictable relative to
  insertion order, which fights the scrollable-strip model (`Mod+Left/Right` navigation
  order, `move-column-left/right`, `consume-or-expel`).
- **A vertical axis** (`dy` is computed and exposed but unused). A `column-anchor-y`
  equivalent has no consumer in the scrolling layout today.
