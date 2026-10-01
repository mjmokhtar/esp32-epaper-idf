[🇮🇩 Bahasa Indonesia](README.md) | 🇬🇧 English

# ePaper Messenger

ESP32 + WeAct Studio 2.13" ePaper display project built with ESP-IDF.  
Displays WiFi info, IP, signal quality, and system info in **250×128 landscape** mode.

---

## Hardware

| Component | Detail |
|---|---|
| MCU | ESP32 (any variant) |
| Display | WeAct Studio 2.13" ePaper (SSD1680) |
| Physical resolution | 122×250 px |
| Logical resolution | 250×122 px (landscape) |
| Color | Black & White |

### Wiring

| ESP32 GPIO | ePaper Pin | Description |
|---|---|---|
| GPIO 23 | DIN (MOSI) | SPI Data |
| GPIO 18 | CLK (SCK) | SPI Clock |
| GPIO 5 | CS | Chip Select |
| GPIO 17 | DC | Data/Command |
| GPIO 16 | RST | Reset |
| GPIO 4 | BUSY | Busy signal |
| 3.3V | VCC | Power |
| GND | GND | Ground |

> Pins can be changed in `components/epaper/epaper.h` under `GPIO Pin Assignment`.

---

## Project Structure

```
epaper_messenger/
│
├── CMakeLists.txt                    ← root build
│
├── main/
│   ├── CMakeLists.txt
│   └── main.c                        ← app_main, WiFi, timer, render
│
└── components/
    ├── epaper/
    │   ├── CMakeLists.txt
    │   ├── epaper.h                  ← API + pin configuration
    │   └── epaper.c                  ← SPI driver, SSD1680 init sequence
    │
    └── epaper_gfx/
        ├── CMakeLists.txt
        ├── epaper_gfx.h              ← API for shapes, text, bitmap
        └── epaper_gfx.c              ← 5×7 font, graphics primitives
```

---

## Configuration

Edit this section in `main/main.c`:

```c
#define WIFI_SSID           "NamaWiFiKamu"
#define WIFI_PASSWORD       "PasswordKamu"
#define WIFI_MAX_RETRY      5
#define REFRESH_INTERVAL_MS (2 * 60 * 1000)   // refresh interval in ms
```

To change the GPIO pins, edit `components/epaper/epaper.h`:

```c
#define EPD_PIN_MOSI    23
#define EPD_PIN_CLK     18
#define EPD_PIN_CS      5
#define EPD_PIN_DC      17
#define EPD_PIN_RST     16
#define EPD_PIN_BUSY    4
```

---

## Build & Flash

```bash
# Set target
idf.py set-target esp32

# Build
idf.py build

# Flash + monitor
idf.py -p /dev/ttyUSB0 flash monitor
```

---

## Features

- **WiFi Station** — connects to WiFi with automatic retry (max 5x, 10-second timeout)
- **Boot screen** — shows "Connecting..." while WiFi is connecting
- **WiFi info** — SSID, IP address, RSSI in dBm and percentage
- **Signal bar** — 5-bar signal quality indicator like a phone's
- **Periodic refresh** — updates RSSI and re-renders every N minutes (default 2 minutes)
- **System info** — uptime, free heap, refresh interval
- **Panel deep sleep** — after rendering finishes, the panel enters deep sleep (<1µA)
- **Landscape mode** — 250×128 px orientation

---

## GFX API

### Text

```c
// Draw a single character, returns the x advance
int epd_draw_char(int x, int y, char c, epd_font_t font, uint8_t color);

// Draw a string with automatic word-wrap and \n support
// Returns the final Y
int epd_draw_string(int x, int y, const char *str, epd_font_t font, uint8_t color);
```

Available fonts:
```c
FONT_SMALL   // 5×7 px  (1× scale)
FONT_MEDIUM  // 10×14 px (2× scale)
FONT_LARGE   // 15×21 px (3× scale)
```

### Shapes

```c
void epd_draw_hline(int x, int y, int w, uint8_t color);
void epd_draw_vline(int x, int y, int h, uint8_t color);
void epd_draw_line(int x0, int y0, int x1, int y1, uint8_t color);

void epd_draw_rect(int x, int y, int w, int h, uint8_t color);
void epd_fill_rect(int x, int y, int w, int h, uint8_t color);
void epd_draw_round_rect(int x, int y, int w, int h, int r, uint8_t color);
void epd_fill_round_rect(int x, int y, int w, int h, int r, uint8_t color);

void epd_draw_circle(int cx, int cy, int r, uint8_t color);
void epd_fill_circle(int cx, int cy, int r, uint8_t color);

void epd_draw_triangle(int x0, int y0, int x1, int y1, int x2, int y2, uint8_t color);
void epd_fill_triangle(int x0, int y0, int x1, int y1, int x2, int y2, uint8_t color);
```

### Bitmap

```c
// Format: 1bpp MSB-first (matches GIMP monochrome export)
void epd_draw_bitmap(int x, int y, const uint8_t *bitmap, int w, int h, uint8_t color);
```

### ePaper Core

```c
void epd_init(void);          // init SPI + GPIO + panel, call once
void epd_clear_buffer(void);  // fill the buffer with white
void epd_display(void);       // send buffer to the panel + trigger refresh (~2-8 seconds)
void epd_clear_screen(void);  // clear + display in one call
void epd_sleep(void);         // panel deep sleep <1µA
void epd_wake(void);          // wake the panel (hw reset + re-init)
void epd_draw_pixel(int x, int y, uint8_t color); // set a pixel in the buffer
uint8_t *epd_get_buffer(void);  // direct access to the framebuffer
```

---

## Technical Notes

### Why is `epd_display()` slow?

The SSD1680 performs a full refresh that physically moves ink particles. This process takes ~2–8 seconds and cannot be sped up. It is a characteristic of ePaper hardware, not a bug.

### Why doesn't the timer render directly?

```
Timer callback → set flag → main loop → render
```

The timer callback runs in the FreeRTOS timer daemon task with a limited stack (~2KB).
ePaper rendering needs a larger stack and blocks for ~2-8 seconds.
Calling render directly from the callback would cause a stack overflow or a watchdog reset.

### Panel deep sleep

After `epd_sleep()`, the panel cannot accept any command.
`epd_wake()` performs a hardware reset + a full re-init before it can display again.
The framebuffer in the ESP32's RAM stays safe — it is not lost while the panel sleeps.

# Preview Results

Portrait (122×250)
---
![Portrait](images/Capture.PNG)

Landscape (250×122)
---
![Landscape](images/image.jpeg)


### Changing Display Orientation (Portrait ↔ Landscape)

Only **1 file** needs to be edited — `components/epaper/epaper.h`.  
`epaper.c` already uses `#if`, so the transform follows automatically.

#### `components/epaper/epaper.h` — uncomment one

```c
// ─── Panel Physical Specs ─────────────────────────────────────────────────────
#define EPD_PHYSICAL_WIDTH   122
#define EPD_PHYSICAL_HEIGHT  250
#define EPD_BUF_WIDTH        ((EPD_PHYSICAL_WIDTH + 7) / 8)        // 16
#define EPD_BUF_SIZE         (EPD_BUF_WIDTH * EPD_PHYSICAL_HEIGHT) // 4000

// ─── Choose orientation — uncomment one ──────────────────────────────────────

// PORTRAIT (122×250)
// #define EPD_WIDTH    122
// #define EPD_HEIGHT   250

// LANDSCAPE (250×122) ← active by default
#define EPD_WIDTH    250
#define EPD_HEIGHT   122
```

#### `components/epaper/epaper.c` — no changes needed

`epd_draw_pixel()` already handles both orientations automatically via `#if`:

```c
void epd_draw_pixel(int x, int y, uint8_t color)
{
    if (x < 0 || x >= EPD_WIDTH || y < 0 || y >= EPD_HEIGHT) return;

#if (EPD_WIDTH == 250)
    // LANDSCAPE (250×122) — Case 5
    int px = y;
    int py = x;
#else
    // PORTRAIT (122×250) — flip X and Y
    int px = (EPD_PHYSICAL_WIDTH  - 1) - x;
    int py = (EPD_PHYSICAL_HEIGHT - 1) - y;
#endif

    int byte_idx = py * EPD_BUF_WIDTH + (px / 8);
    int bit_pos  = 7 - (px % 8);

    if (color == EPD_BLACK)
        s_buf[byte_idx] &= ~(1 << bit_pos);
    else
        s_buf[byte_idx] |=  (1 << bit_pos);
}
```

Summary:

| | Portrait | Landscape |
|---|---|---|
| `EPD_WIDTH` | `122` | `250` |
| `EPD_HEIGHT` | `250` | `122` |
| `EPD_PHYSICAL_WIDTH` | `122` (unchanged) | `122` (unchanged) |
| `EPD_PHYSICAL_HEIGHT` | `250` (unchanged) | `250` (unchanged) |
| `EPD_BUF_SIZE` | `4000` (unchanged) | `4000` (unchanged) |
| `px` transform | `(122-1) - x` | `y` |
| `py` transform | `(250-1) - y` | `x` |

> The physical buffer, init sequence, LUT, and all other code **need no changes** at all.

### GFX adapted from Adafruit GFX

The primitive algorithms (Bresenham, Midpoint circle, scanline triangle, fillCircleHelper)
are adapted from the [Adafruit-GFX-Library](https://github.com/adafruit/Adafruit-GFX-Library)
and ported to pure C for ESP-IDF without any dependency on the Arduino framework.

---

## Dependencies

- ESP-IDF v5.x
- Built-in components: `esp_wifi`, `esp_event`, `esp_netif`, `esp_timer`, `nvs_flash`, `driver`

---

## License

MIT — free to use and modify.

---

*Made by Toaster Co Ltd — 2026*
