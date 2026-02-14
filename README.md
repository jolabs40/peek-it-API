# peek-it API Documentation

> **If the Designer isn't enough for you...** if you're the kind of person who reads the ingredients on cereal boxes, who prefers `curl` over clicking buttons, who thinks a GUI is just a pretty face hiding the *real* power underneath — then welcome. You've found the heart of the beast.
>
> peek-it TV exposes a full REST API on port **8081** that lets you do everything the Designer does, and then some. Display notifications, manage templates, upload sounds, control Home Assistant entities, speak text aloud, draw charts, and even build TV menus navigable with a remote — all through simple HTTP calls.
>
> No SDK required. No library to install. Just you, your favorite HTTP client, and this document.
>
> *May the JSON be with you.*

---

## Table of Contents

- [Quick Start](#quick-start)
- [Authentication](#authentication)
- [1. Notifications](#1-notifications)
  - [Display a Notification](#display-a-notification)
  - [Close a Notification](#close-a-notification)
  - [Display from Template](#display-from-template)
  - [Parameter Substitution](#parameter-substitution)
  - [Button Callbacks](#button-callbacks)
- [2. Widget Types](#2-widget-types)
  - [text](#text)
  - [image](#image)
  - [circle](#circle)
  - [video](#video)
  - [webview](#webview)
  - [svg](#svg)
  - [rect / ellipse / hexagon](#rect--ellipse--hexagon)
  - [line / arrow](#line--arrow)
  - [button](#button)
  - [menu](#menu)
- [3. Widget Style Reference](#3-widget-style-reference)
  - [Position & Size](#position--size)
  - [Colors & Appearance](#colors--appearance)
  - [Typography](#typography)
  - [Focus States](#focus-states)
  - [Shadows](#shadows)
  - [Animations](#animations)
- [4. Animations (In/Out)](#4-animations-inout)
- [5. Sound](#5-sound)
  - [Sound in Notifications](#sound-in-notifications)
  - [List Sounds](#list-sounds)
  - [Upload Custom Sound](#upload-custom-sound)
  - [Delete Custom Sound](#delete-custom-sound)
  - [Preview a Sound](#preview-a-sound)
  - [Default Sound Config](#default-sound-config)
- [6. Text-to-Speech (TTS)](#6-text-to-speech-tts)
  - [Speak Text](#speak-text)
  - [Stop TTS](#stop-tts)
  - [TTS in Notifications](#tts-in-notifications)
- [7. Templates](#7-templates)
  - [Template Types](#template-types)
  - [List Templates](#list-templates)
  - [Load a Template](#load-a-template)
  - [Save a Template](#save-a-template)
  - [Delete a Template](#delete-a-template)
  - [Rename a Template](#rename-a-template)
  - [Move a Template](#move-a-template)
  - [Export All (ZIP)](#export-all-zip)
  - [Import from ZIP](#import-from-zip)
- [8. SVG Management](#8-svg-management)
- [9. Home Assistant Integration](#9-home-assistant-integration)
  - [Configuration](#ha-configuration)
  - [Entity Widget (Real-time)](#entity-widget-real-time)
  - [Entity Info](#entity-info)
  - [Entity History](#entity-history)
  - [Chart Widget](#chart-widget)
  - [Camera Snapshot](#camera-snapshot)
- [10. Menu Widget (TV Overlay)](#10-menu-widget-tv-overlay)
  - [Menu Structure](#menu-structure)
  - [Menu Item Types](#menu-item-types)
  - [D-pad Navigation](#d-pad-navigation)
  - [Menu Styling](#menu-styling)
  - [Toggle Polling](#toggle-polling)
- [11. Configuration](#11-configuration)
  - [Clock Overlay](#clock-overlay)
  - [Screen Dimming](#screen-dimming)
  - [Admin Mode](#admin-mode)
  - [Start Menu Override](#start-menu-override)
  - [Language](#language)
  - [Home Assistant Token](#home-assistant-token)
  - [API Key Management](#api-key-management)
- [12. Service Status](#12-service-status)
- [13. Logs](#13-logs)
- [14. Internationalization](#14-internationalization)
- [15. Overlay Behavior](#15-overlay-behavior)
  - [Notification Stack](#notification-stack)
  - [Anti Burn-in](#anti-burn-in)
  - [WebView Synchronization](#webview-synchronization)
- [16. Limits & Constraints](#16-limits--constraints)
- [17. Error Responses](#17-error-responses)

---

## Quick Start

peek-it TV runs an HTTP server on port **8081** (configurable). All endpoints accept and return JSON unless stated otherwise.

**Base URL:** `http://<TV_IP>:8081`

**Display "Hello World" on the TV in 3 seconds:**

```bash
curl -X POST http://192.168.1.100:8081/api/notify \
  -H "Content-Type: application/json" \
  -H "X-API-Key: YOUR_KEY" \
  -d '{
    "action": "DISPLAY",
    "duration": 5000,
    "elements": [
      {
        "type": "text",
        "content": "Hello World!",
        "style": { "left": 30, "top": 40, "width": 40, "height": 10, "size": 40, "color": "#FFFFFF" }
      }
    ]
  }'
```

That's it. You just pushed pixels to a TV from a terminal. Feel the power.

---

## Authentication

peek-it uses a simple API key mechanism. An API key is auto-generated on first launch and can be changed via the Designer or the API.

| Header | Value |
|--------|-------|
| `X-API-Key` | Your API key string |

**Public endpoints** (no authentication required):

| Endpoint | Description |
|----------|-------------|
| `GET /` | Designer web UI |
| `GET /index.html` | Designer web UI |
| `GET /api/status` | Service status |
| `GET /api` | Endpoint list |
| `GET /api/config/language` | Language preference |
| `GET /locales/{lang}.json` | Locale files |
| `GET /api/fonts/mdi.ttf` | Material Design Icons font |
| `GET /api/ha/entity` | HA entity widget |
| `GET /api/ha/chart` | HA chart widget |
| `GET /api/ha/history` | HA entity history |
| `GET /api/camera/snapshot` | Camera snapshot |

All other endpoints require the `X-API-Key` header.

**CORS:** The server reflects the `Origin` header from the request (no wildcard `*`).

---

## 1. Notifications

### Display a Notification

```
POST /api/notify
```

The main event. This endpoint is where the magic happens — send a JSON payload and watch your TV light up.

**Full payload structure:**

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `action` | string | `"DISPLAY"` | `DISPLAY` to show, `CLOSE` to dismiss |
| `duration` | integer | `10000` | Display time in milliseconds. `0` = persistent (stays until closed) |
| `source` | string | `"UNKNOWN"` | Source identifier (for logging) |
| `animationIn` | string | `"fade"` | Entry animation (see [Animations](#4-animations-inout)) |
| `animationOut` | string | `"fade"` | Exit animation |
| `priority` | string | `"normal"` | `normal`, `high`, or `low` |
| `sound` | string | *null* | Sound filename (e.g. `"01_notify.wav"`). `"none"` = silent |
| `soundVolume` | float | `1.0` | Volume 0.0 to 1.0 |
| `tts` | string | *null* | Text to speak aloud (see [TTS](#6-text-to-speech-tts)) |
| `ttsLang` | string | `"fr-FR"` | TTS language code |
| `ttsSpeed` | float | `1.0` | TTS speed (0.5 to 2.0) |
| `ttsPitch` | float | `1.0` | TTS pitch (0.5 to 2.0) |
| `ttsVolume` | float | `1.0` | TTS volume (0.0 to 1.0) |
| `template_id` | string | *null* | Load elements from a saved template UUID |
| `params` | object | *null* | Key-value pairs for parameter substitution |
| `callback_url` | string | *null* | URL to POST when a button is pressed |
| `elements` | array | `[]` | Array of widget elements (see [Widget Types](#2-widget-types)) |

**Response:** `{"status": "ok"}`

### Close a Notification

```bash
curl -X POST http://TV:8081/api/notify \
  -H "X-API-Key: KEY" \
  -d '{"action": "CLOSE"}'
```

Closes the topmost notification. If notifications are stacked, the one underneath resumes.

### Display from Template

Instead of sending all elements in the payload, reference a saved template by its UUID:

```json
{
  "action": "DISPLAY",
  "duration": 8000,
  "template_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

The server loads the template's elements and uses them. If both `template_id` and `elements` are provided, `elements` takes priority (non-empty wins).

### Parameter Substitution

Templates can define **parameter placeholders** via `paramKey` and `actionParamKey` on elements. When you send a notification with `params`, the values are injected.

**Template element:**
```json
{
  "type": "text",
  "content": "Default title",
  "paramKey": "title",
  "style": { "left": 10, "top": 10, "width": 80, "height": 10, "size": 24 }
}
```

**Notification payload:**
```json
{
  "template_id": "...",
  "params": { "title": "Breaking News!" }
}
```

The text widget will display "Breaking News!" instead of "Default title".

You can also use **double-brace placeholders** `{{key}}` inside content strings — they will be replaced by the corresponding `params` value.

### Button Callbacks

When a widget has an `action` field and the user presses it (via remote/D-pad), peek-it sends an HTTP POST to the `callback_url`:

```json
// POST to callback_url
{ "action": "the_action_string" }
```

This enables interactive notifications — buttons that trigger automations, Tasker tasks, or any HTTP endpoint.

---

## 2. Widget Types

Every notification is composed of **elements** — visual widgets positioned on screen. Each element has a `type`, optional `content`, and a `style` object controlling its appearance.

### text

Displays text content. Supports **Material Design Icons** (MDI) — prefix with `mdi:` to render an icon instead of text.

```json
{
  "type": "text",
  "content": "Hello World",
  "style": { "left": 10, "top": 10, "width": 80, "height": 10, "size": 24, "color": "#FFFFFF", "align": "center" }
}
```

**MDI icon example:**
```json
{
  "type": "text",
  "content": "mdi:weather-sunny",
  "style": { "left": 45, "top": 30, "width": 10, "height": 10, "size": 48, "color": "#FFD700" }
}
```

### image

Displays an image from a URL. Loaded via Glide (supports JPEG, PNG, GIF, WebP). Also supports MDI icons.

```json
{
  "type": "image",
  "content": "https://example.com/photo.jpg",
  "style": { "left": 10, "top": 10, "width": 20, "height": 30, "radius": 12 }
}
```

### circle

A circular image or MDI icon. Think profile pictures, status indicators.

```json
{
  "type": "circle",
  "content": "https://example.com/avatar.jpg",
  "style": { "left": 5, "top": 5, "width": 8, "height": 8 }
}
```

### video

Plays an RTSP or HTTP video stream via ExoPlayer (Media3). Audio is **disabled** by design — this is for surveillance cameras, live feeds, etc.

```json
{
  "type": "video",
  "content": "rtsp://192.168.1.50:554/stream1",
  "style": { "left": 0, "top": 0, "width": 50, "height": 50 }
}
```

- **RTSP:** Uses UDP transport (no TCP fallback)
- **Snapshot caching:** The last frame is cached for instant display on next load
- **Buffering:** Widget is hidden off-screen during buffering, then slides in

### webview

Embeds a full web page. JavaScript is enabled, but permissions (camera, mic, location) are **denied** for security.

```json
{
  "type": "webview",
  "content": "https://example.com/dashboard",
  "style": { "left": 0, "top": 0, "width": 100, "height": 100 }
}
```

- **Ready signal:** The page can call `PeekBridge.notifyReady()` to tell peek-it the content is loaded. If not called, a 2-second timeout is used.
- **Transparent background:** WebView background is transparent by default
- **User-Agent:** Appended with `PeekIt-Viewer/1.0`

### svg

Displays an SVG image via WebView. The SVG must be uploaded first via `/api/svg/upload`.

```json
{
  "type": "svg",
  "content": "/api/svg/serve?name=my_icon.svg",
  "style": { "left": 10, "top": 10, "width": 10, "height": 10 }
}
```

### rect / ellipse / hexagon

Shape primitives. Use them for backgrounds, decorative elements, or layout building blocks.

```json
{
  "type": "rect",
  "style": { "left": 5, "top": 5, "width": 90, "height": 90, "bgColor": "#CC1a1a2e", "radius": 16, "borderWidth": 2, "borderColor": "#42A5F5" }
}
```

| Type | Shape |
|------|-------|
| `rect` / `box` | Rectangle (with optional corner radius) |
| `ellipse` | Oval |
| `hexagon` | Custom hexagon path |

### line / arrow

Drawing primitives for connectors, decorative lines, or flow indicators.

```json
{
  "type": "line",
  "style": { "left": 10, "top": 50, "width": 80, "height": 0.5, "bgColor": "#42A5F5" }
}
```

For arrows, you can use `x2`/`y2` style properties to define the endpoint.

### button

An interactive, focusable widget. Automatically receives focus styling and responds to D-pad/remote input.

```json
{
  "type": "button",
  "content": "Click Me",
  "action": "do_something",
  "focusable": true,
  "style": {
    "left": 40, "top": 80, "width": 20, "height": 8,
    "bgColor": "#4CAF50", "color": "#FFFFFF", "size": 18, "radius": 8,
    "focusBgColor": "#66BB6A", "focusColor": "#FFFFFF"
  }
}
```

When pressed, sends `{"action": "do_something"}` to the `callback_url`.

### menu

A full TV overlay menu with D-pad navigation. See [Menu Widget](#10-menu-widget-tv-overlay) for details.

---

## 3. Widget Style Reference

All style values are set in the `style` object of each element.

### Position & Size

All positions and sizes are **percentages of the screen** (0-100).

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `left` | float | `0` | Horizontal position (% of screen width) |
| `top` | float | `0` | Vertical position (% of screen height) |
| `width` | float | `10` | Width (% of screen width) |
| `height` | float | `10` | Height (% of screen height) |
| `angle` | float | `0` | Rotation in degrees |

**Example:** A widget at `left: 50, top: 50` is centered on screen. A widget with `width: 100, height: 100` covers the entire screen.

### Colors & Appearance

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `color` | string | `"#FFFFFF"` | Text / foreground color |
| `bgColor` | string | `"#00000000"` | Background color (supports alpha: `#AARRGGBB`) |
| `opacity` | float | `1.0` | Overall opacity (0.0 = invisible, 1.0 = opaque) |
| `radius` | integer | `0` | Corner radius in dp |
| `borderWidth` | integer | `0` | Border width in dp |
| `borderColor` | string | `"#00000000"` | Border color |

**Alpha channel tip:** Use 2-digit hex prefix for transparency: `#CC000000` = 80% opaque black, `#00000000` = fully transparent.

### Typography

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `size` | integer | `20` | Font size in sp |
| `font` | string | `"Roboto"` | Font family name |
| `weight` | string | `"normal"` | `"normal"` or `"bold"` |
| `align` | string | `"center"` | `"left"`, `"center"`, or `"right"` |

### Focus States

For interactive widgets (buttons, focusable elements):

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `focusable` | boolean | `false` | Can receive D-pad focus |
| `directFocus` | boolean | `false` | Grab focus immediately on display |
| `focusColor` | string | `"#FFFFFF"` | Text color when focused |
| `focusBgColor` | string | `"#00000000"` | Background color when focused |

### Shadows

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `shadowColor` | string | `"#000000"` | Shadow color |
| `shadowOpacity` | float | `0.5` | Shadow opacity |
| `shadowBlur` | integer | `0` | Shadow blur radius (dp) |
| `shadowOffsetX` | float | `0` | Horizontal offset (dp) |
| `shadowOffsetY` | float | `4` | Vertical offset (dp) |

### Animations

Per-widget looping animations:

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `animation` | string | `""` | `"spin"`, `"pulse"`, `"bounce"`, or `""` (none) |
| `animationSpeed` | float | `2.0` | Duration of one animation cycle (seconds) |

| Animation | Effect |
|-----------|--------|
| `spin` | Continuous 360 degree rotation |
| `pulse` | Fade between 30% and 100% opacity |
| `bounce` | Vertical bounce (-20px) |

---

## 4. Animations (In/Out)

Entry and exit animations for the entire notification overlay.

| Value | Entry Effect | Exit Effect |
|-------|-------------|-------------|
| `fade` | Opacity 0 to 1 | Opacity 1 to 0 |
| `slide_right` | Slide in from right (+500px) | Slide out to right |
| `slide_left` | Slide in from left (-500px) | Slide out to left |
| `slide_bottom` | Slide in from bottom (+500px) | Slide out to bottom |
| `slide_top` | Slide in from top (-500px) | Slide out to top |
| `pop` | Scale from 0.8 to 1.0 | Scale from 1.0 to 0.8 |
| `none` | Instant (no animation) | Instant |

**Timing:**
- Entry: 400ms with decelerate interpolation
- Exit: 300ms with accelerate interpolation

---

## 5. Sound

### Sound in Notifications

Add sound to any notification via the payload:

```json
{
  "action": "DISPLAY",
  "sound": "01_notify.wav",
  "soundVolume": 0.8,
  "elements": [...]
}
```

| Value | Behavior |
|-------|----------|
| `"01_notify.wav"` | Play this specific sound file |
| `"none"` | Silent (no sound, overrides default) |
| *omitted* | Use the TV's default sound setting |

### List Sounds

```
GET /api/sounds/list
```

**Response:**
```json
{
  "official": ["01_notify.wav", "02_alert.mp3", "03_chime.ogg", ...],
  "custom": ["my_sound.mp3", "doorbell.wav"]
}
```

Official sounds ship with the app (read-only). Custom sounds are user-uploaded.

### Upload Custom Sound

```
POST /api/sounds/upload
```

```json
{
  "name": "doorbell.mp3",
  "data": "base64_encoded_file_content"
}
```

| Constraint | Value |
|------------|-------|
| Max size | 2 MB |
| Allowed formats | `.mp3`, `.ogg`, `.wav` |

### Delete Custom Sound

```
POST /api/sounds/delete
```

```json
{ "name": "doorbell.mp3" }
```

Only custom sounds can be deleted. Official sounds are read-only.

### Preview a Sound

```
GET /api/sounds/serve?name=01_notify.wav
```

Returns the audio file with the appropriate MIME type (`audio/mpeg`, `audio/ogg`, `audio/wav`).

### Default Sound Config

```
GET /api/config/sound
```

```json
{
  "enabled": true,
  "sound": "08-notify.mp3",
  "volume": 1.0
}
```

```
POST /api/config/sound
```

Same JSON body to update. This is the sound played when a notification doesn't specify one.

---

## 6. Text-to-Speech (TTS)

Make your TV talk. Because why not.

### Speak Text

```
POST /api/tts
```

```json
{
  "text": "Dinner is ready!",
  "lang": "en",
  "speed": 1.25,
  "pitch": 1.0,
  "volume": 1.0
}
```

| Parameter | Type | Default | Range | Description |
|-----------|------|---------|-------|-------------|
| `text` | string | *required* | — | Text to speak |
| `lang` | string | `"fr-FR"` | Any locale | Language code (`"en"`, `"fr"`, `"de"`, `"en-US"`, etc.) |
| `speed` | float | `1.0` | 0.5 — 2.0 | Speech rate. **1.25 recommended** (good balance) |
| `pitch` | float | `1.0` | 0.5 — 2.0 | Voice pitch |
| `volume` | float | `1.0` | 0.0 — 1.0 | Volume level |

### Stop TTS

```
POST /api/tts/stop
```

Immediately stops any ongoing speech.

### TTS in Notifications

You can combine TTS with a visual notification. The TV speaks while displaying:

```json
{
  "action": "DISPLAY",
  "duration": 10000,
  "tts": "You have a new message from Mom",
  "ttsLang": "en",
  "ttsSpeed": 1.25,
  "elements": [
    { "type": "text", "content": "mdi:message", "style": { "left": 45, "top": 40, "width": 10, "height": 10, "size": 60, "color": "#4CAF50" } }
  ]
}
```

---

## 7. Templates

Templates are saved notification layouts that can be reused and triggered by ID.

### Template Types

| Type | Location | Writable | Description |
|------|----------|----------|-------------|
| `official` | Built-in + overrides | Admin mode only | Shipped with the app, can be overridden |
| `draft` | User storage | Yes | Work in progress |
| `custom` | User storage | Yes | Finished user templates |

### List Templates

```
GET /api/templates/list
```

**Response:**
```json
{
  "official": [
    { "filename": "Welcome.json", "id": "a1b2c3d4-...", "params": ["title", "message"] },
    { "filename": "Alert.json", "id": "e5f6g7h8-..." }
  ],
  "draft": [
    { "filename": "WIP_Layout.json", "id": "..." }
  ],
  "custom": [
    { "filename": "My_Notif.json", "id": "...", "params": ["name"] }
  ]
}
```

The `params` array lists all parameter placeholders (`paramKey` / `actionParamKey`) found in the template's elements.

### Load a Template

```
GET /api/templates/load?type=custom&name=My_Notif.json
```

Returns the raw JSON content of the template.

### Save a Template

```
POST /api/templates/save
```

```json
{
  "type": "custom",
  "name": "My_Notif.json",
  "content": {
    "id": "unique-uuid",
    "name": "My Notification",
    "elements": [...]
  }
}
```

Saving to `official` type requires [Admin Mode](#admin-mode) to be enabled.

### Delete a Template

```
POST /api/templates/delete
```

```json
{ "type": "custom", "name": "My_Notif.json" }
```

### Rename a Template

```
POST /api/templates/rename
```

```json
{ "type": "custom", "oldName": "My_Notif.json", "newName": "Better_Name.json" }
```

### Move a Template

```
POST /api/templates/move
```

```json
{ "fromType": "draft", "toType": "custom", "name": "Finished_Layout.json" }
```

### Export All (ZIP)

```
GET /api/templates/export-zip
```

Returns a ZIP file containing all draft and custom templates.

### Import from ZIP

```
POST /api/templates/import-zip
```

Multipart form upload. Templates are **merged by ID** — if a template with the same UUID exists, it's overwritten.

---

## 8. SVG Management

Upload and serve custom SVG files for use in notifications.

**Upload:**
```
POST /api/svg/upload
```
```json
{ "name": "my_icon.svg", "content": "<svg xmlns='http://www.w3.org/2000/svg'>...</svg>" }
```

**List:**
```
GET /api/svg/list
```

**Serve:**
```
GET /api/svg/serve?name=my_icon.svg
```

**Use in a notification:**
```json
{ "type": "svg", "content": "/api/svg/serve?name=my_icon.svg", "style": { "left": 10, "top": 10, "width": 10, "height": 10 } }
```

---

## 9. Home Assistant Integration

peek-it has deep Home Assistant integration — real-time entity widgets, history charts, camera snapshots, and entity toggles in menus.

### HA Configuration

Before using HA features, configure the token:

```
POST /api/config/token
```
```json
{ "token": "eyJ0eXAiOiJKV1QiLCJhbGci..." }
```

The HA IP is auto-detected from the first notification payload's `ha_ip` field, or from the token configuration.

**Check current token:**
```
GET /api/config/token
```
```json
{ "token": "exists", "masked": "eyJ***..." }
```

### Entity Widget (Real-time)

Embed a live Home Assistant entity state as a widget inside a notification.

```
GET /api/ha/entity?entity_id=light.salon&mode=ws&map=on:#4CAF50,off:#F44336&display=both&icon=mdi:lightbulb&label=Living Room
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `entity_id` | *required* | HA entity ID |
| `mode` | `ws` | `ws` (WebSocket real-time) or `poll` (REST polling) |
| `map` | — | State-to-color mapping: `on:#4CAF50,off:#F44336,unavailable:#9E9E9E` |
| `display` | `both` | `both` (icon + text), `color` (icon only), `text` (text only) |
| `icon` | — | MDI icon name (e.g. `mdi:lightbulb`) |
| `label` | — | Custom label text |

**Poll mode** (for generic REST endpoints):

| Parameter | Default | Description |
|-----------|---------|-------------|
| `url` | *required* | Full URL to poll |
| `interval` | `10` | Polling interval in seconds |
| `path` | — | JSONPath to extract value from response |

Returns an HTML page — use it as a `webview` widget content URL.

### Entity Info

```
GET /api/ha/entity-info?entity_id=light.salon
```

```json
{
  "entity_id": "light.salon",
  "state": "on",
  "friendly_name": "Living Room Light",
  "icon": "mdi:lightbulb",
  "unit_of_measurement": null
}
```

### Entity History

```
GET /api/ha/history?entity_id=sensor.temperature&period=24h
```

| Period | Description |
|--------|-------------|
| `1h` | Last hour |
| `6h` | Last 6 hours |
| `24h` | Last 24 hours |
| `3d` | Last 3 days |
| `7d` | Last 7 days |
| `30d` | Last 30 days |

Returns HA history format (JSON array). Results are **cached** for performance.

### Chart Widget

Generate a beautiful pure CSS/SVG chart of entity history. No JavaScript library needed — it's all server-rendered.

```
GET /api/ha/chart?entity_id=sensor.temperature&period=24h&chart_type=area&color=%232196F3&title=Temperature&unit=%C2%B0C
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `entity_id` | *required* | HA entity ID |
| `period` | `24h` | History period (see above) |
| `chart_type` | `area` | `area`, `line`, or `bar` |
| `color` | `#2196F3` | Chart color |
| `title` | — | Chart title |
| `unit` | — | Unit label (e.g. `°C`) |
| `stroke` | `3` | Line width (px) |
| `smooth` | `true` | Smooth curves |
| `fill` | `true` | Fill area under line |
| `grid` | `true` | Show grid lines |
| `refresh` | `0` | Auto-refresh interval (seconds, 0 = off) |
| `min` / `max` | — | Y-axis bounds |
| `bg` | `#1a1a2e` | Background color |
| `bg_opacity` | `100` | Background opacity (0-100%) |

Returns an HTML page — use it as a `webview` widget content URL in your notification.

### Camera Snapshot

```
GET /api/camera/snapshot?entity_id=camera.front_door
```

Returns a **JPEG image** of the camera's latest snapshot. Cached for performance.

Use it as the `content` of an `image` widget:

```json
{
  "type": "image",
  "content": "http://TV_IP:8081/api/camera/snapshot?entity_id=camera.front_door",
  "style": { "left": 0, "top": 0, "width": 50, "height": 50 }
}
```

---

## 10. Menu Widget (TV Overlay)

The `menu` widget type creates a full overlay menu navigable with a TV remote (D-pad). Think Android TV settings menu, but for your automations.

### Menu Structure

The `content` field of a `menu` element contains a JSON string (yes, JSON inside JSON — we heard you like JSON):

```json
{
  "type": "menu",
  "content": "{\"title\":\"My Menu\", \"items\":[...]}",
  "style": { "left": 0, "top": 0, "width": 100, "height": 100 }
}
```

### Menu Config

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `title` | string | *null* | Menu title (displayed at top) |
| `titleIcon` | string | *null* | MDI icon next to title |
| `bgColor` | string | `"#1E1E1E"` | Menu background color |
| `textColor` | string | `"#FFFFFF"` | Item text color |
| `accentColor` | string | `"#00E676"` | Title and focus accent color |
| `fontSize` | integer | `16` | Font size (sp) |
| `cornerRadius` | integer | `12` | Corner radius (dp) |
| `showBorder` | boolean | `false` | Show border |
| `borderColor` | string | `"#333333"` | Border color |
| `borderWidth` | integer | `2` | Border width (dp) |
| `showShadow` | boolean | `false` | Show drop shadow |
| `shadowElevation` | integer | `16` | Shadow elevation (dp) |
| `items` | array | `[]` | Array of menu items |

### Menu Item Types

| Type | Focusable | Description |
|------|-----------|-------------|
| `action` | Yes | Triggers a callback action on press |
| `submenu` | Yes | Opens a nested menu (children array) |
| `toggle` | Yes | HA entity toggle with live ON/OFF state |
| `text` | No | Informational text (not interactive) |
| `close` | Yes | Closes the menu or returns to parent |

**Item structure:**

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | One of the types above |
| `label` | string | Display text |
| `icon` | string | MDI icon (e.g. `mdi:lightbulb`) |
| `action` | string | Callback action string (for `action` and `toggle`) |
| `entity_id` | string | HA entity ID (for `toggle` only) |
| `children` | array | Nested menu items (for `submenu` only) |

**Full example:**

```json
{
  "title": "Home Control",
  "titleIcon": "mdi:home",
  "bgColor": "#1E1E1E",
  "textColor": "#FFFFFF",
  "accentColor": "#00E676",
  "items": [
    { "type": "action", "label": "Play Music", "icon": "mdi:play", "action": "play_music" },
    { "type": "submenu", "label": "Lights", "icon": "mdi:lightbulb-group", "children": [
      { "type": "toggle", "label": "Living Room", "icon": "mdi:lightbulb", "action": "toggle_living", "entity_id": "light.living_room" },
      { "type": "toggle", "label": "Bedroom", "icon": "mdi:lightbulb", "action": "toggle_bedroom", "entity_id": "light.bedroom" },
      { "type": "close", "label": "Back", "icon": "mdi:arrow-left" }
    ]},
    { "type": "text", "label": "v2.0" },
    { "type": "close", "label": "Close", "icon": "mdi:close" }
  ]
}
```

### D-pad Navigation

| Key | Action |
|-----|--------|
| **Up** | Previous item |
| **Down** | Next item |
| **Right** / **Enter** | Open submenu or trigger action |
| **Left** / **Back** | Return to parent menu |
| **Back** (root level) | Close menu |

### Menu Sizing

- **Width:** Auto-calculated from the longest item text. Capped at **50% of screen width**
- **Height:** Auto-calculated from number of items (48dp each + 50dp header). Capped at **95% of screen height**
- Scrolls if content exceeds height cap

### Toggle Polling

Toggle items poll the Home Assistant REST API every **5 seconds** to update their ON/OFF state display. The endpoint used is `GET /api/states/{entity_id}` on the HA server, authenticated with the Bearer token.

---

## 11. Configuration

### Clock Overlay

A persistent clock overlay displayed on top of all content.

**Get:**
```
GET /api/config/clock
```

**Set:**
```
POST /api/config/clock
```

```json
{
  "enabled": true,
  "format": "HH:mm",
  "left": 85,
  "top": 3,
  "color": "#FFFFFF",
  "size": 28,
  "opacity": 0.8,
  "bgColor": "#000000",
  "bgOpacity": 0.0
}
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `enabled` | boolean | `false` | Show/hide clock |
| `format` | string | `"HH:mm"` | Java SimpleDateFormat pattern |
| `left` | float | `85` | Horizontal position (%) |
| `top` | float | `3` | Vertical position (%) |
| `color` | string | `"#FFFFFF"` | Text color |
| `size` | integer | `28` | Font size (sp) |
| `opacity` | float | `0.8` | Text opacity |
| `bgColor` | string | `"#000000"` | Background color |
| `bgOpacity` | float | `0.0` | Background opacity |

**Format examples:** `HH:mm` (24h), `HH:mm:ss` (24h + seconds), `hh:mm a` (12h AM/PM)

Changes are applied **immediately** — no restart needed.

### Screen Dimming

Adds a semi-transparent overlay on top of the entire screen. Notifications display above it.

**Get / Set:**
```
GET /api/config/dimming
POST /api/config/dimming
```

```json
{
  "enabled": true,
  "color": "#000000",
  "opacity": 0.5
}
```

Applied immediately.

### Admin Mode

Unlocks the ability to create, modify, rename, and delete **official** templates.

**Get / Set:**
```
GET /api/config/admin
POST /api/config/admin
```

```json
{ "enabled": true }
```

### Start Menu Override

Configure which template is displayed when the user long-presses the Back button on the TV remote.

**Get / Set:**
```
GET /api/config/startmenu
POST /api/config/startmenu
```

```json
{
  "template_id": "uuid-of-menu-template",
  "template_name": "My Start Menu"
}
```

Set `template_id` to `""` to restore the default built-in start menu.

### Language

**Get:**
```
GET /api/config/language
```

```json
{ "language": "en" }
```

**Set:**
```
POST /api/config/language
```

```json
{ "language": "fr" }
```

Supported: `en`, `fr`, `de`, `es`, `nl`. Both endpoints are **public** (no auth required).

### Home Assistant Token

**Get (masked):**
```
GET /api/config/token
```

```json
{ "token": "exists", "masked": "eyJ***7890" }
```

**Set:**
```
POST /api/config/token
```

```json
{ "token": "eyJ0eXAiOiJKV1QiLCJhbGci..." }
```

### API Key Management

**Get:**
```
GET /api/config/apikey
```

```json
{ "key": "exists", "masked": "abc***xyz" }
```

**Reveal full key:**
```
GET /api/config/apikey?reveal=true
```

```json
{ "key": "exists", "masked": "abc***xyz", "full": "abc123def456xyz" }
```

Requires authentication.

**Set new key:**
```
POST /api/config/apikey
```

```json
{ "key": "my_new_api_key" }
```

---

## 12. Service Status

```
GET /api/status
```

**Response:**

```json
{
  "status": "online",
  "version": "v10.9",
  "device_name": "SHIELD Android TV",
  "api_key_required": true,
  "api_key_valid": true,
  "screen": {
    "width": 1920,
    "height": 1080,
    "density": 2.0
  }
}
```

This endpoint is **public** — no auth required. This is intentional: it allows companion apps (like peek-it Send) to check connectivity and validate API keys.

---

## 13. Logs

```
GET /api/logs
```

Returns the service debug log file content. Useful for troubleshooting notification delivery, HA connections, and sound playback issues.

---

## 14. Internationalization

The Designer web UI supports multiple languages. Locale files are served publicly:

```
GET /locales/en.json
GET /locales/fr.json
GET /locales/de.json
GET /locales/es.json
GET /locales/nl.json
```

Each file contains ~373 translation keys used by the Designer interface.

The language preference is stored server-side and auto-detected in this order:
1. `localStorage` (browser)
2. `/api/config/language` (server setting)
3. `navigator.language` (browser locale)
4. Fallback: `en`

---

## 15. Overlay Behavior

### Notification Stack

peek-it supports up to **5 stacked notifications**. When a new notification arrives while one is already displayed:

- The current notification is **paused** (timer stops)
- The new one displays on top
- When the top one closes, the one underneath **resumes**

Sending `{"action": "CLOSE"}` always closes the **topmost** notification.

### Anti Burn-in

For persistent notifications (`duration: 0`), peek-it applies a subtle **pixel shift** to prevent OLED burn-in:

- Shifts by 2 pixels in a 4-step cycle
- Pattern: (0,0) → (2,0) → (2,2) → (0,2)
- Interval: every 2 minutes

The same protection applies to the clock overlay.

### WebView Synchronization

When a notification contains multiple WebView widgets:

1. All WebViews load in parallel
2. Each can signal readiness via `PeekBridge.notifyReady()`
3. The entry animation waits until **all** WebViews are ready (or 2s timeout)
4. This ensures everything appears at once, no janky half-loaded states

---

## 16. Limits & Constraints

| Resource | Limit |
|----------|-------|
| Notification stack depth | 5 |
| Custom sound file size | 2 MB |
| Sound formats | `.mp3`, `.ogg`, `.wav` |
| Screen position values | 0-100 (%) |
| TTS speed range | 0.5 - 2.0 |
| TTS pitch range | 0.5 - 2.0 |
| Menu width | Max 50% of screen |
| Menu height | Max 95% of screen |
| Toggle polling interval | 5 seconds |
| Entry animation duration | 400ms |
| Exit animation duration | 300ms |
| WebView ready timeout | 2 seconds (5s max) |
| Anti burn-in cycle | 2 minutes |
| Default port | 8081 (configurable) |

### Security

- **Path traversal protection** on all file operations (templates, sounds, SVGs)
- **WebView permissions denied** (camera, mic, location)
- **CORS:** Reflected origin (not wildcard)
- **API key** required for all write operations

---

## 17. Error Responses

| HTTP Code | Meaning |
|-----------|---------|
| `200` | Success |
| `400` | Bad request (missing fields, invalid JSON) |
| `401` | Unauthorized (missing or invalid API key) |
| `403` | Forbidden (path traversal attempt, admin mode required) |
| `404` | Not found (template, sound, or SVG doesn't exist) |
| `500` | Internal server error |

Error response body:

```json
{ "error": "Description of what went wrong" }
```

---

## License

peek-it is proprietary software by [jolabs40](https://github.com/jolabs40).

This documentation describes the public HTTP API exposed by the peek-it TV Android application.

---

*Built with obsessive attention to JSON and an unreasonable love for TV overlays.*
