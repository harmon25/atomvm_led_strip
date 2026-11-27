# AGENTS.md

This repo is an AtomVM Erlang library **plus** an ESP-IDF C Nif for driving
WS2812/SK6812 LED strips on ESP32. It is consumed two ways:

1. As an ESP-IDF component built into the AtomVM image (the Nif).
2. As a rebar3 Erlang library (`led_strip`) used by downstream AtomVM apps.

## Build systems (dual — do not confuse them)

- **Erlang library** (`src/led_strip.erl`, `src/led_strip.app.src`): built with
  rebar3 via the `atomvm_rebar3_plugin` (see `rebar.config`). No deps.
  The user-facing API lives in `src/led_strip.erl`; it calls the Nif.
- **C Nif** (`nifs/atomvm_led_strip.c`, `nifs/led_strip_driver.c`): built as an
  ESP-IDF component. Registered via `CMakeLists.txt` (IDF 5.x) and
  `component.mk` (legacy IDF 4.x). Requires ESP-IDF 5.x per `idf_component.yml`.
  Depends on the `espressif/led_strip` component (^3.0.2) and AtomVM's
  `libatomvm`/`avm_sys`.

The Erlang side and C side are versioned/developed together but built
separately. Changing the Nif API requires matching changes in `led_strip.erl`.

## Backend / Kconfig

LED strip backend is selectable via Kconfig (`Kconfig`):
`AVM_LED_STRIP_BACKEND_AUTO|SPI|RMT`, default Auto. Auto picks SPI+DMA on
ESP32 (best WiFi coexistence) and RMT+DMA on S3/C6/P4. Don't assume RMT —
the README's "new RMT driver API" note predates the SPI-backend addition.

## No tests / lint / typecheck

There is no test suite, lint config, formatter, or CI in this repo. Do not
invent commands. Verify Erlang compiles with `rebar3 compile` (needs the
atomvm_rebar3_plugin available). C builds only inside an ESP-IDF environment
that also has AtomVM's components; there is no standalone build here — see
the [AtomVM Build Instructions](https://doc.atomvm.net/build-instructions.html).

## Layout

- `src/` — Erlang `led_strip` library (the API users call).
- `nifs/` — C Nif sources; `nifs/include/atomvm_led_strip.h` is the public Nif header.
- `examples/led_strip_example/` — standalone rebar3 app demoing usage.
- `markdown/led_strip.md` — programmer's guide (authoritative API detail).

## Conventions

- Erlang module is `led_strip`; Elixir callers use `:led_strip`.
- `render_rgb/2,3` and `render_rgbw/2,3` are the fastest path for per-frame
  animation: they accept a packed binary (`<<R,G,B,...>>` / `<<R,G,B,W,...>>`),
  bulk-set pixels, and refresh in one gen_server call + one NIF. Use these for
  animation loops that update the whole strip each frame. The list-based
  `set_pixels_*/2,3` + `refresh/1` APIs remain for staged updates.
- License headers: Apache-2.0 (some files also LGPL-2.1-or-later, per SPDX
  in `CMakeLists.txt`). Preserve SPDX lines when editing C files.
- `examples/led_strip_example` pins this repo via git main branch; keep the
  example in sync with `src/` API changes.
