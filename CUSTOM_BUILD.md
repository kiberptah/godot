# Custom Godot build (kiberptah)

Branch `custom` holds local changes on top of upstream `master`.

## Changes

- **Exact 3D gizmo handle picking** (`editor/scene/3d/node_3d_editor_viewport.cpp`, `_pick_gizmo_handle`):
  move/scale handles are picked by casting the cursor ray against the handle meshes with the
  transforms they were drawn with. The closest handle actually under the cursor wins. Only when
  nothing is directly hit, the handle whose on-screen outline is within
  `editors/3d/manipulator_gizmo_pick_tolerance` pixels (default 3, `0` = exact only) is used.
- **UID cache count stays right with several Godot processes** (`core/io/resource_uid.cpp`,
  `update_cache`): before appending to `.godot/uid_cache.bin`, it counts the entries actually in
  the file instead of trusting its own tally, which goes stale when another process (an open
  editor, a command-line `--import`) wrote to the file meanwhile. A stale tally wrote too low a
  count into the header, and the last entries (often `default_bus_layout.tres`) were ignored at
  runtime. A file ending in a partial entry is rewritten in full.

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
