# Custom Godot build (kiberptah)

Branch `custom` holds local changes on top of upstream `master`.

## Changes

- **Exact 3D gizmo handle picking** (`editor/scene/3d/node_3d_editor_viewport.cpp`, `_pick_gizmo_handle`):
  move/scale handles are picked by casting the cursor ray against the handle meshes with the
  transforms they were drawn with. The closest handle actually under the cursor wins. Only when
  nothing is directly hit, the handle whose on-screen outline is within
  `editors/3d/manipulator_gizmo_pick_tolerance` pixels (default 3, `0` = exact only) is used.
- **Dry signal kept inside reverb areas** (`scene/3d/audio_stream_player_3d.cpp`, `_update_panning`):
  inside an Area3D with a reverb bus, an AudioStreamPlayer3D still plays on its own bus (or the
  area's override bus), and the reverb send is added on top, as in Godot 3. Upstream sent it only
  to the reverb bus, scaled by the area's reverb amount, so the dry sound went silent and an
  amount of 0 muted it entirely (godotengine/godot#96480).
- **UID cache count stays right with several Godot processes** (`core/io/resource_uid.cpp`,
  `update_cache`): before appending to `.godot/uid_cache.bin`, it counts the entries actually in
  the file instead of trusting its own tally, which goes stale when another process (an open
  editor, a command-line `--import`) wrote to the file meanwhile. A stale tally wrote too low a
  count into the header, and the last entries (often `default_bus_layout.tres`) were ignored at
  runtime. A file ending in a partial entry is rewritten in full.
- **`--no-focus` command-line flag, main window opens behind** (`main/main.cpp`,
  `platform/windows/display_server_windows.cpp` `show_window`): `--no-focus` opens the main window
  with `WINDOW_FLAG_NO_FOCUS`, the same as the `display/window/size/no_focus` project setting, but
  only for that run. On Windows a no-focus *main* window is now also shown at the bottom of the
  stacking order (`SetWindowPos(HWND_BOTTOM, SWP_NOACTIVATE | SWP_SHOWWINDOW)`) instead of on top,
  so automated runs (screenshot or test drivers started by scripts or an AI agent) neither take
  focus nor cover whatever you're working in; it still renders normally, so screenshots work.
  Other no-focus windows (tooltips, popups) behave as upstream. This also applies when a project
  sets `no_focus` itself. Like the project setting, a no-focus window ignores keyboard input, so
  it's meant for unattended runs. Ignored with `--wid` (embedded). `--position` can't move the
  window off-screen (it's pulled back onto the primary screen), so pair it with a small
  `--resolution`. Runs that don't need rendering should use `--headless`, which opens no window.
  Example: `--path <project> --windowed --no-focus --resolution 320x180 --script driver.gd`.

## Remotes

- `origin`   → https://github.com/kiberptah/godot (this fork)
- `upstream` → https://github.com/godotengine/godot

## Updating from upstream

```bash
git fetch upstream
git switch custom
git rebase upstream/master
git push --force-with-lease origin custom
```

## Building (Windows, MSVC)

```bash
python -m SCons platform=windows target=editor d3d12=no -j30
```

Output: `bin/godot.windows.editor.x86_64.exe`. Drop `d3d12=no` after running
`python misc/scripts/install_d3d12_sdk_windows.py` if the Direct3D 12 driver is needed.

## Export templates (Windows)

```bash
python -m SCons platform=windows target=template_release d3d12=no -j30
python -m SCons platform=windows target=template_debug d3d12=no -j30
```

Then install them where the editor looks for them (bash):

```bash
D="$APPDATA/Godot/export_templates/4.8.dev"; mkdir -p "$D"; printf '4.8.dev' > "$D/version.txt"
for t in release debug; do
	cp bin/godot.windows.template_$t.x86_64.exe "$D/windows_${t}_x86_64.exe"
	cp bin/godot.windows.template_$t.x86_64.console.exe "$D/windows_${t}_x86_64_console.exe"
done
```

Rebuild and reinstall them after any change to runtime code, or exported games won't have it.
