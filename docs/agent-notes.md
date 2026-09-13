# Agent notes: architecture in detail

The full reasoning behind the rules in `CLAUDE.md`. Moved here verbatim on
2026-09-13 to keep `CLAUDE.md` short. Read the section for the module you are
about to change before changing it.

### Architecture

Src-layout package at `src/maskingframe/`. Dependency direction is explicit and one-way: `geometry` and `layout` are leaves (neither imports the other, or anything else in the package); `compose` uses both `geometry` and `layout`; `pipeline` uses all three (`geometry`, `layout`, `compose`); `cli` and `gui/` use only `pipeline`, never `geometry`, `layout`, or `compose` directly.

- `geometry.py` — pure image transforms. Takes and returns `PIL.Image` objects, never touches the filesystem. Owns `AspectRatio` (the target-shape dataclass, with a `label` like "Portrait" and a `display` property like "Portrait (4:5)"), the `RATIOS` registry and `DEFAULT_RATIO`, `section_count()` (how many detail frames a panorama gets, floored at 2), `frame_width()`, `position_travel()`, `clamp_position()`, `normalise_positions()` and `default_positions()` — the position model, `make_padded_frame()`, and `make_section()` (takes a position, a ratio, and a style). It also owns the border model: `FrameStyle` (a frozen dataclass of `border_percent`, `border_colour`, `gutter_percent`, `gutter_colour` and `border_detail_frames`, with `border_px(ratio)` and `gutter_px(ratio)` to resolve a percent into output pixels), `DEFAULT_STYLE`, `MAX_PERCENT` (40.0) and `parse_colour()`. There is one colour parser, so a colour is normalised to lowercase `#rrggbb` at the boundary and can never reach PIL malformed. The style is always a parameter with a default, never module state, so a preview and the run it previews cannot disagree about the border. `RATIOS` is insertion-ordered Portrait, Square, Landscape (narrowest to widest) — that is the presentation order everywhere; never `sorted()` it, since alphabetical order puts `1.91:1` first and reads as noise. Fast to test in memory.
- `layout.py` — pure arithmetic, no PIL and no I/O. Solves composite arrangements by expressing each node's width as an affine function of its height, then scoring candidates on frame fill. Candidates are generated rather than listed: an arrangement is one root — a row or a column — whose parts are consecutive blocks of the images in input order, a block of one being a leaf and a block of more a group of the *opposite* orientation. That alternation is what makes the set canonical, since without it `R(R(1,2),3)` and `R(1,2,3)` would both be generated for the same picture. It gives 2, 6, 14, 30 and 62 candidates for two through six panels, and `name_of()` names each one from its tree as `R(1,C(2,3))`, numbered from one to match the numerals the sources list draws.

One level of grouping is a restriction, not a consequence. Deeper trees exist — 394 at six panels rather than 62 — and they fill the frame better, by up to 13 points; they are left out because the arrangements they win with, `C(R(C(R(1,C(2,3)),4),5),6)` and its like, are not ones anybody would lay out. Fill score stops being a good proxy for a good layout past three panels. The two- and three-panel sets are unaffected, because every arrangement the old hand-written list held already had one level.

`solve()` takes every candidate within `TIE_TOLERANCE` (1e-9) of the best score and prefers the shallower tree, then the earlier name — both properties of the arrangement, so generation order decides nothing. Ties are the common case: frames from one camera share an aspect ratio. The depth term turned out to be a guard rather than an active rule — no tie has been found whose members differ in depth, since arrangements that group differently assemble to different shapes — and `test_no_tie_has_ever_been_found_between_two_depths` fails the day one appears, so that claim is checked rather than remembered.

`rank()` returns every placeable arrangement best first, and `solve()` is its head, so the
arrangement chosen automatically and the list a user overrides it from are ordered by one
rule rather than two. The tie rule is that one sort key — `(-round(score, TIE_DECIMALS), depth, name)`
— where rounding at `TIE_DECIMALS` (9) expresses the same fact as `TIE_TOLERANCE` (1e-9).

An arrangement has two spellings and `short_name()`/`parse_name()` own them:
`R(C(1,2,3,4),C(5,6))` and `R4.2`. The short one exists because the long one is a syntax
error unquoted in both zsh and bash, so copy-pasting what the tool prints would fail with
a shell error rather than a message from the tool; it says exactly as much, because one
level of grouping means an arrangement *is* an axis plus its block sizes. `parse_name`
matches against the generated list rather than parsing, so a second definition of what a
name means cannot drift from `name_of` and `short_name`, and it never raises — the caller
decides what an unknown name means.

Never crops, never permutes input order. `solve()` and `evaluate()` take a `FrameStyle`, and a solved `Layout` reports `gutters` — the exact rectangles between adjacent panels — alongside `boxes`, so the renderer can paint them a second colour without redoing the arithmetic. Each gutter rectangle is inflated by a pixel at both ends along the gutter axis, because a panel's rounded edge can land a pixel off the rounded gutter edge; the cross axis is not inflated, since bleeding there would put gutter colour into the outer border.
- `compose.py` — PIL in, PIL out, like `geometry.py`. Three passes in this order: the whole exact-size canvas takes the border colour, the gutter rectangles take the gutter colour, then the panels land on top. That order is what lets the gutters be inflated without ever showing. Refuses a box whose aspect disagrees with its image rather than distorting it.
- `pipeline.py` — the only module that touches the filesystem. Re-exports `AspectRatio`, `RATIOS`, `DEFAULT_RATIO`, `FrameStyle`, `DEFAULT_STYLE`, `parse_colour` and `MAX_PERCENT` from `geometry` specifically so `cli.py` and `gui/` can offer ratio and border selection without importing `geometry` directly — that preserves the stated dependency direction. Do not "simplify" these re-exports away; that would force `cli`/`gui` to import `geometry` directly and break the invariant. `pipeline.py` also owns the output-filename contract (`{prefix}_1_padded.jpg`, `{prefix}_2_section1.jpg`, `{prefix}_3_section2.jpg`, ... one entry per detail frame) via `output_paths()`, plus `process_image()`, `find_panoramas()`, and `process_folder()`. `process_folder()` reports per-file failures in the returned `BatchResult` instead of raising, so one bad image doesn't abort a batch; callers (CLI, GUI) decide how to surface them. It also owns `compose_images()`, which solves a layout via `layout.solve()` and renders it via `compose.render()`, writing `{prefix}_diptych.jpg`, `_triptych`, `_tetraptych`, `_pentaptych` or `_hexaptych` and returning a `CompositeResult(path, layout_name)`. `COMPOSITE_SUFFIXES` is held by a test to exactly `MIN_IMAGES..MAX_IMAGES`, so raising the ceiling again cannot leave a count with nowhere to write to; both bounds are re-exported from `layout` for the same reason the ratio names are. `name_layout`, `compose_preview` and `compose_images` each take an `arrangement` name ahead of `style`, empty meaning the solver chooses; a name that does not parse, or that names an arrangement these sources cannot have, is a `ValueError` rather than a fallback, because silently composing something else would write a picture nobody asked for. Its message states the rule and one example rather than listing every name — at six sources there are 62, and 600 characters of names is a wall to read rather than an answer. `arrangements()` lists every candidate best first in both spellings, reading headers only, so offering the choice costs no decodes. Unlike `process_image()`, `compose_images()` accepts portrait input — mixing orientations is the point. Six entry points take a `FrameStyle` (`process_image`, `preview_frames`, `name_layout`, `compose_preview`, `compose_images`, `process_folder`), and in each one `style` is the last parameter with a default, so a new option goes in front of it rather than displacing it. That is a rule about where `style` lives, not a promise that nothing moves: adding `positions` ahead of it did break positional calls, and `cli.py` had to start spelling `style=style`. It also re-exports the position model (`default_positions`, `normalise_positions`, `move_position`, `insert_position`, `drop_position`, `frame_width`, `position_travel`) for the same reason, and `ribbon_thumbnail()` decodes the small copy of the panorama the ribbon draws — separate from `cached_preview_source()`, which holds a much larger copy for cutting frames from.
- `cli.py` — argparse entry points (`maskingframe`, `maskingframe-gui`). Exposes `--ratio` (default `4:5`/Portrait) via a `type=` converter, not `choices=`, so it accepts both the bare ratio (`4:5`) and the name (`portrait`), case-insensitively; an unknown value is rejected with a message naming every accepted spelling. `--help` lists them in presentation order as `portrait|4:5, square|1:1, landscape|1.91:1`. It also exposes `--border`, `--border-colour`, `--gutter`, `--gutter-colour` and `--border-detail-frames`, assembled into a `FrameStyle` by `_style_from_args()`. `--border-color` and `--gutter-color` are accepted as aliases so nobody has to guess the spelling, but the British form is the one documented. Widths and colours are validated by `type=` converters at parse time, so a typo fails with argparse's own message instead of at render time. There is a `compose` subcommand, and there is deliberately no `split` one: splitting stays at the top level so `maskingframe pano.jpg out` keeps working exactly as it always has, which also rules out argparse subparsers (the `input` positional would compete with the command name). `main()` therefore dispatches on the literal first word — if it is `compose`, the rest goes to `build_compose_parser()`, otherwise to `build_parser()`. The one cost is that a file named exactly `compose` in the working directory is shadowed and has to be spelled `./compose`. `_add_style_arguments()` attaches the ratio and framing flags to both parsers from one definition, so the two commands can't drift apart. Compose takes its sources as `nargs="+"` and checks the count afterwards, because argparse can't express "2 or 3" without mis-assigning arguments, and the error then names how many were actually given. Its output is `-o`/`--output` (default `output`), not a trailing positional, which after a variable-length list would be ambiguous. It prints the winning layout name from `CompositeResult` — the automatic arrangement is the interesting part — and, unlike the split path, applies no landscape check. Never exits on import; the PySide6-availability guard lives in `gui_main()`, not at module scope, so importing the package can never terminate the host process. Imports `run` from `gui`, the package, not from any specific submodule inside it.
- `gui/` — Qt (PySide6) front end. `theme.py` (the palette and one stylesheet; see "The theme" below), `shell.py` (the skeleton both tabs share: `RebateBand`, `TwoColumn`, the `section`/`help_label`/`data_label` helpers, `PathRow`, two hand-drawn widgets — `Combo`, because flattening a combobox takes Qt's themed arrow with it, and `TabBand`, because `QTabWidget` centres its bar on macOS — plus `Swatch`, a flat block of colour that opens the picker, and `BorderControls`, the border section both tabs show in the same place), `settings.py` (the only module that touches `QSettings`), `work.py` (running slow work off the GUI thread — read it before adding any), `app.py` (`MainWindow` and `run()`), `split_tab.py`, `compose_tab.py`, `strip.py` (`ContactStrip`), `ribbon.py` (`FrameRibbon`, the whole panorama with a draggable window per detail frame) and `sources.py` (`SourcesList`). `gui/__init__.py` re-exports `run` and `MainWindow`; `cli.gui_main` imports `run` from the package, so that name must keep working. Only `pipeline` is imported from outside the package, never `geometry`, `layout` or `compose` directly.

Both tabs expose `subject`, `detail` and a `band_changed` signal to the shell, and nothing else. The band belongs to the shell rather than to either tab, so the tabs never know about the band or about each other.

### How the rail is laid out

The rail is a `QScrollArea` with a pinned foot. The settings scroll; the
commitment does not — a primary action below the fold is the one convention this
rail cannot break, and pinning it also stops it moving as the rail grows.
`TwoColumn` exposes `rail_layout` for the scrolling body and `rail_foot_layout`
for the pinned part.

The line used to be drawn at the primary button, and that was one row too low.
The reason for the foot is that what commits to disk must not go below the fold,
and **the destination is part of that commitment rather than a setting**: at
1100x700 the Split rail scrolled DESTINATION away entirely while leaving a live
Cut frames button pinned above it, which is one click from writing files to a
path you cannot see. So both tabs now put everything from the DESTINATION
heading down into the foot. Everything above it is a setting, and settings
scroll.

`TwoColumn` also draws `rail_edge`, a 1px `EDGE` hairline between the two. In a
short window the scroll body is cut through the middle of whatever happens to be
there, and without a line the cut runs straight into the pinned foot and reads as
a rendering fault. It is owned by the shell rather than by each tab, so the two
rails cannot draw the boundary in different places.

**Controls are named, not explained.** Every slider and combo carries a visible
label on one shared column (`theme.FIELD_LABEL_WIDTH`, and `shell.labelled()`
for anything that is not a `PercentSlider`). The names used to reach
`setAccessibleName` and nowhere else, which meant a screen reader was told which
slider it was and a sighted user was not — two identical rows distinguished only
by a sentence printed underneath, so you read past a control to learn what it
was. Help text is now only for what a label cannot say. Labels are short because
the section heading carries the topic: BORDER's slider is `Width`.

**FORMAT is the output frame's shape; FRAMES is everything about the frames
themselves.** The count, the three count buttons, where the frames land and the
way back all used to sit under FORMAT, which described none of them — the
selection sentence and the undo sentence in particular floated under a heading
about ratios. Split in two, FRAMES has an owner for all of it.

The count row is `Count` on the shared label column with the number in the
control position, which is what let `NO_COUNT` go. That constant was a sentence
("Load a source to see the frame count") doing two jobs, because the row had no
label and the sentence was the only thing naming it. Its docstring said the
count must never be stale or guessed, and `split_tab.UNKNOWN_COUNT` — an em dash
— is neither: it is the same claim in one character, next to a label that
already says which row it belongs to. The count buttons cluster at the right
edge with the stretch between them and the value, so a dash becoming a numeral
never slides add and remove out from under the pointer.

Folder mode reads the same dash, and `PER_FILE_COUNT` is said underneath it
instead (`count_note`, hidden while empty like everything else here). Folder
mode is still a different state from no source — a folder *is* loaded, it just
has no one number — but the value slot cannot carry that difference: between the
label column and the three buttons, anything that actually answers "how many?"
overflows the rail, with "one per file" bringing the row to 309px against 284.
Three words that do fit ("one each") are cryptic rather than short. So the value
says there is no one number and the sentence says why.
`test_the_count_row_fits_the_rail_at_every_value_it_can_hold` holds that room,
which `test_no_rail_control_is_clipped_horizontally` cannot: it measures the
rail as constructed, where the value is one character wide.

**Nothing reserves a line for a sentence that has not arrived.** Under FRAMES,
the selection sentence and the undo offer share one row — selection left, offer
right-aligned in the same `QHBoxLayout` — and the row goes when both are empty.
They were two rows, both empty until something happened, which with the old
count sentence left 96px of nothing between the count buttons and BORDER, and in
folder mode left it there permanently, where neither half can ever fill in. It
read as a rendering fault rather than as spacing.

One row rather than two hidden labels because the row takes its space once and
keeps it: selecting a frame and then recording a step move nothing, since the
row that holds both is already standing. Nothing here animates, so a shove per
event would be abrupt. The offer takes the row's slack rather than sitting
against a stretch, so a long offer beside a long selection wraps onto a second
line instead of being cut off — the rail is 284px wide and the two sentences can
both be long at once.

The two labels at the bottom of each pinned foot — Split's error, Compose's hint
— are hidden while empty instead. Both sit above a stretch with nothing under
them, so the space they give back is space nothing was using.

**`shell.Disclosure` folds a section away and states its value in the heading**
(`+ FRAME 1   whole panorama`). Folding is only acceptable because the
summary is there — otherwise the setting is buried rather than out of the way.
Instant, never animated, like everything else here.

A summary that elides defeats the fold as completely as one that was never
written, so it has to fit the header's own width for every value it can take,
not just the one open on screen. FRAME 1's summary (`split_tab._frame1_summary`)
learned this twice over (maskingframe-2rg.11): the row clause says "whole
panorama" rather than the rows list's own "the whole panorama", because a
dropdown row can afford the article and a 284px heading cannot; and the border
clause says "own border" rather than the stored percent, because `f"{own:g}%"`
has no fixed width and a slider parked on 3.3% or 12.5% is longer than one
parked on 9%. Both clauses now say *whether* something is set, never *what
to*, which is also the more honest claim for a summary a click will expand
into the real controls. `test_no_frame1_summary_overflows_its_disclosure_header`
drives every reachable row count and border state through real font metrics
rather than trusting that the state someone thought to render is the widest
one.

Everything frame 1 alone decides lives in one folded section: the row layout,
the gap between rows, and its own border. Those were scattered across FORMAT and
BORDER, each carrying a sentence to re-explain the scope the heading now states
once. **On Split the gap is inside FRAME 1** because a split's gap separates
nothing but frame 1's rows; on Compose it stays in BORDER, where it separates
panels. One control and one stored field, two labels.

Above that fold, BORDER reads as one thing: preset, width and colour, what the
percent means, which frames carry it. The detail-frames checkbox used to sit
just under the fold's heading, closer to it than the gap that separates every
other break in the rail, so a control about every frame except frame 1 read as
one of frame 1's own. It sits above the fold now, straight after the sentence
it follows. That sentence's wording follows what is actually beside it:
singular on Split, because with the fold closed the row gap has moved inside
it and Width is the only percent slider left up here; plural on Compose, where
the panel gap is a second one that never folds away, placed after both rather
than above the one it hadn't reached yet.

### Remembering the border

`gui/settings.py` is the only module that constructs a `QSettings`, so a reader and a writer can never end up on different files. It states its format and scope explicitly: the two-argument `QSettings(organisation, application)` constructor pins itself to the platform's native format and then ignores `setDefaultFormat` and `setPath`, which on macOS means a plist a test cannot redirect.

A stored value is untrusted input — the file is plain text the user can edit, and it outlives any release — so `load_style()` validates every field through `FrameStyle` and falls back to `DEFAULT_STYLE` whole if any of it is wrong. Half a remembered setting is more confusing than none.

Split and Compose store their styles under separate scopes (`settings.SPLIT`, `settings.COMPOSE`). A split border and a composite border are different decisions, and sharing one value would surprise whichever tab the user touched second.

A test that builds any widget touching `QSettings` isolates it for the whole module, with `pytestmark = pytest.mark.usefixtures("isolated_settings")` once at the top of the file, not per test that happens to need it. `QSettings.setPath` is process-global, so an unmarked test's result depends on whether some other, isolated test already ran first in the same process and left it pointed at a throwaway store — which test ran first is not a fact a reviewer can see in the diff, so it must not be the thing deciding whether a test reads the developer's real preferences.

`BorderControls.frame_style()` is deliberately not called `style()`: `QWidget` already owns that name for its `QStyle`, and shadowing it would return a different type from an inherited method. `set_style()` restores stored state without emitting, so a tab that saves on `style_changed` does not write back what it has just read. `scope` is keyword-only, so a caller cannot silently hand a Compose rail the Split list.

A border can also be saved under a name. Presets follow the scopes: Split and
Compose keep separate lists, and a preset carries only what its own tab can
show — the split entries leave the gap at its default, because Split has no gap
control.

A stored preset is untrusted input like a stored style, but the failure rule
differs on purpose. `load_style` falls back whole, because half a remembered
style is more confusing than none. `load_presets` drops a malformed preset on
its own, because losing four good presets over one bad one would be worse than
the bug that wrote it.

Three built-ins are seeded per scope on first run and are ordinary presets
afterwards: delete one and it stays deleted, edit one and the edit stands. The
cost is that a preset added in a later release will not reach an existing
install — accepted against a tombstone list and two kinds of preset the
interface would then have to tell apart.

A preset is a *look*: border width and colour, gap width and colour, frame 1's
own border, and which frames carry one. It deliberately does not carry
`padded_rows` — a preset named "Gallery" that silently re-laid frame 1 out in
three rows would be changing the composition rather than the look, and the row
count belongs to the panorama's shape. `test_a_preset_carries_the_look_but_not_the_composition`
pins both halves of that.

The preset row stays inside the BORDER section rather than moving onto its
heading. On the heading the combo gets 72px of text room, which elides
`Black surround` — one of the three built-ins — and every name carrying the
`(edited)` marker, which exists precisely to be read. It would have bought 44px
of height, and since the rail stopped scrolling that buys nothing.

The row carries a `Preset` label on the shared column, like every other
control, and the combo carries a placeholder, `No preset`, instead of showing
blank. It is the only editable combo in the app, and with no placeholder an
empty one is visually a `QLineEdit` nobody has filled in rather than a picker
(maskingframe-2rg.10). Labelling the row leaves it little width to spare —
`Save` and `Delete` are already sized to their own wordings, not a
neighbour's, and now get less padding of their own for the same reason: a
glyph does not need a word's breathing room, and the combo does not have 14px
a side left to give it. The combo's own floor no longer follows whatever the
longest saved preset happens to be either, since a name is user-authored and
unbounded and would otherwise be able to overflow the row on its own,
regardless of anything else in it.

`shell.PresetRow` owns the naming rules and the button's wording and never
touches `QSettings`; `BorderControls` holds the scope and does the storing. The
button reads Save for a new name and Update for one that exists — that is what
stands in for a confirmation dialog, and it says which you are about to do
before you do it. `shell.EDITED_SUFFIX` marks settings that have moved away
from the preset they started as, and is stripped before any name is saved,
matched or deleted. `clean_name` also rejects a name containing `/` or `\`,
both group separators to `QSettings`; `PresetRow` disables the save button for
such a name rather than letting it reach the store.

At startup a tab calls `BorderControls.restore_style()`, not `set_style()`
directly: it restores the stored style quietly and then names it as a preset
if the style is exactly equal to one, so a border chosen before a relaunch
still shows as chosen. `PresetRow` listens for `Combo.currentIndexChanged`
rather than `activated`, because filling the list on that restore fires an
index change the row has to guard against; a tab that later wants to show a
name without announcing a choice should call `set_current()`, not
`setCurrentIndex()` directly.

### Concurrency

**This replaced the tkinter rule; it does not restate it.** There, nothing on a worker could touch any tk object and `root.after()` was the only sanctioned crossing back, so every worker hand-rolled that crossing with its own guard against the window closing mid-flight.

Qt queues a signal emitted from a worker thread to the receiver's thread automatically, and only auto-disconnects it when the slot is a bound method of a destroyed `QObject`. Every caller here passes a closure instead, which has no receiver to drop against, so that protection does not apply. The rule is: **a job returns plain data, and the callback runs on the GUI thread.** Use `work.submit(job, on_done, on_failed)`, and pass `owner=self` whenever the callback touches a widget — `submit` then skips the callback if `owner` has already been destroyed, and otherwise holds the job alive until the callback runs. Never touch a widget inside a job.

Two things that did *not* go away:

- **Staleness.** A user can pick a second file before the first inspection returns, so every background answer carries a monotonically increasing token that the callback compares before it writes anything.
- **Passing context with the answer.** `inspect_source` runs off the GUI thread; if the callback re-read the ratio combobox it could caption one ratio's frame count with another ratio's name. Whatever the job was computed *for* travels back with its result.

### Re-rendering a preview in place

A preview has its border rendered *into* the frames, so a changed setting makes it a picture of settings nobody has any more. It is made again rather than dropped, and three things keep that honest:

- **Cheap on every move, expensive only when the hand stops.** `shell.PercentSlider` emits `valueChanged` on every movement and `settled` once the value is chosen (slider release, or any change arriving with the handle up — an arrow key, Page, Home/End, a programmatic `setValue`). `BorderControls` mirrors that split as `style_changed` (drives the live overlay) and `style_settled` (drives the re-render). `set_style()` is silent on both.
- **The old picture stays up.** A settle never calls `clear_images()` when a re-render is coming; the new frames swap in when they land. The strip is only put back to the overlay when a re-render is impossible (source gone, a run in flight), and `shell.STALE_PREVIEW` says so. `shell.UPDATING_PREVIEW` covers the interval.
- **The decoded source is cached.** `pipeline.cached_preview_source()` holds one decode, keyed on path plus mtime and size, bounded to `PREVIEW_MAX_PIXELS` (28MP, ~84MB, against ~400MB for a full 132MP decode). The bound is set by the detail frames, which are cut from the source and scaled *up* to the ratio's full width — see the comment on the constant for the arithmetic. Only `preview_frames(..., cached=True)` uses it; anything that writes to disk goes on reading the full-resolution original. `compose_preview` needs no cache of its own: `_load_for_box` already drafts each source down toward its solved box.

### The theme

`gui/theme.py` is the design system, and it is presentation only. Two things about it are load-bearing:

- **The palette has range on purpose.** An earlier version was three greys within a few percent of each other with no white and no dark surface, and the whole interface read as a wash. `TABLE` → `PANEL` → `WELL` → `EDGE` → `INK`/`REBATE` spans white to near-black, and every text pairing clears WCAG AA. Don't add a value that sits between two existing ones without a reason.
- **`CHINAGRAPH` is the marking-up layer, and nothing else.** Think about what a grease pencil is actually for on a contact sheet: numbering the frames, ringing the select, marking the one to print, crossing out the one that failed. So chinagraph carries the primary action, the current selection (a selected list row, a checked radio, selected text), the frame and source numbering, the progress bar while a run is in flight, and error text. Chrome does not get it: surfaces, fields, sliders, dividers and rules stay greyscale. The primary action keeps its primacy under that rule because it is the only large filled block of chinagraph in the app — everything else is a hairline, a numeral or an indicator a few pixels across. A second saturated hue costs it that primacy, and so does filling another large area with this one. It is also why field focus is `INK` rather than red — a field turning red when you click into it reads as invalid. The same rule cost the disabled primary its chinagraph border: with nothing loaded, an outlined red box was the first thing on screen and it said something was wrong rather than that something was unavailable. It is `INK_DIM` now, which is what keeps it apart from `Secondary`'s `EDGE` hairline; a filled `EDGE` slab would have separated them more strongly, but `INK_DIM` on `EDGE` measures 4.3:1 and fails the AA floor for the button's 14px label.

No rounded corners, no drop shadows, no animation, anywhere. The direction is a light table and film rebate, both hard-edged; Qt making all three trivial is not a reason to spend them. `docs/design/2026-07-31-qt-port.md` records why the port happened and what the tkinter build got wrong.

### Border behaviour worth knowing

`make_padded_frame()` fits the panorama inside the target output frame, inset on all four sides by `style.border_px(ratio)`, preserving the panorama's own aspect ratio, then centres it on a full-size canvas filled with the border colour.

The border is a percent of the frame's **short** side, 9% by default, not an absolute pixel count. A fixed width cannot read the same at every ratio: 100px is a modest edge on a 1350px-tall portrait frame and a heavy one on a 566px-tall landscape frame. At 9% that resolves to 97px at `4:5` and `1:1`, and 51px at `1.91:1`. The default gutter is 4%: 43px and 23px.

The border describes the finished output frame, not the source image — whichever axis binds gets exactly the border, and the other axis gets whatever's left over. For a wide panorama at `1:1` or `4:5` that's the width. At `1.91:1` the default inset box is 978x464, ratio 2.108:1, so a panorama flatter than that binds on height instead; this project's own 2.33:1 samples are steeper and bind on width. The threshold moves with the border width, so work out the inset box before assuming the border means "left and right".

That asymmetry is inherent: the panorama's aspect doesn't match the frame's, and frame 1 must show the whole panorama uncropped, so the border can't be made even without cropping content away.

At a tall ratio like `4:5`, most of the padded frame is border — that is the intended aesthetic, not a bug.

`style.padded_border_percent` is frame 1's own border, `None` meaning it follows `border_percent`, resolved by `padded_border_px()` and read only by `make_padded_frame()`. It exists because turning the shared border down to let the panorama fill more of frame 1 also changes every detail frame that carries one, the composite, and the gutter's neighbours. `border_px` is untouched, so nothing about what the border means elsewhere moves with it.

What it cannot do is make a panorama fill a portrait frame. For a 2.33:1 panorama at `4:5` the share of frame 1 goes 23.1% at the default 9%, 29.1% at 4%, and **34.3% at zero** — the ceiling, because what leaves the space is the mismatch between a 2.33:1 picture and a 0.8:1 frame, not the border. `test_the_measured_ceiling_at_portrait` pins that number, since it is the reason the feature is bounded the way it is. Filling more would mean cropping, which frame 1 exists not to do, or cutting the panorama into stacked rows, which is a different feature.

`style.padded_rows` lays frame 1 out as that many equal full-width strips, read top to bottom, which crops nothing — every pixel is still shown, once. It is the only thing that moves the ceiling above: a 6:1 panorama at `4:5` goes from 9% of the frame in one row to 52% in three. Rows hurt at a wide ratio (a 2.33:1 panorama drops from 67% to 19% at `1.91:1`) and the best count moves with the panorama's shape, so it is a choice rather than something the application decides, and 1 stays the default.

`row_bounds()` cuts the strips: equal, adjacent edges shared exactly, the last landing on the panorama's width, so nothing is lost to rounding or shown twice. Equal is a consequence rather than a preference — the rows share a display width, so unequal source widths would give unequal heights and the stack would stop being one. The gap between rows is the composite's `gutter_percent`/`gutter_colour`, which is why Split's rail now shows the gap controls it used to hide. Gaps are painted before the rows, as `compose.render` paints a composite.

`padded_rows_fill()` says what a count is worth without rendering anything, so the rail can label four choices without four decodes of a 132MP scan. `test_the_advertised_fill_is_the_one_that_renders` holds it against the actual picture rather than against a second copy of its own formula — that number is what a user acts on.

`padded_border_percent` and `padded_rows` are two fields only `make_padded_frame` reads. That is still cheaper than a type of its own; if a third appears, the honest move is a `Frame1Style`.

The GUI control is a checkbox and a slider on Split only — a composite has no frame 1, so `BorderControls` takes `show_frame1` alongside `show_gutter` and `show_detail_toggle`. Ticking adopts the shared width, so the box reveals a control rather than moving the frame. `frame_style()` reports `None` while unticked rather than the slider's number, and `set_style` tests the stored value against `None` rather than for truthiness: 0 is full bleed, a real choice, and must not read back as no choice made.

`make_padded_frame()` scales the panorama directly to its fitted size with `Image.Resampling.LANCZOS` and pastes it onto a canvas already sized to `(ratio.width, ratio.height)` — the same pixel size as every detail frame — rather than compositing at source scale and downscaling. That keeps a large-format scan from briefly allocating a huge intermediate canvas as well as avoiding a many-megabyte first frame beside sub-megabyte detail frames.

Detail frames are full-bleed by default. The border is what makes frame 1 the establishing shot, and giving every frame one flattens that distinction. `style.border_detail_frames` turns it on for people who want the whole carousel to read as one object; the crop then targets the inset box and the border is drawn around it.

### Where the detail frames land

A detail frame is one number: its left edge as a fraction of the panorama's
width. The width is derived, not stored — `pano_height * ratio` — so every
detail frame is a full-height crop at exactly the output aspect and moving one
frame never resizes another.

Positions are held ascending, and a frame dragged past its neighbour stops at
it rather than swapping with it: the frames are numbered, and a carousel
running backwards along the picture is confusing. Overlap is allowed, because
two tight crops on one subject is a legitimate choice.

`section_count()` is unchanged and still gives the opening count. It is a first
guess now, not a constraint.

`Even` puts the spacing back without touching the count — `geometry.default_positions`
at the count in hand, not at `section_count`'s guess, because spacing and count are
separate decisions and resetting one must not silently discard the other.
`geometry.positions_are_even` decides whether the button has anything to offer, and it
is off while the frames already are. The recompute lives in `_set_positions`, which
every change to the plan funnels through: `Even` reads the spacing rather than the
count, so leaving it to `add_frame`/`remove_frame` left the button dark for exactly the
person who had just made it applicable. All three count controls start unpressable —
a `QPushButton` does not — and carry an accessible name equal to their tooltip, since
a button whose whole label is `+` tells a screen reader nothing.

The ribbon (`gui/ribbon.py`) sits above the contact strip and shows where the
frames come from; the strip shows what they are. Both write to one tuple owned
by `SplitTab`, so they cannot disagree. Folder mode hides the ribbon and cuts
with the even default — a position is chosen by looking at one photograph — and
says so where the ribbon would be, because silence would read as a missing
feature. The CLI has no position flags for the same reason.

Both views also take focus and handle keys, because a feature that decides what
every output frame contains must not need a pointer. The line under the ribbon
says so (`split_tab.KEY_HELP`): with the ribbon up it states the keys, and with
it hidden it says why there is nothing to place. Left and Right move the
selected frame by `split_tab.KEY_STEP` (1% of the panorama's width), Shift makes
that ten steps, Home and End send it to the ends of its travel, and Up and Down
move the selection, stopping at the ends rather than wrapping. A key press goes
through `geometry.move_position`, the same rule both drags obey, so the two
cannot disagree.

The widgets emit a count of steps rather than a distance. How far a step moves
is policy, and the tab is the only thing that knows what a percent of this
panorama is — so `ribbon.py` and `strip.py` stay presentation only. Each widget
speaks its own numbering: the ribbon's windows are detail frames, the strip's
frames include frame 1, and the tab converts, exactly as it does for the drags.

The selection is marked in chinagraph on the selected frame's edge — the
ribbon window's border, the strip frame's aperture — and stated in words in the
rail (`Frame 3 · 42% along`). Both, not either: a state carried by colour alone
fails the WCAG 2.2 AA floor this project holds. Every numeral in both widgets
stays chinagraph whatever is selected, as the sources list's do: the numbering
is the marked-up sheet, and demoting it to ink to make one numeral stand out
spends the thing that made it read that way.

Focus is a different state from selection and the two can be present at once,
so focus takes the widget's own outer edge in `INK`, two pixels wide
(`strip.FOCUS_EDGE`), exactly as a focused field does. `ContactStrip.edge_pen`,
`ContactStrip.aperture_pen`, `ContactStrip.numeral_colour` and
`FrameRibbon.edge_pen` are the decisions themselves, exposed so a test can
assert the colour rather than re-derive the geometry.
The rail's sentence is also what a screen reader hears: `SplitTab` pushes it
verbatim into both widgets with `setAccessibleDescription`, so the two views and
the rail cannot say different things about the same frame. `setAccessibleName`
stays each widget's plain identity — "Panorama overview", "Contact strip". The
widgets compose no sentence of their own: a percent of the panorama's width is
knowledge neither has ever had, which is the whole reason the tab converts for
them, and the strip's old "Frame 3 of 5" said the same words after every arrow
press.

Setting that property raises no event, so `_show_selection` also raises a
`QAccessibleAnnouncementEvent`. Without it a screen reader read the sentence
when focus arrived and never learned it had changed: holding an arrow key moved
the frame, counted up the rail on screen, and said nothing.

An earlier version raised `DescriptionChanged` instead, which was very likely
never spoken. Both views report `Role.Client` with no value interface — a
generic container — and a description change on one of those is not something
assistive technology is obliged to announce; a `QSlider`, which is announced,
reports `Role.Slider` and a value interface. The announcement event is the tool
for this and does not depend on the role: it says "speak this now", politely, so
it queues behind whatever the user is already being told.

**Writing a sentence and speaking one are two things, and `_show_selection` and
`_say` are the two methods.** `_show_selection` is the only thing that writes
the selection label or either view's accessible description; `_say` raises the
announcement and writes nothing. They were one method, so `_apply_snapshot`
announcing `Undo remove frame` overwrote the selection sentence on screen and in
both views and left it there — a frame marked in chinagraph with nothing naming
it, which is the failure the words exist to prevent, and both views telling a
screen reader about an undo instead of about the selection.

The two also deduplicate differently, on purpose. The selection path drops an
unchanged sentence: one keystroke reaches it twice, and a reader that said the
position twice per arrow press would be worse than one that never said it.
`_say` drops nothing, because every call is a press the user has just made —
two `move` steps walked back are two actions with the same name, and a second
undo that said nothing would read as the key having stopped working.

Both are raised once, on the tab, rather than once per view: the message travels
with the event.

The strip takes its frame count from the header read, in `_apply_facts`,
not from the first render — the rail, the band and the button all state it
there, and a strip still showing its construction-time apertures made two
halves of one screen disagree. It is relabelled only while the apertures are
empty: `ContactStrip.set_frames` discards every thumbnail, and `_rerender`
reads `strip.exposed` to decide whether a render is worth redoing, so
relabelling with a preview up would take the picture down and stop its
replacement ever starting. With a preview up the old picture stays up and the
render that replaces it does the relabelling.

That fix only ever covered the count changing after construction. Split
builds its strip with one aperture (`ContactStrip(..., frames=1)`), not
`ContactStrip`'s own `DEFAULT_FRAME_COUNT` of four: before a source is read
the rail already says the count is unknown, and four numbered apertures
stated a count nobody had confirmed, right for the panorama the samples
happen to use and wrong on principle. Compose builds its own strip the same
way — it never has a frame 1 to count either, which is what to check before
touching `ContactStrip`'s own default rather than a caller's.

### Remembering where the frames were

`gui/settings.py` stores a plan per source, keyed on the source's **path,
mtime and size** — the same three facts `cached_preview_source` keys its
decode on, so there is one answer in this codebase to "is this the same
file". A stat, never a read: deciding whether to restore a handful of floats
must not cost hundreds of megabytes of I/O on a 132MP scan. An edit that
preserves both mtime and size is missed; that takes a deliberate `touch`, and
the cost is crops a few pixels off rather than anything corrupted.

The `QSettings` group is a hash of the resolved path, not the path: a path is
full of the separators `QSettings` reads as group boundaries, which is the
trap a preset name containing a slash fell into. `MAX_PLANS` is 50, least
recently used evicted, and **reading counts as using** — otherwise the
panorama you reopen every day would be evicted by files you saved once and
never came back to. A malformed plan is dropped on its own, following
`load_presets` rather than `load_style`. The key uses `st_mtime_ns`, matching
`cached_preview_source` exactly rather than approximately: whole seconds would
be coarser than the cache it claims parity with.

`padded_border_percent` is stored through `_optional_percent`/`_set_optional`,
which read against the key's *presence* and remove it when the value is None —
so an unticked box and a store written before the field existed are the same
thing on disk, and 0 (full bleed) never reads back as no choice made.

A plan carries the row count as well as the positions — `settings.Plan` — because
the right number of rows depends on the panorama's shape (three for a 6:1, two
for a 2.33:1), so it belongs to the photograph rather than to a taste that would
outlive it. That is also why it is not stored globally the way the border is. A
plan written before rows existed reads back as one row, which is how it was laid
out.

The ratio is not part of the key. A position is a fraction of the panorama's
width, which means the same thing at every ratio, and the count has been the
user's decision since add and remove landed.

Stored on the settles — a drag release, the keys stopping, add, remove,
Even — never on a drag move, which would write a plan per mouse event.
Folder mode stores nothing. `split_tab.RESTORED` says a plan was put back,
above the key help, and goes as soon as anything moves: the sentence is about
how the plan arrived, and once you have changed it, it did not arrive that
way.

A selection is dropped at the cause, not guarded at every reader.
`FrameRibbon.set_plan` and `ContactStrip.set_frames` both drop a selection the
incoming plan or frame list can no longer support, and `SplitTab._set_positions`
makes the same decision for the tab's own copy. That is why no reader of the
selection needs a bounds check. The alternative — clamping frame 3 to frame 2 —
was rejected: it silently substitutes a different frame for the one the user
picked, which looks like the same selection but points at different content.

Focus announces the selection it makes. Both widgets emit `selection_changed`
when focus is what selected the frame. Without that, a single Tab press marked
a frame in chinagraph while the tab still held nothing and the rail was empty —
the state was carried by colour alone, which is the failure this work exists to
remove. `set_selected` stays silent on both widgets, so the tab adopting the
selection and echoing it back cannot loop.

A held key settles rather than rendering per repeat. `split_tab.NUDGE_SETTLE_MS`
is 120 ms, comfortably longer than a single auto-repeat interval (the platform
fires 25 to 30 times a second), so one render happens when the key stops, the
way a drag renders once on release. Positions and the spoken names still update
on every keystroke; only the picture waits. The timer is stopped when a run or
a preview starts, so a pending settle cannot fire into one.

A source narrower than one output tile (a 1.5:1 image at `1.91:1`) has no
travel at all: every position clamps to zero and the crop is the whole width.
Degenerate, but it must not raise.

### Walking a placement back

Placing the detail frames is the craft work here, and `Even` discards every
hand-placed position in one press. `gui/history.py` is the way back: a list
of plans with a cursor, no Qt and no I/O, so it is tested in memory the way
`geometry` is.

A snapshot is the plan and nothing else — the positions and the row count.
Not the selection, which is which frame you are looking at rather than work
you have done, and which `_set_positions` already drops when a plan cannot
hold it. Not the border, the gap or the colours: those have presets and are
remembered between launches, and a key that sometimes moves frames and
sometimes changes a colour is harder to predict than one that always does
the same kind of thing.

`SplitTab._remember()` writes the plan to the store and `_record(label)`
notes it as a step, and the seven settles call both. **Undo and redo call
`_remember()` but never `_record()`** — that is what stops them recording
themselves, and it is the one invariant a test holds. Applying a snapshot
moves the rows combo under `_filling_rows`, so the move cannot be mistaken
for a choice and cannot record a step of its own either.

Both are also guarded on `_positions_source`, the source `_positions` was
computed for, set in `_apply_facts` alongside `_source_size` and cleared in
`_clear_facts`. `_positions` is deliberately left on screen from the
outgoing source until the incoming one's header read lands — see
`_rerender` — so between choosing a new source and that read arriving,
`source_row` names the new file while `_positions` still holds the old
one's plan. A settle firing in that window used to write the old plan
under the new source's key and, separately, seed the new source's --
just-cleared -- history with it, so the first undo walked back into a
photograph that was not the one on screen. Comparing `_positions_source`
against `source_row.text()` closes both at once, since they are one field
naming the same fact.

`record` ignores a snapshot equal to the one on screen. A settle fires
whether or not anything moved — an arrow key held against the clamp, a drag
that went nowhere — and recording those would give undo presses that
visibly do nothing.

The history clears on every change of source, including to none and to
folder mode: undoing into a plan made for a different photograph would
restore crops that mean nothing. It survives a ratio change, because a
position is a fraction of the panorama's width and means the same thing at
every ratio — which is also why the stored plan is not keyed on the ratio.

Undo writes to the store like any other change, so quitting after one and
reopening gives back what was on screen rather than the version you undid.

`MAX_STEPS` is 50, dropping from the front. A snapshot is a handful of
floats, so the bound is against a list growing without limit over a long
session rather than against the memory.

`QShortcut` is built from `QKeySequence.StandardKey.Undo` and `.Redo`
rather than hardcoded keys, and `undo_keys()`/`redo_keys()` ask Qt what this
platform calls them, so macOS reads ⌘Z and every other platform reads its
own convention without this file knowing which one it is on. These are the
first shortcuts in the application, so the scope is a decision:
`WidgetWithChildrenShortcut` on the Split tab, so ⌘Z on Compose does
nothing rather than quietly changing a tab you cannot see.

`undo_keys()` and `redo_keys()` are `functools.cache`d rather than module
constants computed at import time, because constructing a `QKeySequence`
before a `QApplication` exists is a hard segfault (exit 139): evaluating
either at import time would crash any import of `split_tab`, including a
test that never touches a widget. A cache gives the same once-only cost
without a `global` statement or a "constant" that reads `""` until
something happens to have called it -- which is what the two functions
replaced, and which needed `assert split_tab.UNDO_KEYS != ""` in tests to
stop `"" in text` passing vacuously.

`split_tab.undo_line` sits under the count controls and reads
`Undo Even   ⌘Z`, falling back to `Redo <label>` once undo has nothing
left and empty when there is neither. When both directions are available it
reads `Undo Even   ⌘Z   ·   Redo  ⇧⌘Z`: one action named, both keys shown.
Redo used to appear only once undo was exhausted, which meant someone one
step into a three-step history never saw the redo shortcut at all — and the
argument for printing a key here is that there is no menu bar, so by that
argument redo was half-advertised. Naming both actions does not fit:
`Undo remove frame   ⌘Z    Redo remove frame   ⇧⌘Z` measures 315px against
the rail's 284px. The label is the expensive part to read and the key is
not, so redo contributes only its key, and the widest reachable wording
comes to 212px on macOS. `undo_wording()` and `redo_wording()` own the strings so a
test can measure every one of them; the labels it measures are read out of
the `_record` call sites rather than listed, so a seventh settle with a
longer label cannot slip past.

The spaces around each key, and before the separator, are `split_tab.NB` —
no-break spaces. Sharing a row means a long offer wraps, and left to break
wherever it liked Qt put the redo key alone on the second line. Bound, the
line breaks after the separator, not before it: undo and the middot stay on
one line and redo starts clean on the next. The separator was bound forward
at first — a no-break space after it, a plain one before — which fixed the
stranded key but left the middot itself starting the wrapped line, reading
as a typo. Binding it to the text it follows instead of the text it
introduces is what a separator normally does anyway; this just makes the
line-break rule agree with that. The action's own name stays breakable,
which is a sentence wrapping rather than a key stranded, and it is what
keeps the row's minimum width small enough to share.

It shares one row with the selection
sentence — see "How the rail is laid out" — taking the row's slack rather
than sitting against a stretch, so a long offer beside a long selection
wraps onto a second line instead of being cut off. It carries the key because the
application has no menu bar, so there is nowhere else a shortcut could be
advertised — and an undo nobody knows about is close to no undo. The label
is the risk rather than the depth: a line that says `Undo Even` and then
restores something else is worse than no line, so each of the seven sites
names its own action and the tests assert the label beside the state.

It is `undo_line`, not `undo_label`, because `History.undo_label` is a
string and having the two spellings mean different things one attribute
apart is a trap.

Compose gets none of this: its arrangement is already reversible by
choosing another, and its source list has explicit add and remove.

### The two colours on a composite

A composite has a border colour and a gutter colour, and they cover different things. The gutter colour fills **only** the strips between adjacent panels — the rectangles `layout` reports in `Layout.gutters`. Everything else is the border colour: the outer margin, and the slack left over from centring the assembled block inside it. Set both to the same value and the composite looks exactly as it did before the two were separable, which is why the default for both is white.

### Behaviour changes from the pre-refactor scripts

- `find_panoramas()` no longer double-counts files on case-insensitive filesystems (the old code globbed `*.jpg` and `*.JPG` separately, matching the same file twice).
- Folder mode defaults its output to `./output` when omitted; the old CLI required it and errored otherwise.
- Batch failures are reported rather than silently swallowed: the CLI exits non-zero and prints failures to stderr, and the GUI status reads "Wrote N of M images at {ratio.display}, K failed" instead of always claiming success. The GUI also names the ratio and frame count for single-file runs ("Wrote N detail frames at {ratio.display}"), and shows an explicit "No panoramas found" dialog instead of a green success dialog when a folder has no JPEGs.
- `process_image()` converts the source to RGB (`.convert("RGB")`) before processing, so a grayscale or RGBA/P-mode source now produces RGB sections instead of preserving its original mode. The legacy script had no such conversion and crashed with "cannot write mode RGBA as JPEG" on non-RGB inputs; the refactor fixes that but means the "byte-identical to the legacy output" guarantee is scoped to RGB inputs.
- The output is no longer a fixed four images (one padded square plus three 1080x1080 sections). It is one padded frame plus a variable number of detail frames — never fewer than two — derived from the chosen ratio and the panorama's aspect ratio via `section_count()`. The frames after the first are a zoom, not a tiling, which is why the count floors at two rather than one.
- A detail frame is no longer a `pano_width // count` tile. It is a full-height crop exactly `pano_height * ratio` wide, placed by a position — its left edge as a fraction of the panorama's width. The tiles no longer meet edge to edge, which they never needed to: the detail frames are a zoom, not a tiling. The gain is that nothing vertical is discarded. Before, a 2.4:1 panorama at `1.91:1` got two 1.2:1 tiles that `make_section` scaled to cover and then centre-cropped the top and bottom away from. The golden hashes were re-baselined on 2026-08-01; the padded frame is unaffected.
- Portrait input (width < height) is rejected with a `ValueError` naming the offending file; in a batch this is caught per-file so the rest of the batch still runs.
- `Image.MAX_IMAGE_PIXELS` is set to `None` in `pipeline.py`. The user's own large-format scans can exceed Pillow's ~178MP decompression-bomb guard (the largest sample here is 132MP); lifting the guard stops a legitimate scan being reported as a corrupt file. Malformed input is still caught by the per-file exception handling in `process_folder()`.
- The padded output was renamed from `{prefix}_1_padded_square.jpg` to `{prefix}_1_padded.jpg`, since it is only square when the target ratio is `1:1`.
- The border and the composite gutter are settings rather than constants. The `SIDE_PADDING`, `GUTTER` and `BACKGROUND` constants are gone, replaced by a `FrameStyle` passed down the call chain. Both widths and both colours are set from the CLI flags or the GUI's Border section, and the GUI remembers them between launches.
- Composites take two to six images rather than two or three. The arrangement is still chosen automatically and input order is still never permuted; the candidate list is generated rather than hand-written, and is restricted to one level of grouping (see `layout.py` above for why).
- Arrangement names changed from slugs to notation: `row-one-then-two` is now `R(1,C(2,3))`, images numbered from one. The three-panel arrangements are the same six pictures under new names, and the rail's English for them is unchanged — `present_layout()` parses the notation instead of the slug and reaches the same words.
- The composite arrangement can be forced rather than only accepted: an Arrange combo in the Compose tab's FORMAT section, beside Ratio rather than under a heading of its own, and `--arrangement` on the CLI. Labelled "Arrange" rather than "Arrangement" -- the full word overflows the rail's shared 68px label column, and FORMAT already carries the topic. The Compose tab holds the choice as a *name*, never a row index — the ranking moves with the ratio and the border, so an index would quietly come to mean a different arrangement. The choice survives a ratio, border, gutter or reorder change and is dropped only when the number of sources changes, where the shape it names stops existing. It is not remembered between launches: a border is a house style, which is why presets exist for it, but an arrangement belongs to one particular set of photographs. At rest, with fewer than two sources loaded, the combo shows the single entry `Automatic` and is disabled — an empty enabled combo is indistinguishable from a broken one, so it always carries at least that one honest entry.
- Frame 1 can carry a different border from everything else (`--frame1-border`, or the checkbox in the Split rail). Unset by default, which is exactly the old behaviour, so no stored style, stored preset, scripted run or golden hash moved.
- Frame 1 can be laid out as two, three or four stacked rows (`--frame1-rows`, or the list in FRAME 1). Off by default, so nothing anyone has made changes.
- The Split tab remembers where a source's detail frames were placed and puts them back when it is reopened. The CLI has no position flags and gains none: a position is chosen by looking at a photograph.
- The default border at `1.91:1` changed from 100px to 51px. That is the point of measuring in percent: the old fixed 100px took a third of a 566px-tall frame, while the same constant was a thin edge at `4:5`. Output at `4:5` and `1:1` moved too, from 100px to 97px.
- The Split tab's frame placement can be undone and redone (⌘Z and ⇧⌘Z, or the platform equivalent). Fifty steps, cleared when the source changes, and the undone plan is written to the store like any other change. Nothing on the CLI changes: a position is chosen by looking at a photograph, so there was never a flag to undo.


<!-- BEGIN BEADS INTEGRATION v:1 profile:minimal hash:6cd5cc61 -->
