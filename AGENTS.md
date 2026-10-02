# Elementor v4 design system: agent guide

Use this when a developer reports a problem with Elementor v4 global classes or global variables. Typical symptoms:

- "Some classes are missing" in the editor after importing a design system
- A "Missing variable" chip on a colour or font control
- Styling gone or changed on elements after an import or a copy and paste between sites
- Classes or variables that look different on two sites that should match

## Read this first

Everything here comes from one test pair of sites on Elementor 4.2.x, plus a read of the Elementor 4.2.4 source. Treat it as a strong lead, not proof.

- Check each claim against the site in front of you and its Elementor version.
- Read any suggested fix in full and understand it before you run it.
- If a fix here disagrees with what you see on the site, trust the site and tell the user.

## Safety rules

1. Stay read-only until the user approves a change.
2. Take a database backup before any write, and try the fix on a staging copy first.
3. Never run commands against a remote or production server yourself. Give the user the commands to run.
4. Never assume the table prefix is `wp_`. Use `wp db prefix`, or `$wpdb->prefix` inside `wp eval`.
5. Scope every change to the exact IDs involved. No broad updates.

## What goes wrong

Elementor identifies each class and variable by an internal ID (for example `g-43fd75b` for a class, `e-gv-4204356` for a variable). This is not a CSS `id`. Elements store these IDs, not the names.

On import, Elementor matches classes and variables **by name** (case-insensitive). When the same name already exists on the destination site, the destination keeps its own internal ID. The source site's ID is never created there.

Anything that still points at the source site's ID is now pointing at nothing:

- an element copied from the source site
- another imported class that uses a colliding variable

Nothing shows an error. The styling just stops applying.

What happens to a same-named item depends on the choice in Elementor's import dialog:

| Dialog choice | What Elementor does with a class | What it does with a variable |
|---|---|---|
| `Replace existing values` | Replaces the whole class with the imported one. The destination's ID is kept. A setting that exists only on the destination is lost. | Updates only the value, and only if both variables are the same type. The destination's ID is kept. If the types differ, a new variable is added with a renamed label. |
| `Keep existing values` | Skips it. The destination class is untouched. | Skips it. The destination variable is untouched. |

Items with a name that does not exist on the destination are added and keep their imported ID. The exception is an ID the destination already uses for something else, which gets a fresh ID.

When Elementor reuses the destination's item, the source site's ID is never created on the destination. That covers both choices for classes, and both choices for variables of the same type. `Keep existing values` does not avoid the dangling reference problem.

Source for the table (Elementor 4.2.4): `modules/global-classes/import-export-utils/import-utils.php`, `modules/variables/import-export-customization/runners/import.php`, and the dialog code in `assets/js/packages/editor-design-system/`. The dialog sends `skip` for `Keep existing values` and `replace` for `Replace existing values`.

## Step 1: get the exports

Ask the user to export the design system from **each** site in the editor:

`Elementor > Design System > Export`

- To compare two sites, you need the source site export and the destination site export.
- Export the destination **fresh**, just before the check.
- For a single site, one export is enough.

Do not try to build the export yourself. The built-in WP-CLI export (`wp elementor kit export`) leaves out variables, so it is not a substitute.

## Step 2: run the differ

Open the differ: https://elementor-v4-design-diff.netlify.app (page hosted from `design-system-diff.html` in this repo). This guide is served at https://elementor-v4-design-diff.netlify.app/agents.md.

1. Drop the source export on the left and the destination export on the right.
2. Read the three warning types:
   - **Name clash:** same name, different internal ID. Anything pointing at the source ID will break.
   - **Indirect risk:** the class name is fine, but it uses a variable that clashes.
   - **Value differs:** same name, different values. Not an error. The user needs to decide which value is right before choosing `Replace existing values` or `Keep existing values`.
3. For a specific broken element, have the user select it in the editor, copy it (Ctrl/Cmd+C), then paste the clipboard text into the cross-check box, or save it as a `.json` file and upload it.
   - Copy it from the destination site. A copy from the source site carries the same IDs and tells you nothing extra.
   - The result lists which clash or value difference affects that element, and what each import choice would do to it.

Everything runs in the browser. Nothing is uploaded.

## Step 3: confirm on the site (read-only, optional)

If WP-CLI is available on a local copy, you can look at the stored data. Run these only on a local or staging copy, and verify the class and method names against the installed version.

Where the data lives:

| Data | Location |
|---|---|
| Variables | JSON in post meta `_elementor_global_variables` on the active kit post (option `elementor_active_kit` holds its ID) |
| Soft-deleted variable | Same entry, with a `deleted_at` key. It stays in the data, so existing references keep working. |
| Classes | One post per class, post type `e_global_class`. Class ID in meta `_elementor_global_class_id`. Styles in `_elementor_global_class_data` (PHP-serialized, so do not edit it with raw SQL). |
| Class order and labels | Meta on the kit post: `_elementor_global_classes_order` and `_elementor_global_classes_labels` |

Read the data with Elementor's own classes, not raw SQL:

```bash
wp eval '
$kit = \Elementor\Plugin::$instance->kits_manager->get_active_kit();
echo wp_json_encode( $kit->get_json_meta( "_elementor_global_variables" ), JSON_PRETTY_PRINT );
'
```

```bash
wp eval '
$kit = \Elementor\Plugin::$instance->kits_manager->get_active_kit();
$classes = \Elementor\Modules\GlobalClasses\Global_Classes_Repository::make( $kit )->all( true )->get();
echo wp_json_encode( $classes, JSON_PRETTY_PRINT );
'
```

The raw variable meta stores values in a wrapped format. The differ and the editor show the unwrapped values.

To find a dangling reference, list every variable ID used inside the class data (values starting `e-gv-`) and check each one exists in the variables data. An ID that is missing, or present only with a `deleted_at`, is the problem.

With a login or application password, the same data is available over REST: `GET /wp-json/elementor/v1/variables/list` and `GET /wp-json/elementor/v1/global-classes`. These are read-only.

## Step 4: fix it

Prefer editor fixes. They need no database access and they are what Elementor supports.

| Problem | Fix in the editor |
|---|---|
| A class uses a variable that clashes | Class Manager: edit the class and pick the variable again from its dropdown. One edit fixes every element using that class, on every page. |
| An element copied from the source site shows "Some classes are missing" | Open the element, remove the broken class chip, add the same-named class again from the picker. |
| A control shows "Missing variable" | Pick the variable again in that control. |
| Not broken yet, import still to come | Rename or delete the clashing item on the destination first. Renaming after the import does not help. |

Do not duplicate the broken element as a workaround. The copy carries the same dangling ID.

### Database fixes (last resort)

Use these only when many separate elements each carry their own broken local reference and the editor has no way to pick it again. These are suggestions from one test pair. Work out whether they apply before running anything.

- **Add a stand-in variable at the missing ID.** Create a variable entry under the dangling ID with the value you want. Existing references resolve again. Elementor has no screen for choosing an ID, so this means writing to the kit meta.
- **Repoint the stored reference.** Change the class or element data so it uses the destination site's real ID.

Whichever you use:

1. Back up first, and test on a staging copy.
2. Make the change through Elementor's PHP classes (for example the repository classes above), not raw SQL on serialized data.
3. Touch only the exact ID pair you identified.
4. Elementor writes compiled CSS to files. After the change, reload the editor or run Elementor > Tools > Regenerate CSS, or the old styling will keep showing.
5. Ask the user to confirm on the real page.

## Known limits

- Tested with colour variables and a handful of classes. Font and size variables are expected to behave the same way but are not tested.
- The mapping from dialog choice to `skip` / `replace` was read from the 4.2.4 source. It has not been tested on other versions.
- Elementor v4 is still changing quickly. Re-check the source if the version is newer than 4.2.4.
- `Keep existing values` skipping a same-named item is read from the source. The effect on already-pasted elements has not been tested.
