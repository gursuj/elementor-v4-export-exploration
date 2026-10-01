# Elementor v4 design-system export/import — reliability test

**Context for anyone (or any agent) picking this up:** Evaluating whether Elementor v4's design-system export/import (classes + variables, `Elementor > Design System > Export/Import`) is safe to recommend for migrating between real client sites. Two local test sites: **staging** (source) and **test-v4** (destination), both on Elementor 4.2.4, `WP_DEBUG`/`WP_DEBUG_LOG`/`WP_DEBUG_DISPLAY` all on.

## The bug

Copying an element that uses a global class/variable to a site that already has a class/variable **of the same label** silently breaks the copied element's styling — no error in the editor, browser console, or `debug.log`.

**Root cause:** the exported class/element JSON hardcodes the *source* site's variable/class id, baked in at export time (see `design-system-export/global-classes/g-43fd75b.json`). On import, if a class/variable of the same **label** already exists on the destination, Elementor matches by label, updates its *value*, but keeps the **destination's own id**. It never rewrites the id references already baked into other imported classes or already-pasted elements. Those keep pointing at the source's id, which was never created on the destination — dangling reference, style fails silently.

Only happens on **label collisions**. A class/variable with a unique label imports fine and keeps the source's id unchanged (confirmed with a throwaway `newcol` variable + unique class — copy + import worked, either order).

## Confirmed findings

- **Order doesn't matter.** Copy-then-import and import-then-copy both break on a collision, both work fine without one. A paste always bakes in the literal id at copy time; import never retroactively rewrites already-pasted data.
- **Classes collide the same way as variables.** Same-name class on both sites (different styles, no variables involved) → import correctly fixes the destination's *own* pre-existing class + widget, but a widget pasted from source afterward still references the source's class id, which was never created on destination (import merged into the destination's own id instead). Editor shows "some classes are missing." Repro element: `imported-class-only` on test-v4 post 8.
- **Transitive risk:** a class can have a unique label (no collision on the class itself) but still break, if one of its style props references a *variable* that collides.
- **Two independent live fixes proved the mechanism** (see `notes.md` history / git log for the exact DB edits): (a) insert a stub variable at the dangling id, (b) repoint the class's stored prop to the destination's real id. Either restores styling. Elementor caches compiled CSS to files (`wp-content/uploads/elementor/css/*.css`) — a DB fix alone won't show until the editor is reloaded (or Tools → Regenerate CSS), the frontend doesn't auto-regenerate those on its own.
- **Save does not silently strip a dangling reference** — confirmed via DB before/after. Safe to diagnose an already-broken element any time, no rush.
- **Duplicate propagates the problem instead of fixing it** — the new copy inherits the same dangling id. Also noticed: Duplicate didn't uniquify `_cssid` (both elements ended up sharing `imported-class-only`).
- **Variable delete is a soft delete** — tombstoned (`deleted_at`), but the id→value mapping stays in the kit's `_elementor_global_variables` data, so existing references keep resolving. No equivalent safety net exists on the import path — that's exactly why the collision case breaks.
- **Ids aren't visible in the DOM or compiled CSS** — both use labels only (`class="class1"`, `var(--primary)`). Only visible via the editor's own REST API while logged in: `GET /wp-json/elementor/v1/global-classes`, `GET /wp-json/elementor/v1/variables/list`.
- Collision evidence is **permanent**, not a pre-import-only signal — diffing two exports shows the same mismatched ids whether done before or after an import ran. So checking reactively (after someone hits a bug) is just as valid as checking ahead of time.

## The tool — `design-system-diff.html`

Single local HTML file, no backend, no sign-in — open directly in a browser. Uses JSZip (from cdnjs) client-side.

- **Diff mode:** drop two design-system export `.zip`s (source + destination). Parses `global-variables.json` + `global-classes/*.json`, diffs by label. Flags collisions and transitive risk. Opens with the real staging/test-v4 collision preloaded as example data.
- **Cross-check mode:** paste one broken element's copied JSON (Ctrl/Cmd+C on it in the editor) — only the *destination's* copy is needed, a source-side copy carries identical ids and is on a working site, so it adds nothing. Tool matches the element's class/variable ids against whatever the diff already flagged, and tells you which specific collision is hitting it.
- Every warning has a **Solution** button → dialog with a DB-free fix. All solution text lives in one `SOLUTIONS` object near the top of the script, keyed by warning type (`var-collide` / `class-collide` / `class-risk`) — edit there to update guidance.

## Recommendation to the team

1. **Prevent:** before a real migration, run the diff tool against both sites' current exports. Rename or delete anything flagged as colliding **on the destination**, before importing — renaming after the fact doesn't help, it doesn't touch the id or already-pasted references.
2. **Already broken:** don't duplicate the element as a workaround. Copy its JSON, run it through the tool's cross-check, then fix via whichever it points to:
   - Class-level collision → one edit in Class Manager (reselect the variable from its picker) fixes every element using that class, even across many pages.
   - Element-level stale class → remove the broken class chip on that element, re-add the same-named class from the picker.
   - Local (non-class) variable reference → reselect the variable in that control directly.
   - **DB edit is a last resort** — only when many individually-pasted elements each carry their own local (non-centralized) broken reference, or no live UI equivalent exists to pick. Back up first, scope the query to the exact known id pair, test on staging before touching a live site.
3. Beyond this specific bug: Elementor v4 core is still beta-quality generally (two other bugs seen on unrelated live sites: global classes vanishing on refresh, and an order-based visibility bug). Not yet safe to broadly recommend for client work; safest on a genuinely empty destination, or with the workflow above.

## Repo layout

- `design-system-diff.html` — the collision-checker tool
- `design-system-export/` — real export pulled from the staging site (source), used as the tool's example data and to confirm the export file format
- `images/` — screenshots from the original test run
- `testv4_post8_before_save.txt` / `testv4_post8_after_save.txt` / `save_diff.txt` — DB dumps proving Save/Duplicate don't strip a dangling reference

## Open TODOs

- [ ] Search for other reported design-system export/import bugs online; test any viable ones, flag to team
- [ ] Check whether the two live-site issues noted above (migration gate, order-based visibility bug) also affect test-v4
- [ ] File a GitHub issue for this collision bug (`elementor/elementor`, Editor V4 bug template) — evidence is all in this doc + `design-system-export/`
- [ ] Confirm typography/font variables collide the same way as color variables (only color tested so far)
- [ ] Confirm "keep existing" vs "replace" on an import conflict doesn't change the id-collision outcome (only "replace" tested)
