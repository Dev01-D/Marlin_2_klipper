# Marlin → Klipper G-code Converter

A simple drag-and-drop GTK4 app for Linux that converts Marlin G-code into Klipper-compatible G-code.

## Features
- Drag & drop `.gcode` files
- Saves `<name>_klipper.gcode` next to the original
- Converts a broad set of Marlin commands (G29/M420, M600/M0/M1/M25, M108/M24, M204/M205, M290, etc.)
- Comments firmware-level settings with guidance for `printer.cfg`
- Ships as a `.deb` with app menu entry and icon

## Install from .deb
```bash
sudo apt install ./releases/marlin2klipper_pro_2.0.deb
```

## Build from source
```bash
sudo apt install python3 python3-gi gir1.2-gtk-4.0
python3 marlin2klipper.py
dpkg-deb --build debian/marlin2klipper
```

## License
MIT (see LICENSE)
