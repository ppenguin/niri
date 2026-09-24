## Goal:

- a feature that allows us to override the default "fill direction" of new columns (i.e. first column will appear left/right, next column will appear left/right of the first column, and so on.)

## Ideas

- envisioned as a single option, something like "column-anchor", with values: left (current default), right, toward-center, away-from-center
- left/right just mean the monitor's own edges. toward-center/away-from-center are relative to the center of the connected monitors, i.e. if there are two monitors side by side, toward-center gives "right" behaviour on the left monitor and "left" behaviour on the right monitor, and away-from-center is the mirror of that. with three monitors in a row the middle one is already at the center, so it falls back to a fixed direction (see below).
- TBD: the middle of 3 monitors would have an anchor as its center, so an edge case. For now, just grow right (as vanilla default). ("symmetric"/zigzag growth around a centered anchor could be also an option, but has limited real value?).
- for a single monitor, or a monitor that sits exactly in the middle of the field, there is no left/right of center, so toward-center/away-from-center need a fixed fallback, e.g. toward-center behaves like right and away-from-center like left.
- always-center-single-column (and center-focused-column "always") should keep taking precedence over the anchor, since those are explicit about what should happen on focus.
- proposed implementation for monitor position is to check the min/max global x coordinates (the coordinate system we know from e.g. wdisplays, must be known by niri internally, and we would check the offset of the left-most monitor and know x_min, and the offset+width of the rightmost monitor and know x_max). We'd then calculate for each monitor whether its absolute coordinate (mon_x_offset+mon_width/2) ?< (x_max-x_min)/2 (might need correction for the absolute offset?) and know the "gravity" of the monitor, i.e. whether it's left or right of center of the "monitors field".
- it would be good to have this monitor gravity available to niri as a queryable property for other future work, and we might as well implement it in a generic fashion that specifies each monitor's center in e.g. polar coordinates from the center pixel (monitors center of gravity) of the entire monitors field, so we can easily express a monitor's relative position for layout purposes. for the anchor itself only the sign of the horizontal offset is needed, the rest is for later.

## Issues Solved By This

The above proposal should cover many use cases in a more or less generic way, such as the ones described in the related issues:

- #861

Related PRs (possibly solved in a more generic fashion by this proposal, or that could benefit from its infrastructure):

- #3787
