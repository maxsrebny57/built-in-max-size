# Built-in max size: a one-file kit for Claude Code

The largest built-in that fits an opening, from a site measurement: width, depth and
height in mm, from 51 numbers taken at three levels. The unit is then scribed to fit on
site. The calculation is a plain Python script with no dependencies; the agent
instructions turn it into a workflow: send a measurement in any form (text, a table, a
photo of a marked-up sketch), the agent fills the template, flags what doesn't add up
before you start designing, and runs the numbers.

This file carries the whole project. Nothing to clone.

## Install

1. Put this file into an empty folder.
2. Open Claude Code in that folder and say:
   `Read built-in-max-size.md and set up the kit exactly as it says.`
3. The agent creates the files below, runs the tests and shows the example result,
   1221 x 605 x 2607 mm. From then on, just send it a measurement.

Needs Python 3.8 or newer. Nothing else. The kit also writes `AGENTS.md`, the same
instructions for Codex and other agents that read that file; so far it has been tested
in Claude Code only.

## Instructions for the agent

You are setting up this kit in the current folder.

This document is about 1,370 lines long and holds eight files; the last one is
`CLAUDE.md`, followed by a short footer. Read it to the end before writing anything.
If your reader truncates it, read the rest in further passes.

1. Create every file listed under **Files** with exactly the content shown: same names,
   same folders (create `measurements/`), content copied verbatim, nothing added or
   reformatted. No line in any file ends with a space. Then create `AGENTS.md` as an
   exact copy of `CLAUDE.md`.
2. Run `python3 test_builtin_max_size.py`. All 19 tests must pass; one of them parses
   `field-map.svg` and checks that all 51 labels are there, so a broken transcription
   of the drawing shows up here too. If a test fails, compare your file with this
   document and fix the transcription; never change the formulas.
3. Run `python3 builtin_max_size.py measurements/example.ini` and show the output.
4. Tell the user the kit is ready and that they can now send a measurement: text, a
   table, or a photo of a sketch with the numbers on it. From here on follow `CLAUDE.md`.

## Files

| File | What it is |
|---|---|
| `builtin_max_size.py` | the calculation and the command line |
| `measurement-template.ini` | the empty template, 51 fields |
| `measurements/example.ini` | a filled-in example with made-up numbers |
| `test_builtin_max_size.py` | 19 tests, including a check of the field map |
| `reference-cases.json` | the example and six random measurements with the numbers recorded from the original spreadsheet |
| `field-map.svg` | where each field sits on the opening (open it in a browser) |
| `README.md` | what it computes and how |
| `CLAUDE.md` | the workflow for the agent; AGENTS.md is an exact copy for Codex |
| `AGENTS.md` | an exact copy of `CLAUDE.md`, for Codex; not shown below, copy it |

### `builtin_max_size.py`

````python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
Maximum built-in size (width, depth, height) from a site measurement of an opening.

Usage:
  python3 builtin_max_size.py measurements/example.ini     # compute from a filled-in template
  python3 builtin_max_size.py measurements/example.ini --json
  python3 builtin_max_size.py --template                   # print an empty template

Input: 51 fields, sizes in mm, angles in degrees. A blank field means "not measured":
it counts as 0 in sums and is skipped in min/max, exactly like an empty spreadsheet
cell. The formulas are the method, not a draft: do not "improve" them, the numbers are
meant to match the spreadsheet they were ported from.
"""
import argparse
import configparser
import json
import math
import sys
from collections import OrderedDict, namedtuple

LEVELS = ("top", "middle", "bottom")
SIDES = ("left", "right")
EDGES = ("left_front", "left_back", "right_back", "right_front")
POSITIONS = ("left", "center", "right")
POINTS = ("left_front", "left_back", "center_front", "center_back", "right_front", "right_back")

Field = namedtuple("Field", "section key label")

SECTIONS = OrderedDict([
    ("object", "Label for this measurement. Not used in the calculation."),
    ("width", "Opening width at three levels: along the back wall and across the front plane."),
    ("depth", "Depth along the left and right side walls at three levels."),
    ("angle", "Angle between each side wall and the back wall, left and right, at three levels. "
              "90 = square, more than 90 = the side wall opens up toward the front."),
    ("edge_plumb", "How far each of the four vertical edges of the opening (left/right x "
                   "front/back) is out of plumb, at three levels."),
    ("back_wall_plumb", "How far the back wall is out of plumb at left, center and right, "
                        "at three levels."),
    ("height", "Height at six points: left, center, right x front and back."),
    ("floor_slope", "Floor slope at the same six points."),
])
META_KEYS = ("name", "date", "note")


def _fields():
    f = []
    for plane in ("back", "front"):
        for lv in LEVELS:
            f.append(Field("width", f"{plane}_{lv}", f"width along the {plane}, {lv}"))
    for side in SIDES:
        for lv in LEVELS:
            f.append(Field("depth", f"{side}_{lv}", f"depth {side}, {lv}"))
    for side in SIDES:
        for lv in LEVELS:
            f.append(Field("angle", f"{side}_{lv}", f"angle {side}, {lv}"))
    for edge in EDGES:
        for lv in LEVELS:
            f.append(Field("edge_plumb", f"{edge}_{lv}", f"edge {edge.replace('_', ' ')}, {lv}"))
    for pos in POSITIONS:
        for lv in LEVELS:
            f.append(Field("back_wall_plumb", f"{pos}_{lv}", f"back wall {pos}, {lv}"))
    for p in POINTS:
        f.append(Field("height", p, f"height {p.replace('_', ' ')}"))
    for p in POINTS:
        f.append(Field("floor_slope", p, f"floor slope {p.replace('_', ' ')}"))
    return f


FIELDS = _fields()
FIELD_INDEX = {(f.section, f.key): f for f in FIELDS}
assert len(FIELDS) == 51 and len(FIELD_INDEX) == 51

# Names of the intermediate values, in the order reference-cases.json stores them.
OUTPUT_NAMES = (
    ["width", "depth", "height"]
    + [f"level_max_{lv}" for lv in LEVELS]
    + ["front_max_with_edges", "back_max_with_edges"]
    + [f"front_offset_left_{lv}" for lv in LEVELS] + [f"front_offset_right_{lv}" for lv in LEVELS]
    + [f"back_offset_left_{lv}" for lv in LEVELS] + [f"back_offset_right_{lv}" for lv in LEVELS]
    + [f"edge_dev_{e}_{lv}" for e in ("left_front", "right_front", "left_back", "right_back") for lv in LEVELS]
    + ["back_wall_plumb_max", "depth_max_left", "depth_max_right", "depth_min_left", "depth_min_right"]
)


# ------------------------------------------------------------------ blank-cell semantics
def n(v):
    """A blank field is 0 in arithmetic and comparisons."""
    return 0.0 if v is None else float(v)


def xmax(*vals):
    """MAX over direct references: blanks are skipped, all blank = 0."""
    vs = [float(v) for v in vals if v is not None]
    return max(vs) if vs else 0.0


def xmin(*vals):
    vs = [float(v) for v in vals if v is not None]
    return min(vs) if vs else 0.0


def xround(x):
    """ROUND(x, 0) as spreadsheets do it: halves go away from zero, unlike Python's round()."""
    return math.copysign(math.floor(abs(x) + 0.5), x) if x else 0.0


def deviation(v, trio):
    """How far this edge reading sits beyond the edge's smallest reading over the three
    levels: v - MIN(trio), negative becomes 0. One reading only gives 0."""
    d = n(v) - xmin(*trio)
    return 0.0 if d < 0 else d


# ------------------------------------------------------------------ the calculation
def calc(m):
    """m: {(section, key): number | None}. Returns the results plus every intermediate
    value, so the user can see where each number came from."""
    g = lambda s, k: m.get((s, k))
    r = OrderedDict()

    # Offsets from the wall angles. front_offset: how much wider the opening gets toward
    # the front where a side wall opens up (angle > 90); it enters the width. back_offset:
    # how much wider the back is where a side wall closes in (angle < 90); shown only.
    front_off = {s: {} for s in SIDES}
    back_off = {s: {} for s in SIDES}
    for side in SIDES:
        for lv in LEVELS:
            d = n(g("depth", f"{side}_{lv}"))
            a = n(g("angle", f"{side}_{lv}"))
            front_off[side][lv] = xround(d * math.sin(math.radians(a - 90))) if a > 90 else 0.0
            back_off[side][lv] = xround(d * math.cos(math.radians(a))) if (a < 90 and a != 0) else 0.0
    r["front_offset"] = front_off
    r["back_offset"] = back_off

    # Largest width at each level: MAX(back, front, back + both front offsets),
    # only when the back width at that level was measured.
    level_max = OrderedDict()
    level_parts = OrderedDict()
    for lv in LEVELS:
        wb = g("width", f"back_{lv}")
        wf = g("width", f"front_{lv}")
        summed = n(wb) + front_off["left"][lv] + front_off["right"][lv]
        level_parts[lv] = (wb, wf, summed)
        level_max[lv] = 0.0 if n(wb) == 0 else xmax(wb, wf, summed)
    r["level_max"] = level_max
    r["level_parts"] = level_parts

    # Each vertical edge, relative to its own smallest reading.
    dev = OrderedDict()
    for edge in EDGES:
        trio = [g("edge_plumb", f"{edge}_{lv}") for lv in LEVELS]
        dev[edge] = OrderedDict((lv, deviation(v, trio)) for lv, v in zip(LEVELS, trio))
    r["edge_dev"] = dev

    # Two more candidates. Front: the largest front width plus BOTH front edges at that
    # level. Back: the largest back width plus the LARGER of the two back edges. On a tie
    # the first level from the top wins. This asymmetry is part of the method.
    C = [n(g("width", f"front_{lv}")) for lv in LEVELS]
    k = C.index(max(C))
    r["front_level"] = LEVELS[k]
    r["front_parts"] = (C[k], dev["left_front"][LEVELS[k]], dev["right_front"][LEVELS[k]])
    r["front_max_with_edges"] = sum(r["front_parts"])
    H = [n(g("width", f"back_{lv}")) for lv in LEVELS]
    k = H.index(max(H))
    r["back_level"] = LEVELS[k]
    r["back_parts"] = (H[k], dev["left_back"][LEVELS[k]], dev["right_back"][LEVELS[k]])
    r["back_max_with_edges"] = H[k] + max(r["back_parts"][1], r["back_parts"][2])

    r["width"] = max(*level_max.values(), r["front_max_with_edges"], r["back_max_with_edges"])

    # Depth. Back wall out of plumb by 10 mm or more anywhere: take the largest depth,
    # otherwise the smallest.
    tilts = [g("back_wall_plumb", f"{pos}_{lv}") for pos in POSITIONS for lv in LEVELS]
    r["back_wall_plumb_max"] = xmax(*tilts)
    dl = [g("depth", f"left_{lv}") for lv in LEVELS]
    dr = [g("depth", f"right_{lv}") for lv in LEVELS]
    r["depth_max_left"], r["depth_max_right"] = xmax(*dl), xmax(*dr)
    r["depth_min_left"], r["depth_min_right"] = xmin(*dl), xmin(*dr)
    r["depth_takes_max"] = r["back_wall_plumb_max"] >= 10
    r["depth"] = (max(r["depth_max_left"], r["depth_max_right"]) if r["depth_takes_max"]
                  else min(r["depth_min_left"], r["depth_min_right"]))

    # Height: the largest of (height + floor slope) over the six points.
    ceiling = OrderedDict()
    for p in POINTS:
        ceiling[p] = (n(g("height", p)), n(g("floor_slope", p)))
    r["ceiling_points"] = ceiling
    r["height"] = max(h + s for h, s in ceiling.values())
    return r


def outputs(r):
    """The results and intermediates as flat numbers, in OUTPUT_NAMES order.
    A result of 0 is reported as None, the way an empty result cell shows as blank."""
    o = OrderedDict()
    for name in ("width", "depth", "height"):
        o[name] = None if r[name] == 0 else r[name]
    for lv in LEVELS:
        o[f"level_max_{lv}"] = r["level_max"][lv]
    o["front_max_with_edges"] = r["front_max_with_edges"]
    o["back_max_with_edges"] = r["back_max_with_edges"]
    for side in SIDES:
        for lv in LEVELS:
            o[f"front_offset_{side}_{lv}"] = r["front_offset"][side][lv]
    for side in SIDES:
        for lv in LEVELS:
            o[f"back_offset_{side}_{lv}"] = r["back_offset"][side][lv]
    for edge in ("left_front", "right_front", "left_back", "right_back"):
        for lv in LEVELS:
            o[f"edge_dev_{edge}_{lv}"] = r["edge_dev"][edge][lv]
    for name in ("back_wall_plumb_max", "depth_max_left", "depth_max_right",
                 "depth_min_left", "depth_min_right"):
        o[name] = r[name]
    assert list(o) == list(OUTPUT_NAMES), set(o) ^ set(OUTPUT_NAMES)
    return o


# ------------------------------------------------------------------ input
def norm_key(s):
    return s.strip().lower()


def parse_number(raw, where):
    """'' -> None. Accepts a comma as the decimal separator and trailing 'mm' / 'deg' / '°'."""
    if raw is None:
        return None
    s = str(raw).strip().lower()
    for junk in ("mm", "deg", "°", " "):
        s = s.replace(junk, "")
    if s in ("", "-", "—"):
        return None
    s = s.replace(",", ".")
    try:
        v = float(s)
    except ValueError:
        raise ValueError(f"{where}: not a number: {raw!r}")
    if not (0 <= v <= 99999):
        raise ValueError(f"{where}: only 0 to 99999 is accepted, got {raw!r}")
    return v


def read_ini(path):
    cp = configparser.ConfigParser(interpolation=None, allow_no_value=True,
                                   delimiters=("=",), comment_prefixes=("#", ";"),
                                   inline_comment_prefixes=("#", ";"))
    cp.optionxform = norm_key
    with open(path, encoding="utf-8") as fh:
        cp.read_file(fh)
    values, meta, problems = {}, {}, []
    for sec in cp.sections():
        s = norm_key(sec)
        if s == "object":
            meta = {k: (v or "").strip() for k, v in cp.items(sec)}
            continue
        if s not in SECTIONS:
            problems.append(f"unknown section [{sec}]")
            continue
        for key, raw in cp.items(sec):
            if (s, key) not in FIELD_INDEX:
                problems.append(f"unknown field [{sec}] {key}")
                continue
            try:
                values[(s, key)] = parse_number(raw, f"[{sec}] {key}")
            except ValueError as e:
                problems.append(str(e))
    return values, meta, problems


def template_text():
    lines = ["# Site measurement of an opening for a built-in: template for builtin_max_size.py",
             "# Sizes in mm, angles in degrees. Blank = not measured (an empty spreadsheet cell).",
             "# Keep the section and field names as they are. Decimals with a point or a comma.",
             "# Where each field sits on the opening: field-map.svg.", ""]
    for sec, comment in SECTIONS.items():
        lines.append(f"[{sec}]")
        lines.append(f"# {comment}")
        if sec == "object":
            lines += [f"{k} =" for k in META_KEYS]
        else:
            lines += [f"{f.key} =" for f in FIELDS if f.section == sec]
        lines.append("")
    return "\n".join(lines)


# ------------------------------------------------------------------ output
def fmt(x, dash="-"):
    if x is None:
        return dash
    x = float(x)
    if x == int(x):
        return str(int(x))
    return f"{x:.2f}".rstrip("0").rstrip(".")


def report(m, r, meta, blanks, source):
    L = []
    title = meta.get("name") or source
    L.append(f"Measurement: {title}" + (f", {meta['date']}" if meta.get("date") else ""))
    if meta.get("note"):
        L.append(f"Note: {meta['note']}")
    if blanks:
        L.append(f"BLANK FIELDS ({len(blanks)}): " + ", ".join(blanks))
    L.append("")
    L.append("MAXIMUM BUILT-IN SIZE")
    for name in ("width", "depth", "height"):
        val = r[name]
        L.append(f"  {name.capitalize():7s} {fmt(None if val == 0 else val):>8s} mm")
    L.append("")
    L.append("Width candidates, the largest wins:")
    for lv in LEVELS:
        wb, wf, summed = r["level_parts"][lv]
        xl, xr = r["front_offset"]["left"][lv], r["front_offset"]["right"][lv]
        if n(wb) == 0:
            L.append(f"  {lv:7s} back width not measured, level skipped")
            continue
        L.append(f"  {lv:7s} back {fmt(wb)}, front {fmt(wf)}, back + offsets {fmt(wb)}+{fmt(xl)}+{fmt(xr)}"
                 f" = {fmt(summed)}  -> {fmt(r['level_max'][lv])}")
    c = r["front_parts"]
    L.append(f"  front max ({r['front_level']}) {fmt(c[0])} + edge dev left {fmt(c[1])}"
             f" + right {fmt(c[2])} = {fmt(r['front_max_with_edges'])}")
    h = r["back_parts"]
    L.append(f"  back max ({r['back_level']}) {fmt(h[0])} + max(edge dev left {fmt(h[1])},"
             f" right {fmt(h[2])}) = {fmt(r['back_max_with_edges'])}")
    bo = r["back_offset"]
    if any(v for side in SIDES for v in bo[side].values()):
        L.append("  back offsets (back wider than front where angle < 90, for information only): left "
                 + "/".join(fmt(bo["left"][lv]) for lv in LEVELS) + ", right "
                 + "/".join(fmt(bo["right"][lv]) for lv in LEVELS))
    L.append("")
    rule = (">= 10 -> take the largest depth" if r["depth_takes_max"]
            else "< 10 -> take the smallest depth")
    L.append(f"Depth: back wall out of plumb at most {fmt(r['back_wall_plumb_max'])} mm {rule}")
    L.append(f"  left: max {fmt(r['depth_max_left'])} / min {fmt(r['depth_min_left'])};"
             f"  right: max {fmt(r['depth_max_right'])} / min {fmt(r['depth_min_right'])}")
    L.append("")
    L.append("Height: the largest of (height + floor slope) over six points")
    L.append("  " + ";  ".join(f"{p} {fmt(h)}+{fmt(s)}={fmt(h + s)}"
                               for p, (h, s) in r["ceiling_points"].items()))
    return "\n".join(L)


def main(argv=None):
    ap = argparse.ArgumentParser(description=__doc__, formatter_class=argparse.RawDescriptionHelpFormatter)
    ap.add_argument("ini", nargs="?", help="filled-in measurement template (.ini)")
    ap.add_argument("--template", action="store_true", help="print an empty template")
    ap.add_argument("--json", action="store_true", help="results as JSON")
    a = ap.parse_args(argv)

    if a.template:
        print(template_text())
        return 0
    if not a.ini:
        ap.print_help()
        return 2
    values, meta, problems = read_ini(a.ini)
    if problems:
        print("INPUT ERRORS:\n  " + "\n  ".join(problems), file=sys.stderr)
        return 1
    blanks = [f"[{f.section}] {f.key}" for f in FIELDS if values.get((f.section, f.key)) is None]
    r = calc(values)
    if a.json:
        out = OrderedDict([("source", a.ini), ("object", meta), ("blank_fields", blanks),
                           ("width", None if r["width"] == 0 else r["width"]),
                           ("depth", None if r["depth"] == 0 else r["depth"]),
                           ("height", None if r["height"] == 0 else r["height"]),
                           ("details", outputs(r))])
        print(json.dumps(out, ensure_ascii=False, indent=2))
    else:
        print(report(values, r, meta, blanks, a.ini))
    return 0


if __name__ == "__main__":
    sys.exit(main())
````

### `measurement-template.ini`

````ini
# Site measurement of an opening for a built-in: template for builtin_max_size.py
# Sizes in mm, angles in degrees. Blank = not measured (an empty spreadsheet cell).
# Keep the section and field names as they are. Decimals with a point or a comma.
# Where each field sits on the opening: field-map.svg.

[object]
# Label for this measurement. Not used in the calculation.
name =
date =
note =

[width]
# Opening width at three levels: along the back wall and across the front plane.
back_top =
back_middle =
back_bottom =
front_top =
front_middle =
front_bottom =

[depth]
# Depth along the left and right side walls at three levels.
left_top =
left_middle =
left_bottom =
right_top =
right_middle =
right_bottom =

[angle]
# Angle between each side wall and the back wall, left and right, at three levels. 90 = square, more than 90 = the side wall opens up toward the front.
left_top =
left_middle =
left_bottom =
right_top =
right_middle =
right_bottom =

[edge_plumb]
# How far each of the four vertical edges of the opening (left/right x front/back) is out of plumb, at three levels.
left_front_top =
left_front_middle =
left_front_bottom =
left_back_top =
left_back_middle =
left_back_bottom =
right_back_top =
right_back_middle =
right_back_bottom =
right_front_top =
right_front_middle =
right_front_bottom =

[back_wall_plumb]
# How far the back wall is out of plumb at left, center and right, at three levels.
left_top =
left_middle =
left_bottom =
center_top =
center_middle =
center_bottom =
right_top =
right_middle =
right_bottom =

[height]
# Height at six points: left, center, right x front and back.
left_front =
left_back =
center_front =
center_back =
right_front =
right_back =

[floor_slope]
# Floor slope at the same six points.
left_front =
left_back =
center_front =
center_back =
right_front =
right_back =
````

### `measurements/example.ini`

````ini
# Example measurement with made-up numbers. Expected result: 1221 x 605 x 2607 mm.
# Sizes in mm, angles in degrees. Blank = not measured. Field positions: field-map.svg.

[object]
name = Example opening (made-up numbers)
date = 2026-10-04
note = Shipped with the kit to check the install.

[width]
# Along the back wall and across the front plane, three levels.
back_top = 1200
back_middle = 1198
back_bottom = 1203
front_top = 1210
front_middle = 1206
front_bottom = 1207

[depth]
left_top = 600
left_middle = 598
left_bottom = 602
right_top = 605
right_middle = 603
right_bottom = 601

[angle]
left_top = 90
left_middle = 91
left_bottom = 89
right_top = 92
right_middle = 90
right_bottom = 90

[edge_plumb]
left_front_top = 2
left_front_middle = 0
left_front_bottom = 1
left_back_top = 0
left_back_middle = 1
left_back_bottom = 3
right_back_top = 1
right_back_middle = 0
right_back_bottom = 0
right_front_top = 0
right_front_middle = 2
right_front_bottom = 4

[back_wall_plumb]
left_top = 3
left_middle = 5
left_bottom = 12
center_top = 0
center_middle = 2
center_bottom = 4
right_top = 1
right_middle = 1
right_bottom = 6

[height]
left_front = 2600
left_back = 2605
center_front = 2598
center_back = 2601
right_front = 2603
right_back = 2599

[floor_slope]
left_front = 0
left_back = 2
center_front = 1
center_back = 3
right_front = 0
right_back = 4
````

### `test_builtin_max_size.py`

````python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
Tests for builtin_max_size.py.

  python3 test_builtin_max_size.py

Two groups: hand-written cases (blank-cell semantics, the 10 mm depth rule, wall angles,
rounding, the front/back asymmetry) and a replay of reference-cases.json, numbers
recorded from the spreadsheet the method was ported from.
"""
import json
import tempfile
import unittest
from pathlib import Path

import builtin_max_size as bms

HERE = Path(__file__).resolve().parent


def meas(**sections):
    """meas(width={'back_top': 1200}, ...) -> {(section, key): value}; everything else blank."""
    m = {(f.section, f.key): None for f in bms.FIELDS}
    for sec, kv in sections.items():
        for k, v in kv.items():
            assert (sec, k) in m, (sec, k)
            m[(sec, k)] = v
    return m


class BlankCellSemantics(unittest.TestCase):
    def test_round_half_away_from_zero(self):
        self.assertEqual(bms.xround(2.5), 3)
        self.assertEqual(bms.xround(0.5), 1)
        self.assertEqual(bms.xround(52.29), 52)
        self.assertEqual(bms.xround(0), 0)

    def test_min_max_skip_blanks(self):
        self.assertEqual(bms.xmin(5, None, None), 5)
        self.assertEqual(bms.xmin(None, None, None), 0)
        self.assertEqual(bms.xmax(None, 3, None), 3)

    def test_edge_with_a_single_reading_deviates_zero(self):
        # top = 5, middle and bottom blank: MIN skips blanks, 5 - 5 = 0
        r = bms.calc(meas(width={"back_top": 1000, "front_top": 1010},
                          edge_plumb={"left_front_top": 5}))
        self.assertEqual(r["edge_dev"]["left_front"]["top"], 0)
        self.assertEqual(r["front_max_with_edges"], 1010)

    def test_all_blank_gives_empty_results(self):
        o = bms.outputs(bms.calc(meas()))
        self.assertIsNone(o["width"])
        self.assertIsNone(o["depth"])
        self.assertIsNone(o["height"])

    def test_depth_with_one_side_blank_is_empty(self):
        # no right depths: MIN(600, 0) = 0 -> shown as blank
        r = bms.calc(meas(depth={"left_top": 600, "left_middle": 600, "left_bottom": 600}))
        self.assertEqual(r["depth"], 0)
        self.assertIsNone(bms.outputs(r)["depth"])


class WidthLogic(unittest.TestCase):
    def test_angle_over_90_adds_front_offset(self):
        r = bms.calc(meas(width={"back_top": 1000, "front_top": 1010},
                          depth={"left_top": 600}, angle={"left_top": 95}))
        self.assertEqual(r["front_offset"]["left"]["top"], 52)   # ROUND(600 * SIN(5 deg)) = 52.29 -> 52
        self.assertEqual(r["level_max"]["top"], 1052)
        self.assertEqual(r["width"], 1052)

    def test_angle_under_90_is_informational(self):
        r = bms.calc(meas(width={"back_top": 1000, "front_top": 990},
                          depth={"left_top": 600}, angle={"left_top": 85}))
        self.assertEqual(r["back_offset"]["left"]["top"], 52)    # ROUND(600 * COS(85 deg))
        self.assertEqual(r["front_offset"]["left"]["top"], 0)
        self.assertEqual(r["width"], 1000)

    def test_angle_zero_or_blank_gives_no_offsets(self):
        for a in (0, None):
            r = bms.calc(meas(width={"back_top": 1000}, depth={"left_top": 600},
                              angle={"left_top": a}))
            self.assertEqual(r["front_offset"]["left"]["top"], 0)
            self.assertEqual(r["back_offset"]["left"]["top"], 0)

    def test_level_without_back_width_is_skipped(self):
        r = bms.calc(meas(width={"front_top": 1500, "back_bottom": 1000, "front_bottom": 1001}))
        self.assertEqual(r["level_max"]["top"], 0)
        # but the front candidate still sees 1500
        self.assertEqual(r["front_max_with_edges"], 1500)
        self.assertEqual(r["width"], 1500)

    def test_front_sums_both_edges_back_takes_the_larger(self):
        m = meas(width={"back_top": 1000, "front_top": 1000},
                 edge_plumb={"left_front_top": 4, "left_front_bottom": 0,
                             "right_front_top": 3, "right_front_bottom": 0,
                             "left_back_top": 4, "left_back_bottom": 0,
                             "right_back_top": 3, "right_back_bottom": 0})
        r = bms.calc(m)
        self.assertEqual(r["front_max_with_edges"], 1007)   # 1000 + 4 + 3
        self.assertEqual(r["back_max_with_edges"], 1004)    # 1000 + max(4, 3)
        self.assertEqual(r["width"], 1007)

    def test_first_level_from_the_top_wins_a_tie(self):
        m = meas(width={"front_top": 1000, "front_middle": 1000, "front_bottom": 1000},
                 edge_plumb={"left_front_top": 0, "left_front_middle": 9, "left_front_bottom": 0})
        r = bms.calc(m)
        self.assertEqual(r["front_level"], "top")
        self.assertEqual(r["front_max_with_edges"], 1000)


class DepthHeight(unittest.TestCase):
    def test_depth_rule_at_10_mm(self):
        base = dict(depth={"left_top": 600, "left_middle": 598, "left_bottom": 602,
                           "right_top": 605, "right_middle": 603, "right_bottom": 601})
        r = bms.calc(meas(back_wall_plumb={"center_top": 9.9}, **base))
        self.assertEqual(r["depth"], 598)
        r = bms.calc(meas(back_wall_plumb={"center_top": 10}, **base))
        self.assertEqual(r["depth"], 605)

    def test_height_is_max_of_height_plus_floor_slope(self):
        r = bms.calc(meas(height={"left_front": 2600, "right_back": 2590},
                          floor_slope={"right_back": 15}))
        self.assertEqual(r["height"], 2605)


class InputFormats(unittest.TestCase):
    def test_template_parses_to_all_blank(self):
        values, meta, problems = bms.read_ini(HERE / "measurement-template.ini")
        self.assertEqual(problems, [])
        self.assertEqual(len(values), 51)
        self.assertTrue(all(v is None for v in values.values()))

    def test_example_ini(self):
        values, meta, problems = bms.read_ini(HERE / "measurements" / "example.ini")
        self.assertEqual(problems, [])
        r = bms.calc(values)
        self.assertEqual((r["width"], r["depth"], r["height"]), (1221, 605, 2607))
        self.assertEqual(r["level_max"], {"top": 1221, "middle": 1208, "bottom": 1207})
        self.assertEqual(r["front_max_with_edges"], 1212)
        self.assertEqual(r["back_max_with_edges"], 1206)
        self.assertEqual(r["back_offset"]["left"]["bottom"], 11)

    def test_number_parsing(self):
        self.assertEqual(bms.parse_number("1 200,5 mm", "x"), 1200.5)
        self.assertEqual(bms.parse_number("90°", "x"), 90)
        self.assertEqual(bms.parse_number("90 deg", "x"), 90)
        self.assertIsNone(bms.parse_number("", "x"))
        with self.assertRaises(ValueError):
            bms.parse_number("-1", "x")
        with self.assertRaises(ValueError):
            bms.parse_number("abc", "x")

    def test_unknown_field_is_reported(self):
        with tempfile.NamedTemporaryFile("w", suffix=".ini", delete=False, encoding="utf-8") as fh:
            fh.write("[width]\nback_top = 1000\nback_tops = 5\n[attic]\nx = 1\n")
        values, meta, problems = bms.read_ini(fh.name)
        self.assertEqual(len(problems), 2)


class FieldMap(unittest.TestCase):
    def test_field_map_is_well_formed_and_complete(self):
        import xml.etree.ElementTree as ET
        root = ET.parse(HERE / "field-map.svg").getroot()
        self.assertTrue(root.tag.endswith("svg"))
        texts = {el.text for el in root.iter() if el.text}
        for f in bms.FIELDS:
            self.assertIn(f.key, texts, f"{f.section} {f.key} is missing from the field map")
        boxes = [el for el in root.iter() if el.tag.endswith("rect") and el.get("rx") == "4"]
        self.assertEqual(len(boxes), 51)


class ReferenceCases(unittest.TestCase):
    """Replay of numbers recorded from the original spreadsheet."""

    def test_reference_cases(self):
        cases = json.loads((HERE / "reference-cases.json").read_text(encoding="utf-8"))
        self.assertGreater(len(cases), 0)
        for case in cases:
            with self.subTest(case["label"]):
                m = {tuple(k.split(".", 1)): v for k, v in case["input"].items()}
                got = bms.outputs(bms.calc(m))
                for name, want in case["expected"].items():
                    if want is None:
                        self.assertIsNone(got[name], name)
                    else:
                        self.assertIsNotNone(got[name], name)
                        self.assertAlmostEqual(got[name], want, places=6, msg=name)


if __name__ == "__main__":
    unittest.main(verbosity=1)
````

### `reference-cases.json`

````json
[
 {
  "label": "example.ini",
  "input": {
   "width.back_top": 1200, "width.back_middle": 1198, "width.back_bottom": 1203, "width.front_top": 1210, "width.front_middle": 1206, "width.front_bottom": 1207,
   "depth.left_top": 600, "depth.left_middle": 598, "depth.left_bottom": 602, "depth.right_top": 605, "depth.right_middle": 603, "depth.right_bottom": 601,
   "angle.left_top": 90, "angle.left_middle": 91, "angle.left_bottom": 89, "angle.right_top": 92, "angle.right_middle": 90, "angle.right_bottom": 90,
   "edge_plumb.left_front_top": 2, "edge_plumb.left_front_middle": 0, "edge_plumb.left_front_bottom": 1, "edge_plumb.left_back_top": 0, "edge_plumb.left_back_middle": 1, "edge_plumb.left_back_bottom": 3, "edge_plumb.right_back_top": 1, "edge_plumb.right_back_middle": 0, "edge_plumb.right_back_bottom": 0, "edge_plumb.right_front_top": 0, "edge_plumb.right_front_middle": 2, "edge_plumb.right_front_bottom": 4,
   "back_wall_plumb.left_top": 3, "back_wall_plumb.left_middle": 5, "back_wall_plumb.left_bottom": 12, "back_wall_plumb.center_top": 0, "back_wall_plumb.center_middle": 2, "back_wall_plumb.center_bottom": 4, "back_wall_plumb.right_top": 1, "back_wall_plumb.right_middle": 1, "back_wall_plumb.right_bottom": 6,
   "height.left_front": 2600, "height.left_back": 2605, "height.center_front": 2598, "height.center_back": 2601, "height.right_front": 2603, "height.right_back": 2599,
   "floor_slope.left_front": 0, "floor_slope.left_back": 2, "floor_slope.center_front": 1, "floor_slope.center_back": 3, "floor_slope.right_front": 0, "floor_slope.right_back": 4
  },
  "expected": {
   "width": 1221, "depth": 605, "height": 2607,
   "level_max_top": 1221, "level_max_middle": 1208, "level_max_bottom": 1207, "front_max_with_edges": 1212, "back_max_with_edges": 1206,
   "front_offset_left_top": 0, "front_offset_left_middle": 10, "front_offset_left_bottom": 0, "front_offset_right_top": 21, "front_offset_right_middle": 0, "front_offset_right_bottom": 0,
   "back_offset_left_top": 0, "back_offset_left_middle": 0, "back_offset_left_bottom": 11, "back_offset_right_top": 0, "back_offset_right_middle": 0, "back_offset_right_bottom": 0,
   "edge_dev_left_front_top": 2, "edge_dev_left_front_middle": 0, "edge_dev_left_front_bottom": 1, "edge_dev_right_front_top": 0, "edge_dev_right_front_middle": 2, "edge_dev_right_front_bottom": 4,
   "edge_dev_left_back_top": 0, "edge_dev_left_back_middle": 1, "edge_dev_left_back_bottom": 3, "edge_dev_right_back_top": 1, "edge_dev_right_back_middle": 0, "edge_dev_right_back_bottom": 0,
   "back_wall_plumb_max": 12, "depth_max_left": 602, "depth_max_right": 605, "depth_min_left": 598, "depth_min_right": 601
  }
 },
 {
  "label": "random 1, all fields",
  "input": {
   "width.back_top": 2744, "width.back_middle": 2721, "width.back_bottom": 2733, "width.front_top": 2721, "width.front_middle": 2745, "width.front_bottom": 2744,
   "depth.left_top": 681, "depth.left_middle": 677, "depth.left_bottom": 682, "depth.right_top": 685, "depth.right_middle": 677, "depth.right_bottom": 686,
   "angle.left_top": 90, "angle.left_middle": 95, "angle.left_bottom": 90, "angle.right_top": 100, "angle.right_middle": 90.5, "angle.right_bottom": 95,
   "edge_plumb.left_front_top": 8, "edge_plumb.left_front_middle": 5, "edge_plumb.left_front_bottom": 0, "edge_plumb.left_back_top": 1, "edge_plumb.left_back_middle": 12, "edge_plumb.left_back_bottom": 12, "edge_plumb.right_back_top": 1, "edge_plumb.right_back_middle": 3, "edge_plumb.right_back_bottom": 5, "edge_plumb.right_front_top": 2, "edge_plumb.right_front_middle": 0, "edge_plumb.right_front_bottom": 2,
   "back_wall_plumb.left_top": 3, "back_wall_plumb.left_middle": 10, "back_wall_plumb.left_bottom": 10, "back_wall_plumb.center_top": 20, "back_wall_plumb.center_middle": 7, "back_wall_plumb.center_bottom": 9, "back_wall_plumb.right_top": 11, "back_wall_plumb.right_middle": 10, "back_wall_plumb.right_bottom": 2,
   "height.left_front": 2608, "height.left_back": 2607, "height.center_front": 2605, "height.center_back": 2592, "height.right_front": 2606, "height.right_back": 2599,
   "floor_slope.left_front": 8, "floor_slope.left_back": 5, "floor_slope.center_front": 5, "floor_slope.center_back": 5, "floor_slope.right_front": 0, "floor_slope.right_back": 1
  },
  "expected": {
   "width": 2863, "depth": 686, "height": 2616,
   "level_max_top": 2863, "level_max_middle": 2786, "level_max_bottom": 2793, "front_max_with_edges": 2750, "back_max_with_edges": 2744,
   "front_offset_left_top": 0, "front_offset_left_middle": 59, "front_offset_left_bottom": 0, "front_offset_right_top": 119, "front_offset_right_middle": 6, "front_offset_right_bottom": 60,
   "back_offset_left_top": 0, "back_offset_left_middle": 0, "back_offset_left_bottom": 0, "back_offset_right_top": 0, "back_offset_right_middle": 0, "back_offset_right_bottom": 0,
   "edge_dev_left_front_top": 8, "edge_dev_left_front_middle": 5, "edge_dev_left_front_bottom": 0, "edge_dev_right_front_top": 2, "edge_dev_right_front_middle": 0, "edge_dev_right_front_bottom": 2,
   "edge_dev_left_back_top": 0, "edge_dev_left_back_middle": 11, "edge_dev_left_back_bottom": 11, "edge_dev_right_back_top": 0, "edge_dev_right_back_middle": 2, "edge_dev_right_back_bottom": 4,
   "back_wall_plumb_max": 20, "depth_max_left": 682, "depth_max_right": 686, "depth_min_left": 677, "depth_min_right": 677
  }
 },
 {
  "label": "random 2, some fields blank",
  "input": {
   "width.back_top": null, "width.back_middle": null, "width.back_bottom": 663, "width.front_top": 658, "width.front_middle": 675, "width.front_bottom": 675,
   "depth.left_top": 412, "depth.left_middle": 415, "depth.left_bottom": 413, "depth.right_top": 422, "depth.right_middle": 417, "depth.right_bottom": 419,
   "angle.left_top": null, "angle.left_middle": null, "angle.left_bottom": 88, "angle.right_top": 100, "angle.right_middle": 95, "angle.right_bottom": 95,
   "edge_plumb.left_front_top": 8, "edge_plumb.left_front_middle": 0, "edge_plumb.left_front_bottom": 5, "edge_plumb.left_back_top": null, "edge_plumb.left_back_middle": 0, "edge_plumb.left_back_bottom": 5, "edge_plumb.right_back_top": 3, "edge_plumb.right_back_middle": 1, "edge_plumb.right_back_bottom": 0, "edge_plumb.right_front_top": null, "edge_plumb.right_front_middle": 1, "edge_plumb.right_front_bottom": null,
   "back_wall_plumb.left_top": 5, "back_wall_plumb.left_middle": 10, "back_wall_plumb.left_bottom": 5, "back_wall_plumb.center_top": 7, "back_wall_plumb.center_middle": 0, "back_wall_plumb.center_bottom": 7, "back_wall_plumb.right_top": 1, "back_wall_plumb.right_middle": 2, "back_wall_plumb.right_bottom": 3,
   "height.left_front": 2823, "height.left_back": 2820, "height.center_front": 2826, "height.center_back": null, "height.right_front": 2828, "height.right_back": 2820,
   "floor_slope.left_front": 10, "floor_slope.left_back": 0, "floor_slope.center_front": 0, "floor_slope.center_back": 0, "floor_slope.right_front": 1, "floor_slope.right_back": 5
  },
  "expected": {
   "width": 700, "depth": 422, "height": 2833,
   "level_max_top": 0, "level_max_middle": 0, "level_max_bottom": 700, "front_max_with_edges": 675, "back_max_with_edges": 668,
   "front_offset_left_top": 0, "front_offset_left_middle": 0, "front_offset_left_bottom": 0, "front_offset_right_top": 73, "front_offset_right_middle": 36, "front_offset_right_bottom": 37,
   "back_offset_left_top": 0, "back_offset_left_middle": 0, "back_offset_left_bottom": 14, "back_offset_right_top": 0, "back_offset_right_middle": 0, "back_offset_right_bottom": 0,
   "edge_dev_left_front_top": 8, "edge_dev_left_front_middle": 0, "edge_dev_left_front_bottom": 5, "edge_dev_right_front_top": 0, "edge_dev_right_front_middle": 0, "edge_dev_right_front_bottom": 0,
   "edge_dev_left_back_top": 0, "edge_dev_left_back_middle": 0, "edge_dev_left_back_bottom": 5, "edge_dev_right_back_top": 3, "edge_dev_right_back_middle": 1, "edge_dev_right_back_bottom": 0,
   "back_wall_plumb_max": 10, "depth_max_left": 415, "depth_max_right": 422, "depth_min_left": 412, "depth_min_right": 417
  }
 },
 {
  "label": "random 3, edge case",
  "input": {
   "width.back_top": null, "width.back_middle": null, "width.back_bottom": 1180, "width.front_top": 1178, "width.front_middle": 1172, "width.front_bottom": 1174,
   "depth.left_top": 751, "depth.left_middle": 743, "depth.left_bottom": 740, "depth.right_top": 743, "depth.right_middle": 749, "depth.right_bottom": 747,
   "angle.left_top": 90, "angle.left_middle": 90, "angle.left_bottom": 85, "angle.right_top": 90.5, "angle.right_middle": 95, "angle.right_bottom": 100,
   "edge_plumb.left_front_top": 1, "edge_plumb.left_front_middle": 3, "edge_plumb.left_front_bottom": 2, "edge_plumb.left_back_top": 5, "edge_plumb.left_back_middle": 3, "edge_plumb.left_back_bottom": 3, "edge_plumb.right_back_top": 12, "edge_plumb.right_back_middle": 2, "edge_plumb.right_back_bottom": 2, "edge_plumb.right_front_top": 1, "edge_plumb.right_front_middle": 8, "edge_plumb.right_front_bottom": 1,
   "back_wall_plumb.left_top": 9, "back_wall_plumb.left_middle": 8, "back_wall_plumb.left_bottom": 10, "back_wall_plumb.center_top": 9, "back_wall_plumb.center_middle": 0, "back_wall_plumb.center_bottom": 0, "back_wall_plumb.right_top": 8, "back_wall_plumb.right_middle": 8, "back_wall_plumb.right_bottom": 0,
   "height.left_front": 2830, "height.left_back": 2838, "height.center_front": 2829, "height.center_back": 2839, "height.right_front": 2835, "height.right_back": 2838,
   "floor_slope.left_front": 1, "floor_slope.left_back": 2, "floor_slope.center_front": 3, "floor_slope.center_back": 3, "floor_slope.right_front": 1, "floor_slope.right_back": 5
  },
  "expected": {
   "width": 1310, "depth": 751, "height": 2843,
   "level_max_top": 0, "level_max_middle": 0, "level_max_bottom": 1310, "front_max_with_edges": 1178, "back_max_with_edges": 1180,
   "front_offset_left_top": 0, "front_offset_left_middle": 0, "front_offset_left_bottom": 0, "front_offset_right_top": 6, "front_offset_right_middle": 65, "front_offset_right_bottom": 130,
   "back_offset_left_top": 0, "back_offset_left_middle": 0, "back_offset_left_bottom": 64, "back_offset_right_top": 0, "back_offset_right_middle": 0, "back_offset_right_bottom": 0,
   "edge_dev_left_front_top": 0, "edge_dev_left_front_middle": 2, "edge_dev_left_front_bottom": 1, "edge_dev_right_front_top": 0, "edge_dev_right_front_middle": 7, "edge_dev_right_front_bottom": 0,
   "edge_dev_left_back_top": 2, "edge_dev_left_back_middle": 0, "edge_dev_left_back_bottom": 0, "edge_dev_right_back_top": 10, "edge_dev_right_back_middle": 0, "edge_dev_right_back_bottom": 0,
   "back_wall_plumb_max": 10, "depth_max_left": 751, "depth_max_right": 749, "depth_min_left": 740, "depth_min_right": 743
  }
 },
 {
  "label": "random 4, some fields blank",
  "input": {
   "width.back_top": 2773, "width.back_middle": null, "width.back_bottom": 2767, "width.front_top": null, "width.front_middle": 2757, "width.front_bottom": 2770,
   "depth.left_top": 662, "depth.left_middle": 664, "depth.left_bottom": 662, "depth.right_top": 663, "depth.right_middle": 668, "depth.right_bottom": 661,
   "angle.left_top": 100, "angle.left_middle": 85, "angle.left_bottom": 85, "angle.right_top": 90, "angle.right_middle": 95, "angle.right_bottom": 90,
   "edge_plumb.left_front_top": 2, "edge_plumb.left_front_middle": 2, "edge_plumb.left_front_bottom": 5, "edge_plumb.left_back_top": 12, "edge_plumb.left_back_middle": 12, "edge_plumb.left_back_bottom": null, "edge_plumb.right_back_top": 12, "edge_plumb.right_back_middle": 3, "edge_plumb.right_back_bottom": 3, "edge_plumb.right_front_top": 8, "edge_plumb.right_front_middle": 5, "edge_plumb.right_front_bottom": 2,
   "back_wall_plumb.left_top": null, "back_wall_plumb.left_middle": 3, "back_wall_plumb.left_bottom": 0, "back_wall_plumb.center_top": 9, "back_wall_plumb.center_middle": 9, "back_wall_plumb.center_bottom": 10, "back_wall_plumb.right_top": null, "back_wall_plumb.right_middle": 11, "back_wall_plumb.right_bottom": 0,
   "height.left_front": 2829, "height.left_back": 2837, "height.center_front": null, "height.center_back": 2829, "height.right_front": 2834, "height.right_back": 2835,
   "floor_slope.left_front": null, "floor_slope.left_back": null, "floor_slope.center_front": 0, "floor_slope.center_back": null, "floor_slope.right_front": 0, "floor_slope.right_back": 8
  },
  "expected": {
   "width": 2888, "depth": 668, "height": 2843,
   "level_max_top": 2888, "level_max_middle": 0, "level_max_bottom": 2770, "front_max_with_edges": 2773, "back_max_with_edges": 2782,
   "front_offset_left_top": 115, "front_offset_left_middle": 0, "front_offset_left_bottom": 0, "front_offset_right_top": 0, "front_offset_right_middle": 58, "front_offset_right_bottom": 0,
   "back_offset_left_top": 0, "back_offset_left_middle": 58, "back_offset_left_bottom": 58, "back_offset_right_top": 0, "back_offset_right_middle": 0, "back_offset_right_bottom": 0,
   "edge_dev_left_front_top": 0, "edge_dev_left_front_middle": 0, "edge_dev_left_front_bottom": 3, "edge_dev_right_front_top": 6, "edge_dev_right_front_middle": 3, "edge_dev_right_front_bottom": 0,
   "edge_dev_left_back_top": 0, "edge_dev_left_back_middle": 0, "edge_dev_left_back_bottom": 0, "edge_dev_right_back_top": 9, "edge_dev_right_back_middle": 0, "edge_dev_right_back_bottom": 0,
   "back_wall_plumb_max": 11, "depth_max_left": 664, "depth_max_right": 668, "depth_min_left": 662, "depth_min_right": 661
  }
 },
 {
  "label": "random 5, edge case",
  "input": {
   "width.back_top": null, "width.back_middle": null, "width.back_bottom": 2162.5, "width.front_top": 2173, "width.front_middle": 2174.5, "width.front_bottom": 2162.5,
   "depth.left_top": 387.5, "depth.left_middle": 387, "depth.left_bottom": 381.5, "depth.right_top": 386.5, "depth.right_middle": 393, "depth.right_bottom": 393,
   "angle.left_top": 90, "angle.left_middle": 90.5, "angle.left_bottom": 90, "angle.right_top": 90, "angle.right_middle": 85, "angle.right_bottom": 90.5,
   "edge_plumb.left_front_top": 3, "edge_plumb.left_front_middle": 8, "edge_plumb.left_front_bottom": 0, "edge_plumb.left_back_top": 5, "edge_plumb.left_back_middle": 8, "edge_plumb.left_back_bottom": 0, "edge_plumb.right_back_top": 1, "edge_plumb.right_back_middle": 0, "edge_plumb.right_back_bottom": 8, "edge_plumb.right_front_top": 3, "edge_plumb.right_front_middle": 0, "edge_plumb.right_front_bottom": 0,
   "back_wall_plumb.left_top": 9, "back_wall_plumb.left_middle": 5, "back_wall_plumb.left_bottom": 9, "back_wall_plumb.center_top": 1, "back_wall_plumb.center_middle": 10, "back_wall_plumb.center_bottom": 1, "back_wall_plumb.right_top": 0, "back_wall_plumb.right_middle": 25, "back_wall_plumb.right_bottom": 15,
   "height.left_front": 2682.5, "height.left_back": 2676.5, "height.center_front": 2682, "height.center_back": 2673, "height.right_front": 2680, "height.right_back": 2671,
   "floor_slope.left_front": 8, "floor_slope.left_back": 0, "floor_slope.center_front": 0, "floor_slope.center_back": 1, "floor_slope.right_front": 0, "floor_slope.right_back": 0
  },
  "expected": {
   "width": 2182.5, "depth": 393, "height": 2690.5,
   "level_max_top": 0, "level_max_middle": 0, "level_max_bottom": 2165.5, "front_max_with_edges": 2182.5, "back_max_with_edges": 2170.5,
   "front_offset_left_top": 0, "front_offset_left_middle": 3, "front_offset_left_bottom": 0, "front_offset_right_top": 0, "front_offset_right_middle": 0, "front_offset_right_bottom": 3,
   "back_offset_left_top": 0, "back_offset_left_middle": 0, "back_offset_left_bottom": 0, "back_offset_right_top": 0, "back_offset_right_middle": 34, "back_offset_right_bottom": 0,
   "edge_dev_left_front_top": 3, "edge_dev_left_front_middle": 8, "edge_dev_left_front_bottom": 0, "edge_dev_right_front_top": 3, "edge_dev_right_front_middle": 0, "edge_dev_right_front_bottom": 0,
   "edge_dev_left_back_top": 5, "edge_dev_left_back_middle": 8, "edge_dev_left_back_bottom": 0, "edge_dev_right_back_top": 1, "edge_dev_right_back_middle": 0, "edge_dev_right_back_bottom": 8,
   "back_wall_plumb_max": 25, "depth_max_left": 387.5, "depth_max_right": 393, "depth_min_left": 381.5, "depth_min_right": 386.5
  }
 },
 {
  "label": "random 6, all fields",
  "input": {
   "width.back_top": 2637, "width.back_middle": 2637, "width.back_bottom": 2655.5, "width.front_top": 2656, "width.front_middle": 2655, "width.front_bottom": 2638.5,
   "depth.left_top": 539, "depth.left_middle": 530, "depth.left_bottom": 539.5, "depth.right_top": 535, "depth.right_middle": 535.5, "depth.right_bottom": 541,
   "angle.left_top": 88, "angle.left_middle": 89.5, "angle.left_bottom": 89.5, "angle.right_top": 100, "angle.right_middle": 88, "angle.right_bottom": 100,
   "edge_plumb.left_front_top": 5, "edge_plumb.left_front_middle": 8, "edge_plumb.left_front_bottom": 3, "edge_plumb.left_back_top": 8, "edge_plumb.left_back_middle": 0, "edge_plumb.left_back_bottom": 1, "edge_plumb.right_back_top": 0, "edge_plumb.right_back_middle": 3, "edge_plumb.right_back_bottom": 5, "edge_plumb.right_front_top": 3, "edge_plumb.right_front_middle": 1, "edge_plumb.right_front_bottom": 1,
   "back_wall_plumb.left_top": 11, "back_wall_plumb.left_middle": 2, "back_wall_plumb.left_bottom": 15, "back_wall_plumb.center_top": 10, "back_wall_plumb.center_middle": 10, "back_wall_plumb.center_bottom": 20, "back_wall_plumb.right_top": 0, "back_wall_plumb.right_middle": 3, "back_wall_plumb.right_bottom": 3,
   "height.left_front": 2558.5, "height.left_back": 2563, "height.center_front": 2559, "height.center_back": 2562, "height.right_front": 2548, "height.right_back": 2549,
   "floor_slope.left_front": 5, "floor_slope.left_back": 2, "floor_slope.center_front": 1, "floor_slope.center_back": 2, "floor_slope.right_front": 2, "floor_slope.right_back": 0
  },
  "expected": {
   "width": 2749.5, "depth": 541, "height": 2565,
   "level_max_top": 2730, "level_max_middle": 2655, "level_max_bottom": 2749.5, "front_max_with_edges": 2660, "back_max_with_edges": 2660.5,
   "front_offset_left_top": 0, "front_offset_left_middle": 0, "front_offset_left_bottom": 0, "front_offset_right_top": 93, "front_offset_right_middle": 0, "front_offset_right_bottom": 94,
   "back_offset_left_top": 19, "back_offset_left_middle": 5, "back_offset_left_bottom": 5, "back_offset_right_top": 0, "back_offset_right_middle": 19, "back_offset_right_bottom": 0,
   "edge_dev_left_front_top": 2, "edge_dev_left_front_middle": 5, "edge_dev_left_front_bottom": 0, "edge_dev_right_front_top": 2, "edge_dev_right_front_middle": 0, "edge_dev_right_front_bottom": 0,
   "edge_dev_left_back_top": 8, "edge_dev_left_back_middle": 0, "edge_dev_left_back_bottom": 1, "edge_dev_right_back_top": 0, "edge_dev_right_back_middle": 3, "edge_dev_right_back_bottom": 5,
   "back_wall_plumb_max": 20, "depth_max_left": 539.5, "depth_max_right": 541, "depth_min_left": 530, "depth_min_right": 535
  }
 }
]
````

### `field-map.svg`

````xml
<svg xmlns='http://www.w3.org/2000/svg' width='1700' height='1400' viewBox='0 0 1700 1400'>
<rect width='1700' height='1400' fill='#ffffff'/>
<text x='850' y='48' font-family='Helvetica, Arial, sans-serif' font-size='22' font-weight='bold' fill='#222' text-anchor='middle'>Field map: where each measurement sits on the opening</text>
<text x='850' y='74' font-family='Helvetica, Arial, sans-serif' font-size='15' fill='#555' text-anchor='middle'>Looking into the opening from the front. Three levels: top, middle, bottom. Sizes in mm, angles in degrees.</text>
<line x1='200' y1='110' x2='30' y2='40' stroke='#000' stroke-width='4' stroke-linecap='round'/>
<line x1='1500' y1='110' x2='1670' y2='40' stroke='#000' stroke-width='4' stroke-linecap='round'/>
<line x1='200' y1='1010' x2='30' y2='1080' stroke='#000' stroke-width='4' stroke-linecap='round'/>
<line x1='1500' y1='1010' x2='1670' y2='1080' stroke='#000' stroke-width='4' stroke-linecap='round'/>
<rect x='430' y='230' width='840' height='660' fill='none' stroke='#000' stroke-width='5'/>
<line x1='200' y1='110' x2='430' y2='230' stroke='#000' stroke-width='4' stroke-linecap='round'/>
<line x1='1500' y1='110' x2='1270' y2='230' stroke='#000' stroke-width='4' stroke-linecap='round'/>
<line x1='200' y1='1010' x2='430' y2='890' stroke='#000' stroke-width='4' stroke-linecap='round'/>
<line x1='1500' y1='1010' x2='1270' y2='890' stroke='#000' stroke-width='4' stroke-linecap='round'/>
<rect x='200' y='110' width='1300' height='900' fill='none' stroke='#d32f2f' stroke-width='3' stroke-dasharray='14,10'/>
<line x1='200' y1='218' x2='430' y2='309' stroke='#1f5fbf' stroke-width='3' stroke-linecap='round'/>
<line x1='430' y1='309' x2='1270' y2='309' stroke='#1f5fbf' stroke-width='3' stroke-linecap='round'/>
<line x1='1270' y1='309' x2='1500' y2='218' stroke='#1f5fbf' stroke-width='3' stroke-linecap='round'/>
<line x1='200' y1='209' x2='200' y2='227' stroke='#1f5fbf' stroke-width='3' stroke-linecap='round'/>
<line x1='430' y1='300' x2='430' y2='318' stroke='#1f5fbf' stroke-width='3' stroke-linecap='round'/>
<line x1='1270' y1='300' x2='1270' y2='318' stroke='#1f5fbf' stroke-width='3' stroke-linecap='round'/>
<line x1='1500' y1='209' x2='1500' y2='227' stroke='#1f5fbf' stroke-width='3' stroke-linecap='round'/>
<rect x='740' y='277' width='98' height='22' rx='4' fill='#dbe8fb' stroke='#1f5fbf' stroke-width='1.5'/>
<text x='747' y='293' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#1f5fbf'>W</text>
<text x='763' y='293' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>back_top</text>
<rect x='862' y='277' width='105' height='22' rx='4' fill='#dbe8fb' stroke='#1f5fbf' stroke-width='1.5'/>
<text x='869' y='293' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#1f5fbf'>W</text>
<text x='884' y='293' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>front_top</text>
<line x1='844' y1='269' x2='856' y2='305' stroke='#1f5fbf' stroke-width='2' stroke-linecap='round'/>
<rect x='438' y='319' width='98' height='22' rx='4' fill='#e6d7f0' stroke='#6a1b9a' stroke-width='1.5'/>
<text x='445' y='335' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#6a1b9a'>B</text>
<text x='460' y='335' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>left_top</text>
<line x1='490' y1='359' x2='536' y2='341' stroke='#6a1b9a' stroke-width='1.2' stroke-dasharray='3,3'/>
<circle cx='490' cy='359' r='5' fill='#6a1b9a'/>
<rect x='794' y='319' width='113' height='22' rx='4' fill='#e6d7f0' stroke='#6a1b9a' stroke-width='1.5'/>
<text x='801' y='335' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#6a1b9a'>B</text>
<text x='816' y='335' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>center_top</text>
<line x1='850' y1='359' x2='794' y2='341' stroke='#6a1b9a' stroke-width='1.2' stroke-dasharray='3,3'/>
<circle cx='850' cy='359' r='5' fill='#6a1b9a'/>
<rect x='1157' y='319' width='105' height='22' rx='4' fill='#e6d7f0' stroke='#6a1b9a' stroke-width='1.5'/>
<text x='1164' y='335' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#6a1b9a'>B</text>
<text x='1179' y='335' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>right_top</text>
<line x1='1210' y1='359' x2='1262' y2='341' stroke='#6a1b9a' stroke-width='1.2' stroke-dasharray='3,3'/>
<circle cx='1210' cy='359' r='5' fill='#6a1b9a'/>
<rect x='438' y='277' width='98' height='22' rx='4' fill='#f8d7e3' stroke='#c2185b' stroke-width='1.5'/>
<text x='445' y='293' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#c2185b'>A</text>
<text x='460' y='293' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>left_top</text>
<rect x='1157' y='277' width='105' height='22' rx='4' fill='#f8d7e3' stroke='#c2185b' stroke-width='1.5'/>
<text x='1164' y='293' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#c2185b'>A</text>
<text x='1179' y='293' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>right_top</text>
<rect x='266' y='230' width='98' height='22' rx='4' fill='#d8ead9' stroke='#2e7d32' stroke-width='1.5'/>
<text x='273' y='245' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#2e7d32'>D</text>
<text x='288' y='245' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>left_top</text>
<rect x='1332' y='230' width='105' height='22' rx='4' fill='#d8ead9' stroke='#2e7d32' stroke-width='1.5'/>
<text x='1339' y='245' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#2e7d32'>D</text>
<text x='1355' y='245' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>right_top</text>
<rect x='206' y='258' width='143' height='22' rx='4' fill='#fde0cc' stroke='#e65100' stroke-width='1.5'/>
<text x='213' y='274' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#e65100'>E</text>
<text x='228' y='274' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>left_front_top</text>
<line x1='200' y1='218' x2='206' y2='258' stroke='#e65100' stroke-width='1.2' stroke-dasharray='3,3'/>
<circle cx='200' cy='218' r='5' fill='#e65100'/>
<rect x='288' y='373' width='136' height='22' rx='4' fill='#fde0cc' stroke='#e65100' stroke-width='1.5'/>
<text x='295' y='389' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#e65100'>E</text>
<text x='311' y='389' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>left_back_top</text>
<line x1='430' y1='309' x2='424' y2='373' stroke='#e65100' stroke-width='1.2' stroke-dasharray='3,3'/>
<circle cx='430' cy='309' r='5' fill='#e65100'/>
<rect x='1276' y='373' width='143' height='22' rx='4' fill='#fde0cc' stroke='#e65100' stroke-width='1.5'/>
<text x='1283' y='389' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#e65100'>E</text>
<text x='1298' y='389' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>right_back_top</text>
<line x1='1270' y1='309' x2='1276' y2='373' stroke='#e65100' stroke-width='1.2' stroke-dasharray='3,3'/>
<circle cx='1270' cy='309' r='5' fill='#e65100'/>
<rect x='1343' y='258' width='151' height='22' rx='4' fill='#fde0cc' stroke='#e65100' stroke-width='1.5'/>
<text x='1350' y='274' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#e65100'>E</text>
<text x='1365' y='274' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>right_front_top</text>
<line x1='1500' y1='218' x2='1494' y2='258' stroke='#e65100' stroke-width='1.2' stroke-dasharray='3,3'/>
<circle cx='1500' cy='218' r='5' fill='#e65100'/>
<line x1='200' y1='560' x2='430' y2='560' stroke='#1f5fbf' stroke-width='3' stroke-linecap='round'/>
<line x1='430' y1='560' x2='1270' y2='560' stroke='#1f5fbf' stroke-width='3' stroke-linecap='round'/>
<line x1='1270' y1='560' x2='1500' y2='560' stroke='#1f5fbf' stroke-width='3' stroke-linecap='round'/>
<line x1='200' y1='551' x2='200' y2='569' stroke='#1f5fbf' stroke-width='3' stroke-linecap='round'/>
<line x1='430' y1='551' x2='430' y2='569' stroke='#1f5fbf' stroke-width='3' stroke-linecap='round'/>
<line x1='1270' y1='551' x2='1270' y2='569' stroke='#1f5fbf' stroke-width='3' stroke-linecap='round'/>
<line x1='1500' y1='551' x2='1500' y2='569' stroke='#1f5fbf' stroke-width='3' stroke-linecap='round'/>
<rect x='718' y='528' width='120' height='22' rx='4' fill='#dbe8fb' stroke='#1f5fbf' stroke-width='1.5'/>
<text x='725' y='544' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#1f5fbf'>W</text>
<text x='740' y='544' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>back_middle</text>
<rect x='862' y='528' width='128' height='22' rx='4' fill='#dbe8fb' stroke='#1f5fbf' stroke-width='1.5'/>
<text x='869' y='544' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#1f5fbf'>W</text>
<text x='884' y='544' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>front_middle</text>
<line x1='844' y1='520' x2='856' y2='556' stroke='#1f5fbf' stroke-width='2' stroke-linecap='round'/>
<rect x='438' y='570' width='120' height='22' rx='4' fill='#e6d7f0' stroke='#6a1b9a' stroke-width='1.5'/>
<text x='445' y='586' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#6a1b9a'>B</text>
<text x='460' y='586' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>left_middle</text>
<line x1='490' y1='610' x2='438' y2='592' stroke='#6a1b9a' stroke-width='1.2' stroke-dasharray='3,3'/>
<circle cx='490' cy='610' r='5' fill='#6a1b9a'/>
<rect x='782' y='570' width='136' height='22' rx='4' fill='#e6d7f0' stroke='#6a1b9a' stroke-width='1.5'/>
<text x='789' y='586' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#6a1b9a'>B</text>
<text x='804' y='586' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>center_middle</text>
<line x1='850' y1='610' x2='782' y2='592' stroke='#6a1b9a' stroke-width='1.2' stroke-dasharray='3,3'/>
<circle cx='850' cy='610' r='5' fill='#6a1b9a'/>
<rect x='1134' y='570' width='128' height='22' rx='4' fill='#e6d7f0' stroke='#6a1b9a' stroke-width='1.5'/>
<text x='1141' y='586' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#6a1b9a'>B</text>
<text x='1156' y='586' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>right_middle</text>
<line x1='1210' y1='610' x2='1262' y2='592' stroke='#6a1b9a' stroke-width='1.2' stroke-dasharray='3,3'/>
<circle cx='1210' cy='610' r='5' fill='#6a1b9a'/>
<rect x='438' y='528' width='120' height='22' rx='4' fill='#f8d7e3' stroke='#c2185b' stroke-width='1.5'/>
<text x='445' y='544' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#c2185b'>A</text>
<text x='460' y='544' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>left_middle</text>
<rect x='1134' y='528' width='128' height='22' rx='4' fill='#f8d7e3' stroke='#c2185b' stroke-width='1.5'/>
<text x='1141' y='544' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#c2185b'>A</text>
<text x='1156' y='544' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>right_middle</text>
<rect x='255' y='526' width='120' height='22' rx='4' fill='#d8ead9' stroke='#2e7d32' stroke-width='1.5'/>
<text x='262' y='542' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#2e7d32'>D</text>
<text x='277' y='542' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>left_middle</text>
<rect x='1321' y='526' width='128' height='22' rx='4' fill='#d8ead9' stroke='#2e7d32' stroke-width='1.5'/>
<text x='1328' y='542' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#2e7d32'>D</text>
<text x='1343' y='542' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>right_middle</text>
<rect x='206' y='600' width='166' height='22' rx='4' fill='#fde0cc' stroke='#e65100' stroke-width='1.5'/>
<text x='213' y='616' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#e65100'>E</text>
<text x='228' y='616' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>left_front_middle</text>
<line x1='200' y1='560' x2='206' y2='600' stroke='#e65100' stroke-width='1.2' stroke-dasharray='3,3'/>
<circle cx='200' cy='560' r='5' fill='#e65100'/>
<rect x='266' y='624' width='158' height='22' rx='4' fill='#fde0cc' stroke='#e65100' stroke-width='1.5'/>
<text x='273' y='640' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#e65100'>E</text>
<text x='288' y='640' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>left_back_middle</text>
<line x1='430' y1='560' x2='424' y2='624' stroke='#e65100' stroke-width='1.2' stroke-dasharray='3,3'/>
<circle cx='430' cy='560' r='5' fill='#e65100'/>
<rect x='1276' y='624' width='166' height='22' rx='4' fill='#fde0cc' stroke='#e65100' stroke-width='1.5'/>
<text x='1283' y='640' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#e65100'>E</text>
<text x='1298' y='640' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>right_back_middle</text>
<line x1='1270' y1='560' x2='1276' y2='624' stroke='#e65100' stroke-width='1.2' stroke-dasharray='3,3'/>
<circle cx='1270' cy='560' r='5' fill='#e65100'/>
<rect x='1320' y='600' width='174' height='22' rx='4' fill='#fde0cc' stroke='#e65100' stroke-width='1.5'/>
<text x='1327' y='616' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#e65100'>E</text>
<text x='1343' y='616' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>right_front_middle</text>
<line x1='1500' y1='560' x2='1494' y2='600' stroke='#e65100' stroke-width='1.2' stroke-dasharray='3,3'/>
<circle cx='1500' cy='560' r='5' fill='#e65100'/>
<line x1='200' y1='902' x2='430' y2='811' stroke='#1f5fbf' stroke-width='3' stroke-linecap='round'/>
<line x1='430' y1='811' x2='1270' y2='811' stroke='#1f5fbf' stroke-width='3' stroke-linecap='round'/>
<line x1='1270' y1='811' x2='1500' y2='902' stroke='#1f5fbf' stroke-width='3' stroke-linecap='round'/>
<line x1='200' y1='893' x2='200' y2='911' stroke='#1f5fbf' stroke-width='3' stroke-linecap='round'/>
<line x1='430' y1='802' x2='430' y2='820' stroke='#1f5fbf' stroke-width='3' stroke-linecap='round'/>
<line x1='1270' y1='802' x2='1270' y2='820' stroke='#1f5fbf' stroke-width='3' stroke-linecap='round'/>
<line x1='1500' y1='893' x2='1500' y2='911' stroke='#1f5fbf' stroke-width='3' stroke-linecap='round'/>
<rect x='718' y='779' width='120' height='22' rx='4' fill='#dbe8fb' stroke='#1f5fbf' stroke-width='1.5'/>
<text x='725' y='794' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#1f5fbf'>W</text>
<text x='740' y='794' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>back_bottom</text>
<rect x='862' y='779' width='128' height='22' rx='4' fill='#dbe8fb' stroke='#1f5fbf' stroke-width='1.5'/>
<text x='869' y='794' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#1f5fbf'>W</text>
<text x='884' y='794' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>front_bottom</text>
<line x1='844' y1='771' x2='856' y2='807' stroke='#1f5fbf' stroke-width='2' stroke-linecap='round'/>
<rect x='438' y='821' width='120' height='22' rx='4' fill='#e6d7f0' stroke='#6a1b9a' stroke-width='1.5'/>
<text x='445' y='836' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#6a1b9a'>B</text>
<text x='460' y='836' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>left_bottom</text>
<line x1='490' y1='861' x2='438' y2='843' stroke='#6a1b9a' stroke-width='1.2' stroke-dasharray='3,3'/>
<circle cx='490' cy='861' r='5' fill='#6a1b9a'/>
<rect x='782' y='821' width='136' height='22' rx='4' fill='#e6d7f0' stroke='#6a1b9a' stroke-width='1.5'/>
<text x='789' y='836' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#6a1b9a'>B</text>
<text x='804' y='836' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>center_bottom</text>
<line x1='850' y1='861' x2='782' y2='843' stroke='#6a1b9a' stroke-width='1.2' stroke-dasharray='3,3'/>
<circle cx='850' cy='861' r='5' fill='#6a1b9a'/>
<rect x='1134' y='821' width='128' height='22' rx='4' fill='#e6d7f0' stroke='#6a1b9a' stroke-width='1.5'/>
<text x='1141' y='836' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#6a1b9a'>B</text>
<text x='1156' y='836' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>right_bottom</text>
<line x1='1210' y1='861' x2='1262' y2='843' stroke='#6a1b9a' stroke-width='1.2' stroke-dasharray='3,3'/>
<circle cx='1210' cy='861' r='5' fill='#6a1b9a'/>
<rect x='438' y='779' width='120' height='22' rx='4' fill='#f8d7e3' stroke='#c2185b' stroke-width='1.5'/>
<text x='445' y='794' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#c2185b'>A</text>
<text x='460' y='794' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>left_bottom</text>
<rect x='1134' y='779' width='128' height='22' rx='4' fill='#f8d7e3' stroke='#c2185b' stroke-width='1.5'/>
<text x='1141' y='794' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#c2185b'>A</text>
<text x='1156' y='794' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>right_bottom</text>
<rect x='255' y='812' width='120' height='22' rx='4' fill='#d8ead9' stroke='#2e7d32' stroke-width='1.5'/>
<text x='262' y='828' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#2e7d32'>D</text>
<text x='277' y='828' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>left_bottom</text>
<rect x='1321' y='812' width='128' height='22' rx='4' fill='#d8ead9' stroke='#2e7d32' stroke-width='1.5'/>
<text x='1328' y='828' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#2e7d32'>D</text>
<text x='1343' y='828' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>right_bottom</text>
<rect x='206' y='840' width='166' height='22' rx='4' fill='#fde0cc' stroke='#e65100' stroke-width='1.5'/>
<text x='213' y='856' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#e65100'>E</text>
<text x='228' y='856' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>left_front_bottom</text>
<line x1='200' y1='902' x2='206' y2='862' stroke='#e65100' stroke-width='1.2' stroke-dasharray='3,3'/>
<circle cx='200' cy='902' r='5' fill='#e65100'/>
<rect x='266' y='771' width='158' height='22' rx='4' fill='#fde0cc' stroke='#e65100' stroke-width='1.5'/>
<text x='273' y='786' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#e65100'>E</text>
<text x='288' y='786' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>left_back_bottom</text>
<line x1='430' y1='811' x2='424' y2='793' stroke='#e65100' stroke-width='1.2' stroke-dasharray='3,3'/>
<circle cx='430' cy='811' r='5' fill='#e65100'/>
<rect x='1276' y='771' width='166' height='22' rx='4' fill='#fde0cc' stroke='#e65100' stroke-width='1.5'/>
<text x='1283' y='786' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#e65100'>E</text>
<text x='1298' y='786' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>right_back_bottom</text>
<line x1='1270' y1='811' x2='1276' y2='793' stroke='#e65100' stroke-width='1.2' stroke-dasharray='3,3'/>
<circle cx='1270' cy='811' r='5' fill='#e65100'/>
<rect x='1320' y='840' width='174' height='22' rx='4' fill='#fde0cc' stroke='#e65100' stroke-width='1.5'/>
<text x='1327' y='856' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#e65100'>E</text>
<text x='1343' y='856' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>right_front_bottom</text>
<line x1='1500' y1='902' x2='1494' y2='862' stroke='#e65100' stroke-width='1.2' stroke-dasharray='3,3'/>
<circle cx='1500' cy='902' r='5' fill='#e65100'/>
<rect x='436' y='898' width='105' height='22' rx='4' fill='#cfe9e5' stroke='#00796b' stroke-width='1.5'/>
<text x='443' y='914' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#00796b'>H</text>
<text x='458' y='914' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>left_back</text>
<rect x='436' y='924' width='105' height='22' rx='4' fill='#dde4e7' stroke='#546e7a' stroke-width='1.5'/>
<text x='443' y='940' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#546e7a'>F</text>
<text x='458' y='940' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>left_back</text>
<rect x='790' y='898' width='120' height='22' rx='4' fill='#cfe9e5' stroke='#00796b' stroke-width='1.5'/>
<text x='797' y='914' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#00796b'>H</text>
<text x='812' y='914' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>center_back</text>
<rect x='790' y='924' width='120' height='22' rx='4' fill='#dde4e7' stroke='#546e7a' stroke-width='1.5'/>
<text x='797' y='940' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#546e7a'>F</text>
<text x='812' y='940' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>center_back</text>
<rect x='1151' y='898' width='113' height='22' rx='4' fill='#cfe9e5' stroke='#00796b' stroke-width='1.5'/>
<text x='1158' y='914' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#00796b'>H</text>
<text x='1173' y='914' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>right_back</text>
<rect x='1151' y='924' width='113' height='22' rx='4' fill='#dde4e7' stroke='#546e7a' stroke-width='1.5'/>
<text x='1158' y='940' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#546e7a'>F</text>
<text x='1173' y='940' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>right_back</text>
<rect x='206' y='1018' width='113' height='22' rx='4' fill='#cfe9e5' stroke='#00796b' stroke-width='1.5'/>
<text x='213' y='1034' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#00796b'>H</text>
<text x='228' y='1034' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>left_front</text>
<rect x='206' y='1044' width='113' height='22' rx='4' fill='#dde4e7' stroke='#546e7a' stroke-width='1.5'/>
<text x='213' y='1060' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#546e7a'>F</text>
<text x='228' y='1060' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>left_front</text>
<rect x='786' y='1018' width='128' height='22' rx='4' fill='#cfe9e5' stroke='#00796b' stroke-width='1.5'/>
<text x='793' y='1034' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#00796b'>H</text>
<text x='808' y='1034' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>center_front</text>
<rect x='786' y='1044' width='128' height='22' rx='4' fill='#dde4e7' stroke='#546e7a' stroke-width='1.5'/>
<text x='793' y='1060' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#546e7a'>F</text>
<text x='808' y='1060' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>center_front</text>
<rect x='1374' y='1018' width='120' height='22' rx='4' fill='#cfe9e5' stroke='#00796b' stroke-width='1.5'/>
<text x='1381' y='1034' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#00796b'>H</text>
<text x='1396' y='1034' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>right_front</text>
<rect x='1374' y='1044' width='120' height='22' rx='4' fill='#dde4e7' stroke='#546e7a' stroke-width='1.5'/>
<text x='1381' y='1060' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#546e7a'>F</text>
<text x='1396' y='1060' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>right_front</text>
<text x='60' y='1130' font-family='Helvetica, Arial, sans-serif' font-size='16' font-weight='bold' fill='#222'>Template fields: the letter is the section, then the key. Top / middle / bottom are the three levels.</text>
<rect x='60' y='1147' width='22' height='20' rx='3' fill='#dbe8fb' stroke='#1f5fbf' stroke-width='1.5'/>
<text x='71' y='1162' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#1f5fbf' text-anchor='middle'>W</text>
<text x='92' y='1162' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>[width]: back = along the back wall, front = across the front plane</text>
<rect x='60' y='1175' width='22' height='20' rx='3' fill='#d8ead9' stroke='#2e7d32' stroke-width='1.5'/>
<text x='71' y='1190' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#2e7d32' text-anchor='middle'>D</text>
<text x='92' y='1190' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>[depth]: along the left / right side wall</text>
<rect x='60' y='1203' width='22' height='20' rx='3' fill='#f8d7e3' stroke='#c2185b' stroke-width='1.5'/>
<text x='71' y='1218' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#c2185b' text-anchor='middle'>A</text>
<text x='92' y='1218' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>[angle]: between the side wall and the back wall; 90 = square, more than 90 = opens toward the front</text>
<rect x='60' y='1231' width='22' height='20' rx='3' fill='#fde0cc' stroke='#e65100' stroke-width='1.5'/>
<text x='71' y='1246' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#e65100' text-anchor='middle'>E</text>
<text x='92' y='1246' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>[edge_plumb]: the four vertical edges of the opening, out of plumb</text>
<rect x='880' y='1147' width='22' height='20' rx='3' fill='#e6d7f0' stroke='#6a1b9a' stroke-width='1.5'/>
<text x='891' y='1162' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#6a1b9a' text-anchor='middle'>B</text>
<text x='912' y='1162' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>[back_wall_plumb]: the back wall out of plumb at left / center / right</text>
<rect x='880' y='1175' width='22' height='20' rx='3' fill='#cfe9e5' stroke='#00796b' stroke-width='1.5'/>
<text x='891' y='1190' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#00796b' text-anchor='middle'>H</text>
<text x='912' y='1190' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>[height]: six points, left / center / right x front / back</text>
<rect x='880' y='1203' width='22' height='20' rx='3' fill='#dde4e7' stroke='#546e7a' stroke-width='1.5'/>
<text x='891' y='1218' font-family='Helvetica, Arial, sans-serif' font-size='14' font-weight='bold' fill='#546e7a' text-anchor='middle'>F</text>
<text x='912' y='1218' font-family='Helvetica, Arial, sans-serif' font-size='14' fill='#222'>[floor_slope]: floor slope at the same six points</text>
<text x='60' y='1284' font-family='Helvetica, Arial, sans-serif' font-size='13' fill='#555'>Black: the walls of the opening. Red dashed: the front plane. Blue: the measuring lines. In the template a field is written in its section, for example [depth] left_top.</text>
</svg>
````

### `README.md`

````markdown
# Built-in max size

The largest built-in that fits an opening, from a site measurement: width, depth and
height in mm, plus every intermediate number, so you can see what drove the result.
The unit is then scribed to fit on site from those three numbers.

Plain Python 3, no dependencies. Works on its own or as a Claude Code workflow: send a
measurement in any form (text, a table, a photo of a marked-up sketch), the agent fills
the template, flags what doesn't add up before you start designing, and runs the numbers.
The workflow is in `CLAUDE.md`. `AGENTS.md` is an exact copy for Codex and other agents
that read that file; so far the kit has been tested in Claude Code only.

## Use

```bash
python3 builtin_max_size.py measurements/example.ini          # the result and how it was reached
python3 builtin_max_size.py measurements/example.ini --json   # the same, machine-readable
python3 builtin_max_size.py --template > measurements/2026-10-17_smith-closet.ini
python3 test_builtin_max_size.py                               # 19 tests, all must pass
```

The example gives 1221 x 605 x 2607 mm.

## What to measure

51 numbers. Sizes in mm, angles in degrees, three levels by height: top, middle, bottom.
Where each one sits on the opening: `field-map.svg`. A blank field means "not measured";
it counts as 0 in sums and is skipped in min/max, exactly like an empty spreadsheet cell.

| Section | What it is | Fields |
|---|---|---|
| `[width]` | opening width along the back wall and across the front plane, three levels | 6 |
| `[depth]` | along the left and right side walls, three levels | 6 |
| `[angle]` | side wall to back wall, left and right, three levels; 90 = square, more than 90 = the side wall opens up toward the front | 6 |
| `[edge_plumb]` | the four vertical edges of the opening (left/right x front/back): how far each sits from a plumb line at each level | 12 |
| `[back_wall_plumb]` | the back wall at left, center, right: how far it sits from a plumb line at each level | 9 |
| `[height]` | six points: left, center, right x front, back | 6 |
| `[floor_slope]` | at the same six points: how much higher the floor is there than at its lowest point | 6 |

The exact keys are in `measurement-template.ini`.

## How the numbers combine

**Width** is the largest of five candidates:

- for each level where the back width was measured: `MAX(back, front, back + left offset + right offset)`,
  where an offset is `ROUND(depth x SIN(angle - 90))` on that side, only when the angle is over 90;
- the largest front width plus the deviations of both front edges at that level;
- the largest back width plus the larger of the two back-edge deviations at that level.

An edge deviation at a level is that reading minus the smallest reading of the same edge
over the three levels, never below zero. On a tie between levels the first from the top wins.

**Depth**: if the back wall is out of plumb by 10 mm or more at any of the nine points,
take the largest of the six depths, otherwise the smallest.

**Height**: the largest of (height + floor slope) over the six points.

Shown but not used: the back offset, how much wider the back is where a side wall closes
in (angle under 90).

### Conventions of the method

These are deliberate and the tests pin them; change them only if you change the method.

- An edge deviation is added to the width measured at the same level.
- The front candidate adds both edges; the back candidate adds the larger one.
- Fillers, scribe strips and clearances are not computed. The result is the envelope.

## Where it comes from

The formulas were ported one to one from the spreadsheet we use at work and checked
against it on 41 measurements, the example plus 40 random ones, including blanks and
edge cases: every intermediate value matched. The example and six of those cases ship in
`reference-cases.json`, and `test_builtin_max_size.py` replays them, so a fresh install
checks itself.

## License

MIT License

Copyright (c) 2026 Max Srebny

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.

Home page: https://github.com/maxsrebny57/built-in-max-size
````

### `CLAUDE.md`

````markdown
# Built-in max size: how to work in this folder

This folder turns a site measurement of an opening into the maximum built-in size,
width x depth x height, with `builtin_max_size.py`. Read `README.md` once. Where each
field sits on the opening: `field-map.svg`.

## When the user sends a measurement

It can be text, a table, a spreadsheet, or a photo of a sketch with numbers written on
it. Do this, in this order:

1. **Transcribe it into a copy of the template.** Copy `measurement-template.ini` to
   `measurements/<date>_<object>.ini` and fill it. Every number goes to its field.
   Anything you had to interpret (which of two numbers is the back width, which level a
   pair of readings belongs to, a sign on an angle) goes into a `#` comment next to the
   field. Leave a field blank if it wasn't measured. Never invent a value.
2. **Show your reading before computing.** A short list: how you read each ambiguous
   mark, which fields are blank, what you left out and why (objects in the opening,
   numbers that aren't part of the method).
3. **Check that the measurement adds up.** This is the step that saves the most time.
   Look for partial lengths that should sum to a full length but don't (a step plus a
   box plus the rest); left and right depths that differ more than the wall angles can
   explain; back and front widths that disagree with the angles; heights that jump
   between neighboring points; anything in or in front of the opening that the unit or
   its doors would hit (lights, outlets, pipes, intercoms, trim). Say what doesn't add
   up and which number is the likely culprit. Don't fix it yourself: the measurement
   goes back to whoever took it, and the user waits for a confirmed one before designing.
4. **Run the script** and show the result:
   `python3 builtin_max_size.py measurements/<file>.ini`. Say which readings drove the
   width, which depth rule applied (back wall out of plumb by 10 mm or more takes the
   largest depth, otherwise the smallest), and which point set the height.
5. **Keep the file.** The `.ini` with its comments is the record of what was measured
   and how it was read.

## Rules

- Units are mm and degrees. If the measurement came in inches, convert first.
- Don't change the formulas in `builtin_max_size.py`. They are the method; the tests pin them.
- A blank field means "not measured", not zero.
- The result is the envelope the unit must fit into; it gets scribed to fit on site.
  Fillers, scribe strips and clearances are the designer's call, not part of the calculation.
````

---

Kit assembled 2026-10-19. Home page: https://github.com/maxsrebny57/built-in-max-size. MIT License, Copyright (c) 2026 Max Srebny; the full text is in `README.md` above. The spreadsheet check lives with the author; this file is the whole distribution.
