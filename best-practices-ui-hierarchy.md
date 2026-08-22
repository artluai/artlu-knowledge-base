# Best Practices: nesting hierarchy in lists and boards

reference doc for all projects. read this before building or reviewing any UI
where rows contain rows — job boards, outliners, file trees, task lists,
comment threads, nav menus.

unlike the design rules of thumb in `best-practices.md`, these are laws, not
heuristics. they came out of shipping the spoolcast board (2026-08-22) and each
one exists because its absence was visible on screen.

---

## the two failures

a board with several nesting levels and one visual treatment is a flat list
that happens to be indented. two causes, both easy to ship without noticing:

1. **cues too small to survive a real screen.** the spoolcast board's original
   ramp was a 1.5% background delta. it existed in the CSS and was invisible in
   daylight on an uncalibrated laptop. if you have to put two swatches
   side by side to see a difference, users never will.
2. **the container styled like its contents.** the group header and the
   top-level items inside it were byte-identical — same size, same weight, same
   fill, same left edge. indentation was the only thing carrying structure, and
   indentation alone reads as decoration.

---

## the laws

1. **the container is chrome; its contents are paper.** a group header is a
   filled band, not another row. separate it from the rows beneath by fill
   *and* size — never by weight alone, because weight is also how levels are
   separated inside the group.
2. **every level gets a visible fill step.** roughly 3–5% in lightness per
   level, not 1–2%. the test: can you spot the difference when the two levels
   are *not* adjacent?
3. **fill and type step together.** level should be legible from either cue
   alone. type recedes with depth (bold → regular → muted). nobody should have
   to count indents to know what they're looking at.
4. **one rail per ancestor.** each row paints a 1px vertical line in every
   ancestor's column, full row height. indentation says "this is deeper"; rails
   say "deeper than *this specific thing*". this is the cue that still works
   after you've scrolled past the parent — the one that actually makes
   containment legible instead of implied.
5. **hover leaves the level ramp's hue.** if levels step along blue, hover must
   not be blue, or a hovered row reads as a level change. hover is a different
   axis, not another step along the same one. same for selection and any other
   transient row state.
6. **fills are monotonic, and the container anchors one end.** order every fill
   by lightness with no ties. two treatments at the same lightness are two
   things the eye can't rank.
7. **semantic left edges live outside the rail gutter.** blocked / error /
   selected markers take the first few pixels, left of where rails start. a
   structural cue and a semantic cue must never share a column.

laws 5, 6 and 7 are the ones that only show up once it's on screen. they are
invisible when reading a diff, and they are where a reimplementation regresses.

---

## implementation

derive level from the parent link at render time. persist nothing about depth.

rows can stay flat in the DOM — no nested containers needed. tell each row its
depth through a custom property and let it paint its own rails as one repeating
gradient clipped to the indent gutter. handles any depth with no per-level rule:

```css
.row {
  --indent: 17px;
  --rail-origin: 10px;
  padding-left: calc(var(--rail-origin) + (var(--depth, 0) + 1) * var(--indent));
  background-color: var(--row-fill, var(--level-0));
  background-image: repeating-linear-gradient(
    to right,
    var(--rail) 0 1px,
    transparent 1px var(--indent)
  );
  background-repeat: no-repeat;
  background-position: var(--rail-origin) 0;
  background-size: calc((var(--depth, 0) + 1) * var(--indent)) 100%;
}
```

each depth class sets `--row-fill` only. hover overrides `background-color`
alone so the rails survive it.

on narrow screens shrink `--indent` and `--rail-origin` instead of dropping
levels — three levels at an 11px indent still fit a 320px screen.

---

## verifying it

screenshots, not description. all of:

- two adjacent levels, and two levels separated by several rows.
- a row with a semantic left edge, at more than one depth (law 7).
- hover, shown against the container band (law 5).
- the narrow breakpoint.
- **a full-height scroll with several groups stacked.** a band that reads well
  in isolation reads heavy seven deep. a single-group screenshot won't show
  this, and it's the most likely thing to be wrong.

---

## porting

port the laws, not the values. the spoolcast board is a warm-paper light skin;
a dark board that copies its hex codes gets a broken result and blames the
system. pick a ramp in your own palette that satisfies laws 2, 5 and 6.

reference implementation: `spoolcast-web/tools/board/` — `HIERARCHY.md` for the
system, `public/styles.css` for the code.
