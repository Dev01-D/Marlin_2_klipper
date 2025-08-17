#!/usr/bin/env python3
# -*- coding: utf-8 -*-
# Marlin -> Klipper G-code converter (Drag & Drop GTK4 GUI) - Pro

import sys
import re
from pathlib import Path
import gi
gi.require_version("Gtk", "4.0")
gi.require_version("Gdk", "4.0")
from gi.repository import Gtk, Gdk, GLib

APP_TITLE = "Marlin \u2192 Klipper G-code Converter"

def parse_params(s: str, keys_pattern=r'([A-Za-z])\s*(-?\d+(?:\.\d+)?)'):
    return {k: v for k, v in re.findall(keys_pattern, s)}

# ---------- Conversion rules ----------
RULES = [
    # Bed mesh & leveling
    (re.compile(r'^\s*G29\b.*', re.IGNORECASE),
     lambda m: "BED_MESH_PROFILE LOAD=default ; was G29 (assumes saved mesh 'default')"),
    (re.compile(r'^\s*M420\s+S(?P<S>[01])\b.*', re.IGNORECASE),
     lambda m: ("BED_MESH_PROFILE LOAD=default ; was M420 S1" if m.group('S') == '1'
                else "; NOTE: M420 S0 (disable mesh) not needed in Klipper. Original: " + m.group(0).strip())),
    (re.compile(r'^\s*M420\b.*', re.IGNORECASE),
     lambda m: "; NOTE: M420 options not directly used by Klipper. Original: " + m.group(0).strip()),

    # Pauses & filament change
    (re.compile(r'^\s*M600\b.*', re.IGNORECASE),
     lambda m: "PAUSE ; filament change (was M600)"),
    (re.compile(r'^\s*M0\b.*', re.IGNORECASE),
     lambda m: "PAUSE ; was M0"),
    (re.compile(r'^\s*M1\b.*', re.IGNORECASE),
     lambda m: "PAUSE ; was M1"),
    (re.compile(r'^\s*M25\b.*', re.IGNORECASE),
     lambda m: "PAUSE ; was M25 (SD pause)"),
    (re.compile(r'^\s*M108\b.*', re.IGNORECASE),
     lambda m: "RESUME ; was M108 (stop waiting)"),
    (re.compile(r'^\s*M24\b.*', re.IGNORECASE),
     lambda m: "RESUME ; was M24 (SD resume)"),
    (re.compile(r'^\s*M226\b.*', re.IGNORECASE),
     lambda m: "PAUSE ; was M226 (wait for pin)"),

    # Linear/pressure advance
    (re.compile(r'^\s*M900\b.*', re.IGNORECASE),
     lambda m: "; NOTE: M900 (Linear Advance) not used by Klipper. Configure pressure_advance in printer.cfg. Original: " + m.group(0).strip()),

    # Acceleration / jerk
    (re.compile(r'^\s*M204\b(?P<params>.*)', re.IGNORECASE),
     lambda m: (lambda p: (
         ("SET_VELOCITY_LIMIT " +
          ("ACCEL=" + p.get('S') if 'S' in p else (("ACCEL=" + p.get('P')) if 'P' in p else "")) +
          ((" ACCEL_TO_DECEL=" + str(int(float(p.get('P','0'))/2))) if 'P' in p else "")
         ).strip()
     ) if p else m.group(0)) (parse_params(m.group('params')))),
    (re.compile(r'^\s*M205\b(?P<params>.*)', re.IGNORECASE),
     lambda m: (lambda p: (
         ("SET_VELOCITY_LIMIT " +
          ("SQUARE_CORNER_VELOCITY=" + (
              str(max(float(p.get('X','0') or 0.0), float(p.get('Y','0') or 0.0)))
              if ('X' in p or 'Y' in p) else p.get('J','')
          ))
          if (('X' in p) or ('Y' in p) or ('J' in p)) else ""
         ).strip() or ("; NOTE: M205 parameters not directly mapped. Original: " + m.group(0).strip())
     )) (parse_params(m.group('params')))),

    # Firmware-level settings -> comments
    (re.compile(r'^\s*M201\b.*', re.IGNORECASE),
     lambda m: "; NOTE: M201 (max accel) is set in Klipper config. " + m.group(0).strip()),
    (re.compile(r'^\s*M203\b.*', re.IGNORECASE),
     lambda m: "; NOTE: M203 (max feedrate) is set in Klipper config. " + m.group(0).strip()),
    (re.compile(r'^\s*M92\b.*', re.IGNORECASE),
     lambda m: "; NOTE: M92 (steps/mm) is set in Klipper config. " + m.group(0).strip()),
    (re.compile(r'^\s*M301\b.*', re.IGNORECASE),
     lambda m: "; NOTE: M301 (PID) is set in Klipper config. " + m.group(0).strip()),
    (re.compile(r'^\s*M302\b.*', re.IGNORECASE),
     lambda m: "; NOTE: M302 (cold extrusion) is set in Klipper config. " + m.group(0).strip()),
    (re.compile(r'^\s*M851\b.*', re.IGNORECASE),
     lambda m: "; NOTE: M851 (Z probe offset) handled via SET_GCODE_OFFSET/config. " + m.group(0).strip()),

    # EEPROM
    (re.compile(r'^\s*M500\b.*', re.IGNORECASE), lambda m: "; NOTE: M500 ignored in Klipper"),
    (re.compile(r'^\s*M501\b.*', re.IGNORECASE), lambda m: "; NOTE: M501 ignored in Klipper"),
    (re.compile(r'^\s*M502\b.*', re.IGNORECASE), lambda m: "; NOTE: M502 ignored in Klipper"),
    (re.compile(r'^\s*M503\b.*', re.IGNORECASE), lambda m: "; NOTE: M503 ignored in Klipper"),

    # Baby stepping / Z offset adjust
    (re.compile(r'^\s*M290\b(?P<params>.*)', re.IGNORECASE),
     lambda m: (lambda p: (
         ("SET_GCODE_OFFSET Z_ADJUST=" + p.get('Z') + (" MOVE=1" if p.get('M','0') == '1' else ""))
         if 'Z' in p else "; NOTE: M290 with no Z param. " + m.group(0).strip()
     ))(parse_params(m.group('params')))),

    # SD/file management
    (re.compile(r'^\s*M20\b.*', re.IGNORECASE), lambda m: "; NOTE: M20 (SD list) not used. " + m.group(0).strip()),
    (re.compile(r'^\s*M21\b.*', re.IGNORECASE), lambda m: "; NOTE: M21 (init SD) not used. " + m.group(0).strip()),
    (re.compile(r'^\s*M22\b.*', re.IGNORECASE), lambda m: "; NOTE: M22 (release SD) not used. " + m.group(0).strip()),
    (re.compile(r'^\s*M23\b.*', re.IGNORECASE), lambda m: "; NOTE: M23 (select SD file) not used. " + m.group(0).strip()),
    (re.compile(r'^\s*M28\b.*', re.IGNORECASE), lambda m: "; NOTE: M28 (begin write SD) not used. " + m.group(0).strip()),
    (re.compile(r'^\s*M29\b.*', re.IGNORECASE), lambda m: "; NOTE: M29 (end write SD) not used. " + m.group(0).strip()),

    # Filament load/unload info
    (re.compile(r'^\s*M701\b.*', re.IGNORECASE),
     lambda m: "; NOTE: M701 (load filament) - make a LOAD_FILAMENT macro in Klipper. " + m.group(0).strip()),
    (re.compile(r'^\s*M702\b.*', re.IGNORECASE),
     lambda m: "; NOTE: M702 (unload filament) - make an UNLOAD_FILAMENT macro in Klipper. " + m.group(0).strip()),

    # Disable motors
    (re.compile(r'^\s*M84\b.*', re.IGNORECASE),
     lambda m: "M18 ; was M84 (Klipper alias)"),

    # Print timer meta
    (re.compile(r'^\s*M75\b.*', re.IGNORECASE), lambda m: "; NOTE: M75 (Print timer start) handled by Klipper/macros."),
    (re.compile(r'^\s*M76\b.*', re.IGNORECASE), lambda m: "; NOTE: M76 (Print timer pause) handled by Klipper/macros."),
    (re.compile(r'^\s*M77\b.*', re.IGNORECASE), lambda m: "; NOTE: M77 (Print timer stop) handled by Klipper/macros."),
]

HEADER = (
    "; ------------------------------------------------------------\n"
    "; Converted from Marlin to Klipper by marlin2klipper (Pro)\n"
    "; Key mappings:\n"
    ";  • G29 / M420 S1  -> BED_MESH_PROFILE LOAD=default\n"
    ";  • M600/M0/M1/M25 -> PAUSE    | M108/M24 -> RESUME\n"
    ";  • M900           -> comment (use pressure_advance)\n"
    ";  • M204/M205      -> SET_VELOCITY_LIMIT approximations\n"
    ";  • M290 Z         -> SET_GCODE_OFFSET Z_ADJUST=<Z>\n"
    ";  • M201/M203/M92/M301/M302/M851 -> commented (use printer.cfg)\n"
    "; Review your start/end gcode; consider START_PRINT/END_PRINT macros.\n"
    "; ------------------------------------------------------------\n"
)

def convert_line(line: str) -> str:
    if re.match(r'^\s*;', line):
        return line
    for pattern, repl in RULES:
        m = pattern.match(line)
        if m:
            out = repl(m)
            if not out.endswith("\n"):
                out += "\n"
            return out
    return line

def convert_file(in_path: Path, out_path: Path) -> None:
    with in_path.open("r", encoding="utf-8", errors="ignore") as f_in, \
         out_path.open("w", encoding="utf-8", errors="ignore") as f_out:
        f_out.write(HEADER)
        for line in f_in:
            f_out.write(convert_line(line))

class DropWindow(Gtk.ApplicationWindow):
    def __init__(self, app):
        super().__init__(application=app, title=APP_TITLE)
        self.set_default_size(480, 220)
        box = Gtk.Box(orientation=Gtk.Orientation.VERTICAL, spacing=10, margin_top=20, margin_bottom=20, margin_start=20, margin_end=20)
        self.set_child(box)

        title = Gtk.Label(label="Drop your Marlin .gcode here")
        title.get_style_context().add_class("title-2")
        box.append(title)

        self.msg = Gtk.Label(label="Output will be saved next to the original as *_klipper.gcode*")
        self.msg.set_wrap(True)
        box.append(self.msg)

        self.drop_target = Gtk.DropTarget.new(Gdk.FileList, Gdk.DragAction.COPY)
        self.drop_target.connect("drop", self.on_drop)
        self.add_controller(self.drop_target)

    def on_drop(self, _target, value, _x, _y):
        try:
            files = [Path(f.get_path()) for f in value.get_files() if f.get_path()]
            converted = []
            for in_path in files:
                if in_path.suffix.lower() == ".gcode":
                    out_path = in_path.with_name(in_path.stem + "_klipper.gcode")
                    convert_file(in_path, out_path)
                    converted.append(str(out_path))
            if converted:
                self.msg.set_text("Converted:\n" + "\n".join(converted))
            else:
                self.msg.set_text("No .gcode files detected in drop.")
        except Exception as e:
            self.msg.set_text(f"Error: {e}")
        return True

class App(Gtk.Application):
    def __init__(self):
        super().__init__(application_id="dev.geoff.marlin2klipper")
    def do_activate(self, *args, **kwargs):
        win = DropWindow(self)
        win.present()

def main():
    if len(sys.argv) == 2:
        in_path = Path(sys.argv[1]).expanduser().resolve()
        if not in_path.exists() or in_path.suffix.lower() != ".gcode":
            print("Please provide an existing .gcode file.", file=sys.stderr)
            sys.exit(1)
        out_path = in_path.with_name(in_path.stem + "_klipper.gcode")
        convert_file(in_path, out_path)
        print(f"Saved: {out_path}")
        return
    app = App()
    app.run(None)

if __name__ == "__main__":
    main()
