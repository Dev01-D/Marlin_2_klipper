# Marlin → Klipper G-code Converter

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE)

A simple **drag-and-drop GTK4 desktop app** for Linux that converts Marlin G-code files into Klipper-compatible G-code.  
Created to help make migration from Marlin-based printers to Klipper smooth, without having to manually edit every G-code file.

---

## ✨ Features

- 🚀 **Drag & Drop Interface**  
  Drop `.gcode` files into the window – no command line required.

- 📝 **Automatic Conversion**  
  Produces `<filename>_klipper.gcode` next to your original file.

- 🔧 **Extensive Conversion Rules**  
  Handles or comments out most Marlin-specific commands safely:
  - **Bed Mesh / Leveling**
    - `G29`, `M420 S1` → `BED_MESH_PROFILE LOAD=default`
  - **Pause / Resume**
    - `M600`, `M0`, `M1`, `M25` → `PAUSE`
    - `M108`, `M24` → `RESUME`
    - `M226` → `PAUSE`
  - **Linear Advance**
    - `M900` → commented with note (use `pressure_advance` in Klipper)
  - **Acceleration / Jerk**
    - `M204` / `M205` → `SET_VELOCITY_LIMIT` approximations
  - **Baby Stepping**
    - `M290 Z…` → `SET_GCODE_OFFSET Z_ADJUST=…`
  - **Firmware-level Settings**
    - `M201`, `M203`, `M92`, `M301`, `M302`, `M851` → commented with hints for `printer.cfg`
    - EEPROM (`M500–M503`) → commented
    - SD commands (`M20–M29`) → commented
  - **Other**
    - `M84` → `M18`
    - `M701` / `M702` → note to create macros in Klipper
  - Leaves all standard motion, extrusion, heating, and fan commands untouched.

- 🎨 **Desktop Integration**  
  Ships with a `.deb` installer, GNOME/Ubuntu app menu entry, and a custom 2D plotter icon.

---

## 📦 Install (Recommended)

Download the latest `.deb` from [Releases](./releases) and install with:

```bash
sudo apt install ./marlin2klipper_pro_2.0.deb
```

Launch from your **Applications menu**:  
**Marlin → Klipper G-code Converter**

Drag `.gcode` files into the window → `_klipper.gcode` will be created alongside the original.

---

## 🔧 Build from Source

### Dependencies

```bash
sudo apt install python3 python3-gi gir1.2-gtk-4.0
```

### Run directly

```bash
python3 marlin2klipper.py
```

### Build a `.deb` manually

```bash
dpkg-deb --build debian/marlin2klipper
```



---

## 📝 License

Distributed under the MIT License. See [LICENSE](./LICENSE) for details.

---

## 🙌 Credits

Created by **Sabitech** ([@Dev01-D](https://github.com/Dev01-D))  
With thanks to the **Klipper** and **Marlin** communities for their great firmware.

---
