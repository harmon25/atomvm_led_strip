# AtomVM LED Strip Library

This AtomVM Erlang library and Nif can be used to control WS2812 and SK6812 LED strips using the ESP32 SoC for any Erlang/Elixir programs targeted for AtomVM on the ESP32 platform.

## Features

- **RGB LED support** (WS2812, WS2812B) - 24-bit color (3 bytes per pixel)
- **RGBW LED support** (SK6812) - 32-bit color with dedicated white channel (4 bytes per pixel)
- **Global brightness control** - Efficient brightness scaling (0-255) applied at refresh time
- **Multiple color spaces** - RGB, RGBW, HSV, and HSVW
- **Batch operations** - Fill entire strip or set multiple pixels in a single call
- **Binary rendering** - `render_rgb`/`render_rgbw` combine set + refresh in one call with packed binary input (fastest path for per-frame animation)
- **Multiple strips** - Drive multiple LED strips simultaneously on different GPIO pins
- **ESP-IDF 5.x compatible** - Uses the new RMT driver API

This Nif is included as an add-on to the AtomVM base image.  In order to use this Nif in your AtomVM program, you must be able to build the AtomVM virtual machine, which in turn requires installation of the Espressif IDF SDK and tool chain.

## Quick Start

### Basic Usage (Erlang)

```erlang
%% Start a 4-pixel RGB LED strip on GPIO pin 18
{ok, LedStrip} = led_strip:start(18, 4),

%% Set individual pixels using RGB (0-255 per channel)
ok = led_strip:set_pixel_rgb(LedStrip, 0, 255, 0, 0),    %% Red
ok = led_strip:set_pixel_rgb(LedStrip, 1, 0, 255, 0),    %% Green
ok = led_strip:set_pixel_rgb(LedStrip, 2, 0, 0, 255),    %% Blue
ok = led_strip:set_pixel_rgb(LedStrip, 3, 255, 255, 0),  %% Yellow

%% Refresh to display changes
ok = led_strip:refresh(LedStrip).
```

### Basic Usage (Elixir)

```elixir
# Start a 4-pixel RGB LED strip on GPIO pin 18
{:ok, led_strip} = :led_strip.start(18, 4)

# Set individual pixels using RGB (0-255 per channel)
:ok = :led_strip.set_pixel_rgb(led_strip, 0, 255, 0, 0)    # Red
:ok = :led_strip.set_pixel_rgb(led_strip, 1, 0, 255, 0)    # Green
:ok = :led_strip.set_pixel_rgb(led_strip, 2, 0, 0, 255)    # Blue
:ok = :led_strip.set_pixel_rgb(led_strip, 3, 255, 255, 0)  # Yellow

# Refresh to display changes
:ok = :led_strip.refresh(led_strip)
```

### RGBW Strip (SK6812)

```erlang
%% Start a 4-pixel RGBW strip
{ok, LedStrip} = led_strip:start(18, 4, #{led_type => rgbw}),

%% Set pixels with RGBW (includes dedicated white channel)
ok = led_strip:set_pixel_rgbw(LedStrip, 0, 255, 0, 0, 0),    %% Pure red
ok = led_strip:set_pixel_rgbw(LedStrip, 1, 0, 0, 0, 255),    %% Pure white LED
ok = led_strip:refresh(LedStrip).
```

### HSV Color Space

```erlang
%% HSV: Hue (0-359), Saturation (0-100), Value (0-100)
ok = led_strip:set_pixel_hsv(LedStrip, 0, 0, 100, 50),    %% Red at 50% brightness
ok = led_strip:set_pixel_hsv(LedStrip, 1, 120, 100, 50),  %% Green
ok = led_strip:set_pixel_hsv(LedStrip, 2, 240, 100, 50),  %% Blue
ok = led_strip:refresh(LedStrip).
```

### Fill Entire Strip

```erlang
%% Fill all pixels with the same color
ok = led_strip:fill_rgb(LedStrip, 255, 0, 255),  %% Magenta
ok = led_strip:refresh(LedStrip).

%% Or with HSV
ok = led_strip:fill_hsv(LedStrip, 180, 100, 50), %% Cyan
ok = led_strip:refresh(LedStrip).
```

### Set Multiple Pixels at Once

```erlang
%% Set multiple pixels with a list of RGB tuples
Colors = [{255, 0, 0}, {0, 255, 0}, {0, 0, 255}, {255, 255, 0}],
ok = led_strip:set_pixels_rgb(LedStrip, Colors),
ok = led_strip:refresh(LedStrip).

%% With explicit starting index
ok = led_strip:set_pixels_rgb(LedStrip, Colors, 0),
ok = led_strip:refresh(LedStrip).
```

### Render from a Binary (set + refresh in one call)

For the fastest per-frame animation, pack pixel values into a binary and
use `render_rgb`/`render_rgbw` — set and refresh happen in a single
`gen_server` call, and the NIF processes a flat binary instead of walking
Erlang tuples.

```erlang
%% Packed RGB binary: <<R,G,B, R,G,B, ...>>
Bin = <<255,0,0, 0,255,0, 0,0,255, 255,255,0>>,
ok = led_strip:render_rgb(LedStrip, Bin).

%% Starting at offset 5
ok = led_strip:render_rgb(LedStrip, 5, Bin).

%% RGBW strips: <<R,G,B,W, R,G,B,W, ...>>
BinW = <<255,0,0,0, 0,255,0,128, 0,0,255,255>>,
ok = led_strip:render_rgbw(LedStrip, BinW).
```

```elixir
# Elixir
bin = <<255,0,0, 0,255,0, 0,0,255, 255,255,0>>
:ok = :led_strip.render_rgb(led_strip, bin)
```

### Brightness Control

```erlang
%% Set global brightness (0-255, default is 255)
ok = led_strip:set_brightness(LedStrip, 128),  %% 50% brightness

%% Get current brightness
Brightness = led_strip:get_brightness(LedStrip).
```

### Rainbow Animation Example

```erlang
-module(rainbow).
-export([start/0]).

-define(PIN, 18).
-define(NUM_PIXELS, 4).

start() ->
    {ok, LedStrip} = led_strip:start(?PIN, ?NUM_PIXELS),
    ok = led_strip:clear(LedStrip),
    %% Start a rainbow cycle on each pixel
    lists:foreach(
        fun(I) ->
            spawn(fun() -> loop(LedStrip, I, I * 90, 50) end)
        end,
        lists:seq(0, ?NUM_PIXELS - 1)
    ),
    timer:sleep(infinity).

loop(LedStrip, Index, Hue, DelayMs) ->
    ok = led_strip:set_pixel_hsv(LedStrip, Index, Hue, 100, 20),
    ok = led_strip:refresh(LedStrip),
    timer:sleep(DelayMs),
    loop(LedStrip, Index, (Hue + 1) rem 360, DelayMs).
```

## API Reference

### Lifecycle Functions

| Function | Description |
|----------|-------------|
| `start(Pin, NumPixels)` | Start an RGB LED strip driver |
| `start(Pin, NumPixels, Options)` | Start with options (led_type, timeout) |
| `stop(LedStrip)` | Stop the LED strip driver |
| `clear(LedStrip)` | Turn off all pixels |
| `refresh(LedStrip)` | Push pixel buffer to the strip |

### Pixel Functions

| Function | Description |
|----------|-------------|
| `set_pixel_rgb(LedStrip, I, R, G, B)` | Set pixel I to RGB color |
| `set_pixel_rgbw(LedStrip, I, R, G, B, W)` | Set pixel I to RGBW color (SK6812 only) |
| `set_pixel_hsv(LedStrip, I, H, S, V)` | Set pixel I to HSV color |
| `set_pixel_hsvw(LedStrip, I, H, S, V, W)` | Set pixel I to HSVW color (SK6812 only) |

### Batch Functions

| Function | Description |
|----------|-------------|
| `fill_rgb(LedStrip, R, G, B)` | Fill all pixels with RGB color |
| `fill_rgbw(LedStrip, R, G, B, W)` | Fill all pixels with RGBW color |
| `fill_hsv(LedStrip, H, S, V)` | Fill all pixels with HSV color |
| `fill_hsvw(LedStrip, H, S, V, W)` | Fill all pixels with HSVW color |
| `set_pixels_rgb(LedStrip, ColorList)` | Set multiple pixels from list of {R,G,B} tuples |
| `set_pixels_rgbw(LedStrip, ColorList)` | Set multiple pixels from list of {R,G,B,W} tuples |
| `render_rgb(LedStrip, Binary)` | Set pixels from packed `<<R,G,B,...>>` binary and refresh in one call |
| `render_rgb(LedStrip, Offset, Binary)` | Like `render_rgb/2` starting at `Offset` |
| `render_rgbw(LedStrip, Binary)` | Set pixels from packed `<<R,G,B,W,...>>` binary and refresh (SK6812 only) |
| `render_rgbw(LedStrip, Offset, Binary)` | Like `render_rgbw/2` starting at `Offset` |

### Brightness Functions

| Function | Description |
|----------|-------------|
| `set_brightness(LedStrip, Brightness)` | Set global brightness (0-255) |
| `get_brightness(LedStrip)` | Get current brightness setting |

### Options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `led_type` | `rgb \| rgbw` | `rgb` | LED strip type |
| `timeout` | `non_neg_integer()` | `100` | Internal communication timeout (ms) |
| `brightness` | `0..255` | `255` | Initial brightness level |

## Documentation

* [Programmer's Guide](markdown/led_strip.md) - Detailed usage guide
* [LED Strip Example Program](examples/led_strip_example/README.md) - Complete example application

## Building

For instructions on building AtomVM with this component, see the [AtomVM Build Instructions](https://doc.atomvm.net/build-instructions.html).

## License

Apache License 2.0
