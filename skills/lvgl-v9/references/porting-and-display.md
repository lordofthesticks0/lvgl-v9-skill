# Porting and display reference (LVGL v9.6)

Contents: 1 Init order · 2 Tick · 3 Timer handler and sleeping · 4 Threads and RTOS ·
5 Display setup (buffers, render modes, flush) · 6 Color format and byte swapping ·
7 Input devices · 8 Configuration · 9 ESP32 / ESP-IDF · 10 Pipeline picture

---

## 1. Init order

1. Initialize hardware (clocks, SPI/RGB/MIPI peripheral, panel, touch).
2. `lv_init()`.
3. Tick: `lv_tick_set_cb()` (or `lv_tick_inc()` from a timer).
4. `lv_display_create()` → color format → buffers → flush callback.
5. `lv_indev_create()` for each input device.
6. Build UI (inside the LVGL task, or under `lv_lock()`).
7. Start the `lv_timer_handler()` loop/task.

Verify the panel works **without LVGL** first (fill it red, then draw a gradient). If the panel is
wrong, no amount of LVGL tuning helps.

## 2. Tick

LVGL needs a millisecond clock. Either:

- `lv_tick_set_cb(cb)` where `cb` returns ms since boot (preferred), or
- `lv_tick_inc(x)` from a periodic timer/ISR every `x` ms.

Platform one-liners: SDL `SDL_GetTicks`; STM32 `HAL_GetTick`; Arduino wrapper around `millis()`;
ESP32 wrapper returning `esp_timer_get_time() / 1000`; FreeRTOS `xTaskGetTickCount` **only if the
tick rate is 1000 Hz** (otherwise convert with `pdTICKS_TO_MS`).

`lv_tick_inc()` may be called from any thread/ISR if a 32-bit write is atomic on your platform.

## 3. Timer handler and sleeping

Everything LVGL does (rendering, input reads, animations, widget timers, your `lv_timer`s) runs as
a timer inside `lv_timer_handler()`. Call it periodically from `main()` or one OS task.

```c
for(;;) {
    uint32_t wait = lv_timer_handler();               /* ms until next timer is due */
    if(wait == LV_NO_TIMER_READY) wait = LV_DEF_REFR_PERIOD;  /* nothing scheduled: check again soon */
    lv_sleep_ms(wait);
}
```

- `LV_NO_TIMER_READY` (`UINT32_MAX`) is returned when there is nothing to redraw, no enabled indevs,
  no running animations, and no user timers. Do not sleep forever: either sleep briefly, wait on an
  event you post when LVGL has work, or sleep the CPU.
- With `LV_USE_OS` set, `lv_sleep_ms` is the OS sleep; otherwise it is a blocking delay.
- Low-power pattern: skip `lv_timer_handler()` and sleep the MCU when
  `lv_display_get_inactive_time(lv_display_get_default()) > N` and `lv_anim_count_running() == 0`.
- 9.6 adds a function to query the time of the next timer if you need finer scheduling.

## 4. Threads and RTOS

**LVGL is not thread-safe.** Do not call any LVGL function while another LVGL call (including
`lv_timer_handler`) is executing in another thread.

Set `LV_USE_OS` to your OS (`LV_OS_FREERTOS`, `LV_OS_PTHREAD`, `LV_OS_RTTHREAD`, ...) then:

```c
void lvgl_task(void * arg) {
    for(;;) {
        uint32_t wait = lv_timer_handler();   /* takes/releases the lock internally */
        if(wait == LV_NO_TIMER_READY || wait > 10) wait = 10;   /* other threads may make timers ready */
        vTaskDelay(pdMS_TO_TICKS(wait));
    }
}

void sensor_task(void * arg) {
    for(;;) {
        int t = read_temperature();
        lv_lock();
        lv_subject_set_int(temperature_subject, t);   /* any LVGL call goes inside the lock */
        lv_unlock();
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

- No lock needed inside event/timer/animation callbacks (already inside `lv_timer_handler`).
- Allowed from any thread without a lock: `lv_tick_inc()`, `lv_display_flush_ready()`.
- Do **not** call LVGL from an ISR. Post to a queue or set a flag; let the LVGL task apply it.
- Hold the lock for a *group* of related calls, not one lock per call, and never block on I/O while
  holding it (the UI freezes).
- If you run `lv_timer_handler()` in an RTOS task, increase its stack size when you see random
  crashes (this is in LVGL's own FAQ).
- Alternative design that avoids shared locking: producer tasks write into a queue/atomic and a single
  LVGL-thread `lv_timer` drains it into subjects.
- Multiple independent LVGL instances are possible via `LV_GLOBAL_CUSTOM` + thread-local storage
  (rare; read the Integration overview page first).

## 5. Display setup

```c
lv_display_t * disp = lv_display_create(hor_res, ver_res);
lv_display_set_color_format(disp, LV_COLOR_FORMAT_RGB565);
lv_display_set_buffers(disp, buf1, buf2_or_NULL, size_in_BYTES, render_mode);
lv_display_set_flush_cb(disp, flush_cb);
```

The first display created becomes the default display. v9.6 deprecates passing `NULL` as the
display to `lv_display_*` functions: use `lv_display_get_default()` explicitly.

### Render modes

| Mode | Buffer | Behavior | Use when |
|---|---|---|---|
| `PARTIAL` | Smaller than screen (≥ 1/10, 1/5 recommended) | Renders only invalidated areas in chunks; `flush_cb` is called per chunk and must copy the chunk to the panel window | Default for MCUs; lowest RAM |
| `DIRECT` | Whole screen (1 or 2 buffers) | Renders into the right place of a full frame buffer, only invalidated areas; with 2 buffers LVGL syncs the other buffer after flush; `flush_cb` typically just switches the frame-buffer address | RGB/MIPI panels with frame buffers, enough RAM |
| `FULL` | Whole screen (1 or 2 buffers) | Redraws the entire screen every refresh | Simplest traditional double-buffering; wastes CPU on unchanged screens |

Buffers:
- **One buffer**: LVGL renders, calls flush, then *waits* for `lv_display_flush_ready()` before it
  renders again. Serial, simple.
- **Two buffers**: render into one while DMA sends the other. Needs DMA (or a peripheral doing the
  transfer) to be worth it.
- **Three buffers** (`lv_display_set_3rd_draw_buffer`): removes idle time waiting for DMA completion
  in direct/full modes.
- Custom stride for a controller: `lv_display_set_buffers_with_stride()`.
- Draw buffers should be aligned to `LV_DRAW_BUF_ALIGN` (default 4; some accelerators need more,
  e.g. 64 for ESP32-P4 PPA).

Sizing example (RGB565 = 2 B/px, 320×240): full frame = 153,600 B; 1/10 = 15,360 B; 1/5 = 30,720 B.

### The flush callback contract

```c
void flush_cb(lv_display_t * disp, const lv_area_t * area, uint8_t * px_map);
```

- `area` coordinates are **inclusive** (`x2`/`y2` are the last pixel). Width = `x2 - x1 + 1`.
- LVGL may call flush multiple times per frame; `lv_display_flush_is_last(disp)` tells you when it
  is the final chunk (useful to trigger a VSYNC swap or backlight enable on the first frame).
- You **must** eventually call `lv_display_flush_ready(disp)` once per call. This is the single most
  common porting bug. It may be called from the DMA-complete callback or another thread.
- Optionally set `lv_display_set_flush_wait_cb()` to block on a semaphore instead of LVGL spinning
  while it waits for the previous flush. The wait callback does not call `flush_ready` itself.
- Keep the flush callback fast and non-blocking; prefer async DMA.

Direct mode with two frame buffers (RGB/MIPI panels): in `flush_cb`, if `lv_display_flush_is_last()`,
tell the panel to display `px_map`; call `lv_display_flush_ready()` from the VSYNC-done interrupt so
LVGL does not block while waiting for the buffer swap.

### Monochrome (I1) panels

- Use `LV_COLOR_FORMAT_I1`; LVGL reserves 8 bytes at the start of the buffer for a palette, so make
  the buffer 8 bytes larger and **skip the palette** in `flush_cb` (`px_map += 8`).
- Redrawn areas are rounded to byte boundaries; for controllers needing N×8 rows/columns add a
  rounder callback on `LV_EVENT_INVALIDATE_AREA`.
- Round the horizontal resolution up to a multiple of 8.
- Use `lv_draw_i1_convert_to_vtiled()` (9.6 name; `lv_draw_sw_i1_convert_to_vtiled` in ≤ 9.5) for
  page-addressed controllers.

### Rotation and resolution

- `lv_display_set_rotation()` rotates in software (LVGL swaps resolutions internally). If the
  panel/driver can rotate in hardware, prefer that.
- Resolution changes at runtime: `lv_display_set_resolution()` sends `LV_EVENT_RESOLUTION_CHANGED`.
- Display events (`lv_display_add_event_cb`): `LV_EVENT_FLUSH_START/FINISH`, `RENDER_START/READY`,
  `REFR_START/READY`: useful for FPS counters and logic analyzer pins.

## 6. Color format and byte swapping

| Panel interface | Typical setting |
|---|---|
| RGB565 over SPI / 8-bit parallel (big-endian wire order) | `LV_COLOR_FORMAT_RGB565_SWAPPED` (no manual swap needed) |
| RGB565 over native 16-bit RGB/MIPI bus | `LV_COLOR_FORMAT_RGB565` |
| 24-bit | `LV_COLOR_FORMAT_RGB888` (3 B/px) or `XRGB8888` (4 B/px, usually faster) |
| Transparent screen / compositor | `LV_COLOR_FORMAT_ARGB8888` (or `ARGB8888_PREMULTIPLIED` for Wayland) |
| Monochrome | `LV_COLOR_FORMAT_I1` |

- **Swap in exactly one place.** Options, in order of preference: (1) `RGB565_SWAPPED` format,
  (2) let the SPI host/DMA swap, (3) `lv_draw_rgb565_swap(px_map, px_count)` in `flush_cb`
  (`lv_draw_sw_rgb565_swap` in ≤ 9.5). Combining them undoes the swap and gives wrong colors.
- A display's format can be changed at runtime with `lv_display_set_color_format()`, but then
  buffers must be large enough and `flush_cb` must handle the layout.
- `lv_color_t` is always RGB888 internally in v9 regardless of display format. Do not assume it
  is 16-bit.
- Some panels need RGB/BGR order set in the controller (MADCTL) instead of in LVGL.
- Images should be converted to the display's native format to avoid per-frame conversion.

## 7. Input devices

```c
lv_indev_t * indev = lv_indev_create();
lv_indev_set_type(indev, LV_INDEV_TYPE_POINTER);   /* KEYPAD, ENCODER, BUTTON also exist */
lv_indev_set_read_cb(indev, read_cb);
```

- Read callback: set `data->state` (`LV_INDEV_STATE_PRESSED/RELEASED`) and `data->point` for pointers,
  `data->key` for keypads, `data->enc_diff` for encoders. It runs inside `lv_timer_handler()`.
- Keep reads short; do not block on I2C with long timeouts. Debounce/interrupt-gate if the touch
  controller is slow.
- Keypad/encoder navigation needs **groups**: `lv_group_create()`, `lv_group_add_obj()`,
  `lv_indev_set_group()`. Pointer devices do not need groups.
- Indevs can be bound to a specific display with `lv_indev_set_display()`.
- Gesture and click timing thresholds are configurable (9.5+ API for gesture thresholds; 9.6 adds a
  dedicated double-click time and configurable defaults).

## 8. Configuration

- Start from `lv_conf_template.h` → copy to `lv_conf.h` next to the `lvgl` folder, change the first
  `#if 0` to `#if 1`. In 9.6 this template is generated from Kconfig, so every default matches
  Kconfig; hand-written `lv_conf.h` is still fully supported.
- Set `LV_COLOR_FORMAT_DEFAULT` to match the panel (9.6) / `LV_COLOR_DEPTH` (≤ 9.5).
- `LV_MEM_SIZE` is in bytes (the internal TLSF pool). Too small → random failures when creating
  widgets or loading images.
- `LV_USE_OS` must match your OS to get `lv_lock/lv_unlock` and the OS-aware sleep.
- Disable widgets, fonts, libraries you do not use (flash and RAM).
- Enable `LV_USE_LOG` during bring-up (route to `printf`/ESP_LOG).
- v9.6 `LV_USE_CHECK_ARG` is on by default: invalid public-API arguments log a warning and the call
  returns instead of crashing. `LV_USE_ASSERT_*` are off by default; turn them on while debugging.
- Using Kconfig? Function-like macros (`LV_ASSERT_HANDLER`, `LV_ATTRIBUTE_*`, `LV_FONT_CUSTOM_DECLARE`)
  cannot be Kconfig values; use the per-module `*_USE_CUSTOM_INCLUDE` + `*_CUSTOM_INCLUDE` header.
- Old v9.0 note: keep `lv_conf.h` free of unrelated includes (the migration guide warned that
  `<stdint.h>` there broke assembly-optimized code).
- Build with warnings visible after upgrading: deprecations are `#warning`s.

## 9. ESP32 / ESP-IDF

### Recommended integration

Use Espressif's `esp_lvgl_port` component (LVGL docs call it the recommended way to connect an ESP32
chip's displays and inputs). It supports LVGL v8 and v9 and wraps esp_lcd / touch drivers, adds
touch/encoder/button/USB-HID input, rotation, and runs `lv_timer_handler()` in its own task, so you
do not call it yourself:

```
idf.py add-dependency "espressif/esp_lvgl_port"      # check the registry for the current version
idf.py add-dependency "espressif/esp_lcd_<panel>"    # e.g. a controller driver from esp-bsp
```

Pin the LVGL version in `idf_component.yml` if you do not want automatic upgrades
(`lvgl/lvgl` with a version range such as `^9.*`).

Usage pattern (names per the docs' BSP/port examples; confirm config struct fields against the
installed version's README):

```c
const lvgl_port_cfg_t lvgl_cfg = ESP_LVGL_PORT_INIT_CONFIG();
ESP_ERROR_CHECK(lvgl_port_init(&lvgl_cfg));
/* ...lvgl_port_add_disp(...) / lvgl_port_add_touch(...) with your esp_lcd handles... */

lvgl_port_lock(0);            /* with a BSP this is bsp_display_lock(0) */
build_ui();                   /* every LVGL call outside the LVGL task must be locked */
lvgl_port_unlock();
```

`lvgl_port_lock(timeout_ms)` is the ESP32 equivalent of `lv_lock()`: use it from `app_main`, sensor
tasks, Wi-Fi/MQTT callbacks. A newer Espressif component, `esp_lvgl_adapter`, also exists (unified
display management, tearing control, thread safety, PPA acceleration) — evaluate its README before
choosing between the two.

If you hand-roll the port with esp_lcd: the esp_lcd "color transfer done" callback is the place to
call `lv_display_flush_ready()` (it runs in ISR context, so do nothing else with LVGL there and keep
the callback in IRAM); use `LV_COLOR_FORMAT_RGB565_SWAPPED` for SPI panels.

### `sdkconfig.defaults` tuning (from LVGL's Espressif "tips and tricks")

IDF defaults optimize for size; LVGL benefits from speed:

```ini
CONFIG_COMPILER_OPTIMIZATION_PERF=y        # up to ~30% faster overall, uses SIMD where possible
CONFIG_LV_ATTRIBUTE_FAST_MEM_USE_IRAM=y    # put LVGL's hot code in IRAM
CONFIG_ESP_DEFAULT_CPU_FREQ_MHZ_240=y      # or _360 on P4 (some options need CONFIG_IDF_EXPERIMENTAL_FEATURES=y)
```

PSRAM boards (lets you copy read-only data to PSRAM, and use direct mode + dual buffers):

```ini
CONFIG_SPIRAM=y
CONFIG_SPIRAM_MODE_HEX=y
CONFIG_SPIRAM_USE=y
CONFIG_SPIRAM_ALLOW_BSS_SEG_EXTERNAL_MEMORY=y
CONFIG_SPIRAM_RODATA=y
```

Still keep PARTIAL-mode draw buffers in **internal** RAM when you can; PSRAM is slower and LVGL
touches the buffers constantly.

ESP32-P4 specifics:
- Crash with `esp_msync` after enabling PPA → `CONFIG_LV_DRAW_BUF_ALIGN=64` (PPA needs L1 cache-line
  aligned buffers).
- Buffer underrun / FPS drops with PSRAM + PPA → `CONFIG_SPIRAM_SPEED_200M=y`; optionally raise
  `CONFIG_LV_PPA_BURST_LENGTH` (values 8/16/32/64/128; higher can slow other DMA2D users).

Logging and files:

```ini
CONFIG_LV_USE_LOG=y
CONFIG_LV_LOG_LEVEL_INFO=y
CONFIG_LV_LOG_PRINTF=y
```

Filesystem images (SPIFFS/LittleFS/SD) work via `LV_USE_FS_STDIO` with a drive letter
(`CONFIG_LV_FS_STDIO_LETTER=65` for `A:`), then `lv_image_set_src(img, "A:/spiffs/logo.bin")`.
Put `CONFIG_` settings in `sdkconfig.defaults` (not just menuconfig) so they are tracked in git;
`sdkconfig.<chip>` files apply per chip.

## 10. Pipeline picture

```
 your tasks / ISRs
   |  (lv_lock + LVGL calls, or queue -> LVGL thread)
   v
 +----------------------- lv_timer_handler() (one thread) ----------------------+
 |  indev read timers -> events -> your callbacks                                |
 |  animation timers  -> exec callbacks                                          |
 |  refresh timer     -> render invalidated areas into draw buffer(s)            |
 |                       -> flush_cb(area, px_map) -> start DMA                  |
 +-------------------------------------------------------------------------------+
                                          |
              DMA-complete callback / VSYNC ISR -> lv_display_flush_ready(disp)
```