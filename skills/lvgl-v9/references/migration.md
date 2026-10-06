# Migration reference: v8 → v9, v9.5 → v9.6, preparing for v10

Contents: 1 Strategy · 2 v8 → v9 · 3 v9.5 → v9.6 deprecations · 4 Config changes in 9.6 ·
5 Include and private API changes · 6 Checklist

---

## 1. Strategy

- v9.6.0 (16 Sept 2026) is the **last v9 release**. Everything marked `LV_DEPRECATED` is removed in
  v10.0; deprecated names still compile and print a `#warning`. Upgrade to 9.6, fix every warning, then
  v10 is a much smaller jump.
- Authoritative sources, in order: the **migration guide pages** in the LVGL docs
  (`changelog/migration-v9-6`, `changelog/migration-v10`), the **API-map headers** shipped in the repo
  (`lv_api_map_v8.h`, `lv_api_map_v9_x.h`: grep these for old → new names), then the changelog.
- Upgrade one major step at a time (v8 → v9.x → v9.6) and keep the build green between steps.
- Build with warnings enabled and read them: many 9.6 deprecations only show as `#warning`s.

## 2. v8 → v9 (the big jump)

Things that compile but behave differently (check these first):

- `lv_display_set_buffers(disp, buf1, buf2, size, mode)`: buffer size is **in bytes**, not pixels
  (v8's `lv_disp_draw_buf_init` took pixels).
- `lv_color_t` is **always RGB888** internally regardless of the configured display depth.
- `lv_conf.h` changed heavily: regenerate it from the current `lv_conf_template.h`; do not reuse a v8 file.
- Old v8 byte-swap option `LV_COLOR_16_SWAP` is gone/deprecated; use `LV_COLOR_FORMAT_RGB565_SWAPPED`.
- The online image converter lagged v9 at first; use `scripts/LVGLImage.py` and verify formats.
- Image descriptors changed: `lv_image_dsc_t` with `.header.magic/cf/w/h/stride`, `.data_size`, `.data`.

Rename table (verify exact names against `lv_api_map_v8.h` if in doubt):

| v8 | v9 |
|---|---|
| `lv_disp_t`, `lv_disp_drv_t`, `lv_disp_draw_buf_t` | `lv_display_t` (+ `lv_display_create`, `lv_display_set_buffers`, `lv_display_set_flush_cb`) |
| `lv_disp_flush_ready()` | `lv_display_flush_ready()` |
| flush cb `(lv_disp_drv_t*, const lv_area_t*, lv_color_t*)` | `(lv_display_t*, const lv_area_t*, uint8_t * px_map)` |
| `lv_indev_drv_t` + `lv_indev_drv_register` | `lv_indev_create()`, `lv_indev_set_type()`, `lv_indev_set_read_cb()`; read cb `(lv_indev_t*, lv_indev_data_t*)` |
| `lv_scr_act()` / `lv_scr_load()` / `lv_scr_load_anim()` | `lv_screen_active()` / `lv_screen_load()` / `lv_screen_load_anim()` |
| `lv_disp_get_default()` and `lv_disp_*` getters | `lv_display_get_default()` and `lv_display_*` |
| `lv_obj_del()` / `lv_obj_del_async()` | `lv_obj_delete()` / `lv_obj_delete_async()` |
| `lv_obj_clear_flag()` | `lv_obj_remove_flag()` (then deprecated again in 9.6, see below) |
| `lv_btn_*`, `lv_img_*`, `lv_imgbtn_*`, `lv_btnmatrix_*` | `lv_button_*`, `lv_image_*`, `lv_imagebutton_*`, `lv_buttonmatrix_*` |
| `lv_meter` | `lv_scale` (+ line/arc/image needles) |
| `lv_img_dsc_t`, `LV_IMG_CF_*` | `lv_image_dsc_t`, `LV_COLOR_FORMAT_*` |
| `lv_mem_alloc()` / `lv_mem_free()` | `lv_malloc()` / `lv_free()` |
| `lv_anim_del()` / `lv_timer_del()` | `lv_anim_delete()` / `lv_timer_delete()` |
| style `transform_zoom`, `LV_ZOOM_NONE`, `lv_img_set_zoom` | `transform_scale`, `LV_SCALE_NONE`, `lv_image_set_scale` (256 = 1×) |
| `lv_style_set_bg_img_*`, `img_*` style props | `..._bg_image_*`, `..._image_*` |
| `lv_tick_inc()` only | still valid; plus `lv_tick_set_cb()` |
| built-in/hand-written display & indev drivers | many ready drivers now ship in LVGL (SDL, Linux fbdev, ST7789/ILI9341, evdev...) |
| `lv_event_get_target()` | still exists (returns `void *`); prefer `lv_event_get_target_obj()` for widgets |
| `lv_anim_set_time()` | `lv_anim_set_duration()` in later 9.x (check your minor version) |

New in v9 worth adopting: runtime color-format changes, render modes `PARTIAL/DIRECT/FULL`, multi-display and
display/indev events, built-in OS support (`lv_lock`), draw units/GPU architecture, Observer/Subject
data binding, `lv_scale`, vector graphics/canvas, widget names.

## 3. v9.5 → v9.6 deprecations (all removed in v10)

| Area | Deprecated | Use instead |
|---|---|---|
| Object flags | `lv_obj_add_flag`, `lv_obj_remove_flag`, `lv_obj_set_flag`, `lv_obj_has_flag`, `lv_obj_has_flag_any` | `lv_obj_set_<flag>(obj, en)` / `lv_obj_is_<flag>(obj)`, e.g. `lv_obj_set_hidden`, `lv_obj_set_clickable`, `lv_obj_set_scrollable`; user flags: `lv_obj_set_user_flag(obj, 0..3, en)`; script: `scripts/migration/migrate_obj_flags.py` |
| States | generic state calls for common states | `lv_obj_set_checked/is_pressed/set_disabled...`, `lv_obj_set_state(obj, st, bool)` |
| Find by id | `lv_obj_find_by_id()` | `lv_obj_find_by_name()` |
| Style enable | `lv_obj_style_set_disabled/get_disabled` (inverted logic) | `lv_obj_set_style_enabled` / `lv_obj_get_style_enabled` (note: `true` = enabled) |
| Assertions | `LV_ASSERT_OBJ(obj, cls)` | `LV_CHECK_OBJ(obj, cls, return)` (logs, recovers); asserts are off by default, `LV_USE_ASSERT` to enable |
| Subjects | `lv_subject_init_int/float/string/pointer/color/group()`, `lv_subject_deinit()`, `lv_subject_copy_string()` | `lv_subject_create(TYPE)` (pointer) + `lv_subject_set_*`, `lv_subject_delete()`, `lv_subject_set_string()` |
| Bindings | `lv_obj_bind_flag_if_{eq,not_eq,gt,ge,lt,le}`, `lv_obj_bind_state_if_*` | `lv_obj_bind_bool(obj, subj, lv_obj_set_hidden)` or a custom observer via `lv_subject_add_observer_obj()` |
| Observer | `_remove` naming | renamed to `_delete` (e.g. `lv_observer_delete`) |
| Display API | any `lv_display_*` / `lv_sysmon_*` call with `NULL` display | pass `lv_display_get_default()` explicitly |
| Color config | `LV_COLOR_DEPTH` (now derived, read-only), `LV_COLOR_FORMAT_NATIVE[_WITH_ALPHA]`, `LV_COLOR_16_SWAP` | `LV_COLOR_FORMAT_DEFAULT` (`..._RGB565`, `..._RGB565_SWAPPED`, `..._XRGB8888`, `..._ARGB8888`, ...) or `lv_display_set_color_format()` |
| Memory config | `LV_MEM_SIZE_KILOBYTES`, `LV_MEM_POOL_EXPAND_SIZE[_KILOBYTES]` | `LV_MEM_SIZE` in **bytes** (e.g. `64 * 1024`) |
| Threading config | `LV_DRAW_THREAD_STACKSIZE` | `LV_DRAW_THREAD_STACK_SIZE` |
| Draw utils | `lv_draw_sw_i1_to_argb8888`, `_rgb565_swap`, `_rotate`, `_i1_invert`, `_i1_convert_to_vtiled` | same names without `_sw_`: `lv_draw_rgb565_swap`, `lv_draw_rotate`, ... |
| Draw buffers | `lv_image_buf_set_palette`, `lv_image_buf_free`, `lv_snapshot_free`, `lv_snapshot_take_to_buf` | `lv_draw_buf_set_palette`, `lv_draw_buf_destroy`, `lv_snapshot_take_to_draw_buf` |
| Layouts | `lv_layout_register()` | `lv_layout_create(callbacks, user_data)` |
| Widgets | `lv_list`, `lv_menu`, `lv_win`, `lv_file_explorer` | build from `lv_obj` + flex (see `ui-patterns.md`) |
| Span/Textarea | `lv_spangroup_set_align/mode`, `lv_textarea_set_align` | `text_align` style; `lv_obj_set_width(LV_SIZE_CONTENT / fixed)` |
| Scale | `lv_scale_section_set_style()` | `lv_scale_set_section_style_main/indicator/items(scale, section, style)` |
| Libraries | rlottie player, `lv_fragment` (deprecated in 9.5) | `lv_lottie` widget; plain screens/components |
| Config macros | `LV_ASSERT_HANDLER_INCLUDE`, `LV_CALENDAR_DEFAULT_DAY/MONTH_NAMES`, `LV_LIBINPUT_XKB_KEY_MAP`, `LV_USE_LZ4_EXTERNAL`, `LV_USE_THORVG_EXTERNAL`, `LV_VG_LITE_HAL_GPU_*`, `LV_SDL_SINGLE/DOUBLE_BUFFER`, `LV_X11_RENDER_MODE_*`, `LV_CONF_MINIMAL` | see section 4 |

Behavior changes to know about in 9.6 (not just renames):
- `LV_USE_CHECK_ARG` (default on): invalid public-API arguments log a warning and return.
- Assertions off by default; object class/validity checks default off.
- Object style cache on by default.
- `lv_subject_create()` has no initial-value parameter: set min/max **before** the first `set`, then
  set the value; the "previous value" starts at the neutral value (0/NULL/empty).
- `lv_qrcode_set_size/set_quiet_zone` now re-encode when called after data (costs `data_len` bytes per object).
- v9.5 removed the XML engine from the open-source repo (LVGL Pro continues it); v9.5 also rewrote the
  Wayland driver (no client-side decorations) and removed some v8 display-rotation names that 9.6 restored via the API map.

## 4. Config changes in 9.6

- Kconfig is the single source of truth; `lv_conf_template.h`, internal defaults, and the `CONFIG_*`
  bridge are generated from it. **Hand-written `lv_conf.h` remains fully supported** and the default
  for most projects. To migrate an old `lv_conf.h`: copy the new `lv_conf_template.h` over it and
  re-apply your values (or re-run `scripts/generate_lv_conf.py` if you use `lv_conf.defaults`).
- `LV_COLOR_DEPTH` → `LV_COLOR_FORMAT_DEFAULT`:

  | Before | After |
  |---|---|
  | `LV_COLOR_DEPTH 1` | `LV_COLOR_FORMAT_DEFAULT LV_COLOR_FORMAT_I1` |
  | `LV_COLOR_DEPTH 8` | `... LV_COLOR_FORMAT_L8` |
  | `LV_COLOR_DEPTH 16` | `... LV_COLOR_FORMAT_RGB565` (or `RGB565_SWAPPED`) |
  | `LV_COLOR_DEPTH 24` | `... LV_COLOR_FORMAT_RGB888` |
  | `LV_COLOR_DEPTH 32` | `... LV_COLOR_FORMAT_XRGB8888` (or `ARGB8888`) |

  `LV_COLOR_DEPTH` stays readable (`#if LV_COLOR_DEPTH == 32` works) but is derived and lossy.
  `LV_COLOR_FORMAT_DEFAULT` is only each display's *starting* format.
- `LV_COLOR_16_SWAP` users: set `LV_COLOR_FORMAT_DEFAULT = LV_COLOR_FORMAT_RGB565_SWAPPED`, or
  `lv_display_set_color_format(display, LV_COLOR_FORMAT_RGB565_SWAPPED)` at runtime; last resort: swap in
  `flush_cb` with `lv_draw_rgb565_swap()` (in DIRECT mode swap the right rows of the full frame buffer
  using the buffer stride, not just `px_map`).
- Drivers no longer auto-infer their backend: set `LV_SDL_BACKEND` / `LV_LINUX_DRM_BACKEND` explicitly and disable
  the corresponding `_AUTO_BACKEND` flag; Wayland backends (`LV_WAYLAND_USE_SHM/EGL/G2D/DMABUF`) are
  independent options.
- Kconfig can't hold function-like macros: use `LV_<MODULE>_USE_CUSTOM_INCLUDE` + `_CUSTOM_INCLUDE` header
  (FONT, ASSERT, ATTRIBUTE, SYSMON, NEMA, global).
- `LV_USE_PXP` / `LV_USE_G2D` are no-ops: use `LV_USE_DRAW_PXP` / `LV_USE_DRAW_G2D`.
- LZ4/ThorVG: enable `LV_USE_LZ4` / `LV_USE_THORVG` and choose bundled vs external with `_INTERNAL`.
- `LV_CONF_MINIMAL` removed: start from `configs/defconfigs/empty.defconfig`.

## 5. Include and private API changes

- Public headers moved to `include/lvgl/`. Canonical include: add `lvgl/include` to include paths and
  `#include <lvgl/lvgl.h>`. The old `lvgl/lvgl.h` root header still works.
- Headers left under `src/` are **private**; including moved headers directly from `src/` still works in 9.6
  but warns, and is removed in v10.
- Internal types (SVG structures, event list, `lv_array`, `lv_tree`) are private now: if you really need
  them, include `lvgl_private.h` or set `LV_USE_PRIVATE_API`. Normal public-API users are unaffected.

## 6. Checklist

1. Pin LVGL to 9.6.x; copy the new `lv_conf_template.h`; re-apply options (or Kconfig defconfig).
2. Replace `LV_COLOR_DEPTH` / `LV_COLOR_16_SWAP` / `LV_MEM_*_KILOBYTES` / renamed options (section 4).
3. Rebuild with warnings visible; fix each deprecation (section 3).
4. Run `migrate_obj_flags.py` on your sources; review the diff; hand-fix expression-based flag uses.
5. Convert `lv_subject_init_*` to `lv_subject_create`; fix `&subject` → `subject`; add min/max before first set.
6. Replace `lv_list/menu/win/file_explorer` with flex-based components.
7. Pass `lv_display_get_default()` instead of `NULL` to display/sysmon calls.
8. Move includes to `<lvgl/lvgl.h>`; remove `src/...` includes.
9. Run the app with `LV_USE_CHECK_ARG`, class/validity checks and logging on; fix everything it prints.
10. Re-measure with sysmon (the 9.6 renderer is faster; your numbers should improve, not regress).
11. Read `changelog/migration-v10` in the docs to see what else v10 changes.