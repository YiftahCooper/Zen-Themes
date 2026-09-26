# Tall pinned-folder scrolling repair

## Reproduction

On Zen 1.22.3b (Gecko 156.0.1), enable SuperPins 1.7.2's **Keep pinned tabs at
the top when scrolling** option. Create a pinned folder with enough tabs to
exceed the sidebar height and several ordinary tabs outside it. Expand the
folder and scroll. The ordinary section can have a zero-height viewport while
its tabs still exist; collapsing the folder restores it.

Use a window with room for Essentials and the tab sections. An Essentials grid
that itself exceeds the available window height is a separate case.

The native-only fixture did not reproduce this failure. SuperPins' stay-at-top
rules and Even Better New Tab Button 1.0.3's sticky rules each reproduced it
independently. When both are installed, both need the correction: disabling or
fixing only one can leave the other mod's sizing problem active.

## Repair

- Let pinned content keep its natural height when short, shrink when tall, and
  scroll independently. Reserve two native row pitches for the ordinary section
  so the New Tab row cannot consume its entire viewport.
- Set the native arrowscrollbox's shadow `items-wrapper` minimum height to zero
  through an author-origin `::part` rule. In the tested Gecko build, adding that
  rule only to the mod's user-origin stylesheet did not fix the shadow slot.
- Reveal the selected or focused tab after native selection, focus and resize.
  Zen's outer-scroller overflow check does not cover these separate scrollers.
- Continue native drag edge-scrolling in the pinned section. Native selection,
  grouping and drop handlers remain in charge of the actual operation.
- Share one set of input listeners with other owners of the same repair. Sine
  unload removes this mod's style and ownership; the final owner removes the
  listeners and pending animation frames.

The module changes no preferences, tab titles, folders or saved session data.
Existing SuperPins options, scrollbar hiding and the orientation script remain
in place. Both the CSS and input support are gated by vertical tabs and the
stay-at-top preference. The complete repair requires Sine JavaScript support;
copying the CSS alone is insufficient.

## Verification and limits

The repair candidate was exercised in native Zen/Gecko before packaging here:

- 24 layout cases: SuperPins alone, corrected New Tab Button alone and both
  stylesheet orders, at three window heights, with expanded/collapsed sidebar.
- A further native Gecko check of the packaged module with legacy pinned grid
  layout enabled and at least three direct pinned children. The legacy rule's
  important visible overflow initially prevented wheel scrolling in the pinned
  section. Making the stay-at-top vertical overflow important restored scrolling
  (0 to 292 px), kept the ordinary viewport at 80 px, and left the first and last
  tabs in both sections reachable.
- Sine unload/reload, preference gates, workspace changes and cold-start module
  registration; scrollbar-hiding preference checked separately.
- Visible-window wheel scrolling, native New Tab actions and keyboard reveal.
  Manual native drags reordered tabs in both sections and reached both last
  tabs. An automated pointer harness did not deliver native drop events even
  in its control, so those attempts are not counted as successful drag tests.
- A temporary everyday-profile trial restored the ordinary viewport from zero
  to 80 px at the tested density while preserving all 61 tab identities and
  checked folder/preference state. The user confirmed reachability and correct
  expand/collapse behavior. Undo was separately verified in the disposable
  profile. This was an in-memory trial, not a permanent profile installation.

This contribution ports that candidate to TypeScript and the repository's Bun
build. To check its generated module without launching a browser:

```sh
bun install --frozen-lockfile
bun run build
node --test test-superpins.mjs
```

The Node tests cover packaging, reveal behavior, preference changes during
pending work, continuous drag scrolling, cleanup and shared-owner lifetime.
They do not emulate Gecko layout or replace the native checks above. The final
generated package has not been permanently installed in the everyday profile.

At the upstream base, `tsc --noEmit` reports three existing errors in
`scripts/build.ts` because `entry.split(sep)[0]` can be `undefined`. The Bun build
works. This repair does not change that unrelated build script.

## Adoption and rollback

The original mod identity and preferences are retained. A fork install must
replace the existing SuperPins package, not run a second copy alongside it.
Keep the prior package and registry entry before switching its source. Restore
that package and source entry to roll back; tabs and preferences need no
migration. Sine's unload hook removes the new runtime effects, and restarting
also clears them. Other installed mods may still require their own correction.
